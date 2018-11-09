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


  在将`grpc`用在实际项目过程当中，碰到一系列问题，这些问题总结下来基本都与服务发现与负载均衡这个话题有关。

  简化的服务架构如下:

  ![base_arch](/images/2018/0916/base_arch.png)

  上图中，gateway是一组go服务，它除用来做接入控制之外，其中还有项重要工作就是协议转换：对外提供`HTTP`请求，并将其转换为RPC调用请求具体
  上游微服务。

  具体到实际项目中，上游微服务作为gRPC的server端，它负责具体业务逻辑；gateway作为gRPC的client端，将http请求组装成gRPC请求发起RPC调用。

  最后一公里，如何铺设gRPC?

  主要有两种方案: 
  
  - 服务发现与负载均衡都由client来实现: micro-service启动时向gateway注册，由gateway根据注册情况向micro-service发起调用。
  - 通过代理隔离两组服务，服务发现与负载均衡均交由代理来解决。
  - 服务发现借助其他服务，比如DNS，etcd等组件，client侧自己决定如何调度请求。

  先说第一种情况,很明显两组服务之间存在严重耦合，下下策。

  第二种情况看起来最为省心: 两者隔离，烦心事全交给代理来操心。

  第三种方案看起来稍稍比第一种方案好一些，但仍然麻烦，不过好在gRPC内部已经实现了一个基本的`round-robin`负载均衡实现。也实现了使用
  DNS作为负载均衡的服务发现插件。

  我们首先尝试了第二种方案。万金油nginx从1.13版本之后宣布支持`gRPC`负载均衡。

  [Introducing gRPC Support with NGINX 1.13.10](https://www.nginx.com/blog/nginx-1-13-10-grpc/)

  nginx配置中增加`grpc_pass`,使用方式与`proxy_pass`一致:

    worker_processes  1;
    events {
        worker_connections  1024;
    }

    http {
        include       mime.types;
        default_type  application/octet-stream;
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        error_log   logs/error.log debug;
        access_log  logs/access.log  main;

        sendfile        on;
        keepalive_timeout  65;

        upstream grpc_servers {
            #least_conn;
            #keepalive 1000;
            server 127.0.0.1:10006;
        }
        server {
            listen       8080 http2;
            server_name  localhost;
            #access_log  logs/host.access.log  main;
            location / {
                grpc_pass grpc://grpc_servers;
                error_page 502 = /error502grpc;
            }

            location = /error502grpc {
                internal;
                default_type application/grpc;
                add_header grpc-status 14;
                add_header grpc-message "unavailable";
                return 204;
            }
        }
    }

  现在的架构变成这样:

  ![nginx grpc_pass proxy](/images/2018/0916/nginx_grpc_pass.png)

  发起请求，从`gateway` <---> `nginx` <---> `A serivce-providor`, 表面看起来一切正常，发起请求，也成功得到返回，但存在问题：

  **只有1处是长连接，2处的连接是短连接！**

  因为1处是长连接，请求源源不断到达。当后端的微服务部署上线时（微服务ip跑在k8s中，IP经常变动），nginx reload迟迟无法正常结束。
  每次请求结束，2处nginx立马断开连接，造成nginx堆积大量`TIME_WAIT`。一个简单的配置接口，nginx侧有2万多的TIME_WAIT连接状态。

  nginx reload这个问题可以设置`worker_shutdown_timeout`参数，控制nginx优雅关闭时间，避免长时间一直处于shutdown状态,勉强算是可用。

  但对于grpc短连接这个问题，目前暂时无法解决。

  nginx宣称的支持gRPC成了鸡肋，不过，好在nginx还提供了L4层代理。可使用`stream`块让nginx不对应用层做解析

    worker_processes  1;
    error_log logs/debug.log debug;
    events {
        worker_connections  1024;
    }

    stream {
        access_log  logs/access2.log  main;

        upstream grpc_backends {
            server localhost:10006;
            #server localhost:10008;
        }
        server {
            listen 8080;
            proxy_pass grpc_backends;
        }
    }

    http {
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  logs/access.log  main;

        sendfile        on;
        keepalive_timeout  65;

        upstream http_backends {
            server localhost:10005;
        }

        server {
            listen       8000;
            server_name  localhost;

            location / {
                proxy_pass http://http_backends;
            }
        }
    }

  不过这种方式没法使用nginx来做成统一的监控。运维对nginx L4层代理还未做统一规划。

  用nginx做代理这个方案暂时止步到这。接下来，说下第二种方案的具体实施过程，这也是目前我们采取的方式。


  目前，我们采用**etcd做服务发现，client侧做负载均衡**。

  架构图变成这样:

  ![etcd discory](/images/2018/0916/etcd_grpc.png)

  流程大致是这样: 
  1. 微服务启动时，自动向etcd注册，申请租约并定时维持。
  2. gateway订阅某个服务，定时获取某个服务名称下的所有机器，与之建立连接，定时更新。
  3. 当请求到来时，client在活跃机器中做轮询调用。

  具体实现：

#### 服务注册代码

{% highlight go %}

package rpc

import (
	"context"
	"net"
	"os"
	"time"
	"strings"
	"fmt"
	"errors"

	"git.sogou-inc.com/iweb/go-util"
	"git.sogou-inc.com/iweb/ime-ucenter/rpc/proto"
	"github.com/coreos/etcd/clientv3"
	"google.golang.org/grpc"
)

func init() {
	go RPCServeForever()
}

func RPCServeForever() error {
	var (
		err      error
		listener net.Listener
	)

	srv := grpc.NewServer()

	proto.RegisterQcloudServiceServer(srv, new(QCloudRpcServer))

	if listener, err = net.Listen("tcp4", util.AppConfig.RpcListen); err != nil {
		return err
	}

	fmt.Println("[D] rpc will serve on %s", util.AppConfig.RpcListen)

	/* stupid but useful */
	go registerMyself(5)

	return srv.Serve(listener)
}

func registerMyself(ttl int64) {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   util.AppConfig.ETCDCluster,
		DialTimeout: 3 * time.Second,
	})

	if err != nil {
		util.BhAlarm(util.BH_LOG_SYSTEM, err, "New etcd client error")
		panic(err)
	}

	_myself := myself()
	resp := NewLeaseGrant(cli, _myself, ttl*2)

	stop := make(chan struct{})
	util.RegistExitHook(func() error {
		stop <- struct{}{}
		if cli == nil {
			return nil
		}

		defer cli.Close()

		if delResp, err := cli.Delete(context.TODO(), _myself); err == nil {
			fmt.Println("sai yo na ra:", delResp)
		}

		return err
	})

	t := time.NewTicker(time.Duration(ttl) * time.Second)

	for {
		select {
		case <-t.C:
			kres, e := cli.KeepAliveOnce(context.TODO(), resp.ID)
			if e != nil {
                fmt.Fprintf(os.stderr, "[E] keepalive_once error: %s", e.Error())
				// bugfix: 重新注册
				// 某次观察到etcd集群工作正常，但keepAliveOnce一直维持不住导致服务不可用
				resp = NewLeaseGrant(cli, _myself, 2*ttl)
			} else {
				fmt.Println("[D] keepalive response, version:", kres.Revision, "raft_term:", kres.RaftTerm)
			}
		case <-stop:
			t.Stop()
			fmt.Println("Oops: goodbye")
			return
		}
	}

	return
}

