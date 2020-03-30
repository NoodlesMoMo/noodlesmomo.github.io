---
title: go-control-plane source reading
author: Noodles
layout: post
comments: true
permalink: /2020/02/24/go-control-plane-src
photos:
- http://q61qnv6m4.bkt.clouddn.com/2020%2F02%2F24%2Fenvoy.png
---

本文对envoy go-control-plane源码进行梳理

<!--more-->

 ---------------------------------------------------

 `go-control-plane`是envoy提供的一个控制面的go实现。这里分析的是v0.9.1版本，commit号为: 34c8be46e7fdd171a21e25203bc29e9e9ee56886

### AggregatedDiscoveryServiceServer 

  这个包实现了`AggregatedDiscoveryServiceServer`，它可以提供`ads`模式的xDS协议服务。

``` go

func (s *server) StreamAggregatedResources(stream discoverygrpc.AggregatedDiscoveryService_StreamAggregatedResourcesServer) error {
	return s.handler(stream, cache.AnyType) // 调用handler处理函数
}


func (s *server) handler(stream stream, typeURL string) error {
	// a channel for receiving incoming requests
	reqCh := make(chan *v2.DiscoveryRequest)
	reqStop := int32(0) // 控制下面的读取go routine退出。
	go func() {
        // 创建一个go routine，不停的从stream读取数据。
		for {
			req, err := stream.Recv()
			if atomic.LoadInt32(&reqStop) != 0 {
				return
			}
			if err != nil {
				close(reqCh)
				return
			}
			reqCh <- req
		}
	}()

	err := s.process(stream, reqCh, typeURL)

	// prevents writing to a closed channel if send failed on blocked recv
	// TODO(kuat) figure out how to unblock recv through gRPC API
	atomic.StoreInt32(&reqStop, 1)

	return err
}

```

  当双向流中有请求进来时，handler就会创建一个读取流的go routine。这个读取go routine和它的父go程之间通过一个堵塞的
  `v2.DiscoveryRequest` channel进行交换数据。

