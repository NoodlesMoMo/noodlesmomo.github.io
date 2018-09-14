---
title: DNS基础知识及Bind DNS 安装
author: Noodles
layout: post
comments: true
permalink: /2018/09/dns-bind-dns-install
categories:
  - dns
tags:
  - dns
---

<!--more-->

 ---------------------------------------------------

### DNS基础知识

  - 以点`.`结尾的域名称为绝对域名或完全合格的域名`FQDN`（Full Qualified Domain Name）.

  - `arpa`顶级域是一个用作地址到名字转换的特殊域。

  - 一个独立管理的DNS子树称为一个区域(zone)。一个常见的区域是一个二级域。例如`noao.edu`。
  许多二级域将它们的区域划分成为更小的区域。例如,大学可能根据不同的系来划分区域，公司可能
  根据不同的部门来划分区域。


 DNS服务器中有两个比较重要的记录。一个是SOA,另一个是NS.

    SOA: 起始授权机构
    NS: Name server

    SOA记录表明了DNS服务器之间的关系。SOA记录表明了谁是这个区域的所有者。

 `BIND`者, `Berkeley Internet Name Domain`也。

### ClientConn

{% highlight go %}

type ClientConn struct {
	ctx    context.Context      // 标准库的context.Context
	cancel context.CancelFunc   // 标准库的context.CancelFunc 

	target       string             // 连接目的地址
	parsedTarget resolver.Target    // 解析后的目的地址
	authority    string
	dopts        dialOptions
	csMgr        *connectivityStateManager

	balancerBuildOpts balancer.BuildOptions // 负载均衡选项
	resolverWrapper   *ccResolverWrapper
	blockingpicker    *pickerWrapper

	mu    sync.RWMutex
	sc    ServiceConfig
	scRaw string
	conns map[*addrConn]struct{}
	// Keepalive parameter can be updated if a GoAway is received.
	mkp             keepalive.ClientParameters
	curBalancerName string
	preBalancerName string // previous balancer name.
	curAddresses    []resolver.Address
	balancerWrapper *ccBalancerWrapper
	retryThrottler  atomic.Value

	channelzID int64 // channelz unique identification number
	czData     *channelzData
}

{% endhighlight %}