func myself() string {
	hostName, _ := os.Hostname()
	return `/xxx_service/` + hostName
}

// FIXME: the first one ?
func getLocalIPV4Addr(port string) string {
	addrs, err := net.InterfaceAddrs()
	if err != nil {
		panic(err)
	}

	for _, addr := range addrs {
		if ip, ok := addr.(*net.IPNet); ok && !ip.IP.IsLoopback() {
			if ip.IP.To4() == nil {
				continue
			}

			if strings.HasPrefix(port, ":") {
				return ip.IP.String() + port
			}

			return net.JoinHostPort(ip.IP.String(), port)
		}
	}

	return ""
}

func NewLeaseGrant(client *clientv3.Client, value string, ttl int64) *clientv3.LeaseGrantResponse {
	if client == nil {
		panic(errors.New("Invalid etcd client"))
	}

	resp, err := client.Grant(context.TODO(), ttl*2) // must longer
	if err != nil {
		util.BhAlarm(util.BH_LOG_SYSTEM, err, "Grant ucenter error")
		panic(err)
	}

	_, err = client.Put(context.TODO(), value, getLocalIPV4Addr(util.AppConfig.RpcListen), clientv3.WithLease(resp.ID))
	if err != nil {
		util.BhAlarm(util.BH_LOG_SYSTEM, err, "Put myself into etcd cluster error")
		panic(err)
	}

	return resp
}

