---
title: grpc 名称发现与负载均衡
author: Noodles
layout: post
comments: true
permalink: /2018/09/grpc-name-resolve-loadbalance
categories:
  - golang
tags:
  - golang
---

<!--more-->

 ---------------------------------------------------

 *grpc目前代码几乎每天都在更新。这里是截止到2018-9-6号的最新代码, 版本是1.15-dev*



### ClientConn

{% highlight go %}

type ClientConn struct {

    // ctx, cancel引用标准库context中的结构，用来管理超时
	ctx    context.Context
	cancel context.CancelFunc

	target       string             // 连接目的地址
	parsedTarget resolver.Target    // 解析后的目的地址
	authority    string             // 认证相关
	dopts        dialOptions        // 构建连接选项

    // 连接状态管理
	csMgr        *connectivityStateManager

	balancerBuildOpts balancer.BuildOptions // 负载均衡选项
	resolverWrapper   *ccResolverWrappe     // 地址解析
	blockingpicker    *pickerWrapper

    // 服务侧配置
	mu    sync.RWMutex
	sc    ServiceConfig // 已废弃
	scRaw string

	conns map[*addrConn]struct{}
	
    // Keepalive parameter can be updated if a GoAway is received.
	mkp             keepalive.ClientParameters

    // LB相关
    curBalancerName string
    preBalancerName string // previous balancer name.
	curAddresses    []resolver.Address  // 存储当前地址。当地址有更新时，用来做比较，如果相同则不做处理。
	balancerWrapper *ccBalancerWrapper
	retryThrottler  atomic.Value

	channelzID int64 // channelz unique identification number
	czData     *channelzData
}

{% endhighlight %}

看这个连接的创建:

{% highlight go %}
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		target:         target,
		csMgr:          &connectivityStateManager{},
		conns:          make(map[*addrConn]struct{}),
		dopts:          defaultDialOptions(),
		blockingpicker: newPickerWrapper(),
		czData:         new(channelzData),
	}
	cc.retryThrottler.Store((*retryThrottler)(nil))
	cc.ctx, cc.cancel = context.WithCancel(context.Background())

	for _, opt := range opts {
		opt.apply(&cc.dopts)
	}

	cc.mkp = cc.dopts.copts.KeepaliveParams

	if cc.dopts.copts.Dialer == nil {
		cc.dopts.copts.Dialer = newProxyDialer(
			func(ctx context.Context, addr string) (net.Conn, error) {
				network, addr := parseDialTarget(addr)
				return dialContext(ctx, network, addr)
			},
		)
	}

	if cc.dopts.copts.UserAgent != "" {
		cc.dopts.copts.UserAgent += " " + grpcUA
	} else {
		// 设置UA
		cc.dopts.copts.UserAgent = grpcUA
	}

	defer func() {
		select {
		case <-ctx.Done():
			conn, err = nil, ctx.Err()
		default:
		}

		if err != nil {
			cc.Close()
		}
	}()

	// 设置重新建立连接的最大时间间隔，默认为120秒
	if cc.dopts.bs == nil {
		cc.dopts.bs = backoff.Exponential{
			MaxDelay: DefaultBackoffConfig.MaxDelay,
		}
	}

	if cc.dopts.resolverBuilder == nil {
		// Only try to parse target when resolver builder is not already set.
		cc.parsedTarget = parseTarget(cc.target)
		grpclog.Infof("parsed scheme: %q", cc.parsedTarget.Scheme)
		cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		if cc.dopts.resolverBuilder == nil {
			// If resolver builder is still nil, the parse target's scheme is
			// not registered. Fallback to default resolver and set Endpoint to
			// the original unparsed target.
			grpclog.Infof("scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)

			// 如果未设置`schema`,则使用默认的`passthrough`
			cc.parsedTarget = resolver.Target{
				Scheme:   resolver.GetDefaultScheme(),
				Endpoint: target,
			}
			cc.dopts.resolverBuilder = resolver.Get(cc.parsedTarget.Scheme)
		}
	} else {
		cc.parsedTarget = resolver.Target{Endpoint: target}
	}
	creds := cc.dopts.copts.TransportCredentials
	if creds != nil && creds.Info().ServerName != "" {
		cc.authority = creds.Info().ServerName
	} else if cc.dopts.insecure && cc.dopts.authority != "" {
		cc.authority = cc.dopts.authority
	} else {
		// Use endpoint from "scheme://authority/endpoint" as the default
		// authority for ClientConn.
		cc.authority = cc.parsedTarget.Endpoint
	}

	cc.balancerBuildOpts = balancer.BuildOptions{
		DialCreds:        credsClone,
		Dialer:           cc.dopts.copts.Dialer,
		ChannelzParentID: cc.channelzID,
	}

	// Build the resolver.
	cc.resolverWrapper, err = newCCResolverWrapper(cc)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}
	// Start the resolver wrapper goroutine after resolverWrapper is created.
	//
	// If the goroutine is started before resolverWrapper is ready, the
	// following may happen: The goroutine sends updates to cc. cc forwards
	// those to balancer. Balancer creates new addrConn. addrConn fails to
	// connect, and calls resolveNow(). resolveNow() tries to use the non-ready
	// resolverWrapper.
	cc.resolverWrapper.start()

	// A blocking dial blocks until the clientConn is ready.
	if cc.dopts.block {
		for {
			s := cc.GetState()
			if s == connectivity.Ready {
				break
			}
			if !cc.WaitForStateChange(ctx, s) {
				// ctx got timeout or canceled.
				return nil, ctx.Err()
			}
		}
	}

	return cc, nil
}

{% endhighlight %}

### GRPC负载均衡

  **GRPC的负载均衡针对的是每次的请求调用，而不是每个连接: 即使所有的请求都是在一条连接，GRPC负载均衡的目标也是将这条连接的所有请求均衡到后端服务中。**


#### 三种典型的负载均衡模型

- 代理模型(Proxy Model)

  clients ---> proxy ---> backends

  由Proxy充当调度，这类模型对clients和backends来说最简单，clients侧不需要做额外的工作。比如, Nginx从1.13开始已经支持grpc反向代理。很明显，这类模型需要临时缓存请求和响应，需要通常会占用更多的资源，显著增加GRPC调用延迟。

- Balanceing-aware Client

  这种能够感知平衡调度的客户端模型