{% endhighlight %}


#### etcd resolver

  截止目前，官方暂时还未实现etcd的名称解析。要想使用etcd，目前需要自己实现。

  {% highlight go %}

package resolver

import (
	"context"
	"errors"
	"net"
	"sync"
	"time"

	"strings"

	"github.com/coreos/etcd/clientv3"
	"google.golang.org/grpc/grpclog"
	"google.golang.org/grpc/resolver"
)

const (
	defaultPort = "2379"
)

var (
	defaultMinFrequency = 120 * time.Second
)

func init() {
}

type etcdBuilder struct {
	watchKeyPrefix string
}

func NewETCDBuilder() resolver.Builder {
	return &etcdBuilder{}
}

func RegisterResolver(keyPrefix string) {
	builder := &etcdBuilder{watchKeyPrefix: keyPrefix}

	resolver.Register(builder)
}

func (b *etcdBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
	etcdProxys, err := parseTarget(target.Endpoint)
	if err != nil {
		return nil, err
	}

	grpclog.Infoln("etcd resolver, endpoints:", etcdProxys)

	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   etcdProxys,
		DialTimeout: 3 * time.Second,
	})
	if err != nil {
		return nil, errors.New("connect to etcd proxy error")
	}

	ctx, cancel := context.WithCancel(context.Background())
	rlv := &etcdResolver{
		cc:             cc,
		cli:            cli,
		ctx:            ctx,
		cancel:         cancel,
		watchKeyPrefix: b.watchKeyPrefix,
		freq:           5 * time.Second,
		t:              time.NewTimer(0),
		rn:             make(chan struct{}, 1),
		im:             make(chan []resolver.Address),
		wg:             sync.WaitGroup{},
	}

	rlv.wg.Add(2)
	go rlv.watcher()
	go rlv.FetchBackendsWithWatch()

	return rlv, nil
}

func (b *etcdBuilder) Scheme() string {
	return "etcd"
}

type etcdResolver struct {
	retry  int
	freq   time.Duration
	ctx    context.Context
	cancel context.CancelFunc
	cc     resolver.ClientConn
	cli    *clientv3.Client
	t      *time.Timer

	watchKeyPrefix string

	rn chan struct{}
	im chan []resolver.Address

	wg sync.WaitGroup
}

func (r *etcdResolver) ResolveNow(opt resolver.ResolveNowOption) {
	select {
	case r.rn <- struct{}{}:
	default:
	}
}

func (r *etcdResolver) Close() {
	r.cancel()
	r.wg.Wait()
	r.t.Stop()
}

func (r *etcdResolver) watcher() {
	defer r.wg.Done()

	for {
		select {
		case <-r.ctx.Done():
			return
		case addrs := <-r.im:
			if len(addrs) > 0 {
				r.retry = 0
				r.t.Reset(r.freq)
				r.cc.NewAddress(addrs)
				continue
			}
		case <-r.t.C:
		case <-r.rn:
		}

		result := r.FetchBackends()

		if len(result) == 0 {
			r.retry++
			r.t.Reset(r.freq)
		} else {
			r.retry = 0
			r.t.Reset(r.freq)
		}

		r.cc.NewAddress(result)
	}
}

func (r *etcdResolver) FetchBackendsWithWatch() {
	defer r.wg.Done()

	for {
		select {
		case <-r.ctx.Done():
			return
		case _ = <-r.cli.Watch(r.ctx, r.watchKeyPrefix, clientv3.WithPrefix()):
			result := r.FetchBackends()
			r.im <- result
		}
	}
}

func (r *etcdResolver) FetchBackends() []resolver.Address {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	result := make([]resolver.Address, 0)

	resp, err := r.cli.Get(ctx, r.watchKeyPrefix, clientv3.WithPrefix())
	if err != nil {
		grpclog.Errorln("Fetch etcd proxy error:", err)
		return result
	}

	for _, kv := range resp.Kvs {
		if strings.TrimSpace(string(kv.Value)) == "" {
			continue
		}
		result = append(result, resolver.Address{Addr: string(kv.Value)})
	}

	grpclog.Infoln(">>>>> endpoints fetch: ", result)

	return result
}

func parseTarget(target string) ([]string, error) {
	var (
		endpoints = make([]string, 0)
	)

	if target == "" {
		return nil, errors.New("invalid target")
	}

	for _, endpoint := range strings.Split(target, ",") {
		if ip := net.ParseIP(endpoint); ip != nil {
			endpoints = append(endpoints, net.JoinHostPort(endpoint, defaultPort))
			continue
		}

		if _, port, err := net.SplitHostPort(endpoint); err == nil {
			if port == "" {
				return endpoints, errors.New("Invalid address format")
			}
			endpoints = append(endpoints, endpoint)
		}
	}

	return endpoints, nil
}

  {% endhighlight %}

#### gRPC client
  
  {% highlight go %}

package rpc

import (
	"context"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/keepalive"
	"google.golang.org/grpc/grpclog"
	"os"
)

const (
	const_grpc_lbname = `round_robin`
)

var (
	qcloudRpcClient proto.QcloudServiceClient
)

func init() {

     //将插件注册进gRPC
	resolver.RegisterResolver("/xxx_server/")

	keepAlive := keepalive.ClientParameters{
		10 * time.Second,
		20 * time.Second,
		true,
	}

	etcdCluster := "etcd:///" + util.AppConfig.ETCDCluster // 指定使用etcd来做名称解析
	ctx, _ := context.WithTimeout(context.Background(), 5*time.Second)

     // grpc.WithInsecure: 不使用安全连接
     // grpc.WithBalancerName("round_robin"), 轮询机制做负载均衡
     // grpc.WithBlock: 握手成功才返回
     // grpc.WithKeepaliveParams: 连接保活，防止因为长时间闲置导致连接不可用
	conn, err := grpc.DialContext(ctx, etcdCluster, grpc.WithInsecure(), grpc.WithBalancerName(const_grpc_lbname),
		grpc.WithBlock(), grpc.WithKeepaliveParams(keepAlive))
	if err != nil {
		panic(err)
	}

	grpclog.SetLoggerV2(grpclog.NewLoggerV2WithVerbosity(os.Stdout, os.Stderr, os.Stderr, 9))
	qcloudRpcClient = proto.NewQcloudServiceClient(conn)
}

func GetQCloudClient() proto.QcloudServiceClient {
	return qcloudRpcClient
}

  {% endhighlight %}

  这里，etcd resolver插件的正确放置目录应该在`grpc/resolver`目录下。这里并未放到该目录下的原因是因为我们写的部分参数
  暂时无法像grpc.DialContext(...)这种灵活地在上层传入。另外，版本管理造成麻烦。

#### 几点注意

  1. 从实现可以看出，只有定时地读写etcd，etcd不存在压力。
  2. 如果后端服务一直正常运行(包括keepalive也正常)，则每次从etcd读取的配置有序。
  3. client每次获取服务ip并非都要重建连接: gRPC内部会做比较，只有前后不一致时才会重建连接。
  4. 当搭建的是etcd集群，etcd同一时间只能跟其中一个节点建立连接，当这个节点失败时，etcd client会跟集群的其他活跃节点建立连接。
