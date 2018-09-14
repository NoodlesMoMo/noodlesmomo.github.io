---
title: go build条件编译
author: Noodles
layout: post
comments: true
permalink: /2018/08/go-condition-build
categories:
  - golang
tags:
  - golang
---

<!--more-->

 ---------------------------------------------------
  
  在某个项目需要支持多平台时，某个功能可能需要针对不同平台编写专属这个平台的具体实现。
  在c/c++中，不同平台的实现或者某个平台的特性往往通过`#if`, `#else`, `#endif`这类预处理指令来配合
  交叉编译达到。

  [原文地址](https://golang.org/pkg/go/build/)

 ---------------------------------------------------

### go build -tags
  
  go某种程度上也可以支持条件编译。go中的条件编译显得格外的隐蔽，并且条件编译也仅限于包级别。

  假设某个服务，对外既提供HTTP服务，也可选的提供grpc服务。以此为例来说明如何支持可选的grpc。

  go条件编译以一行特殊的 `// +build` 注释行开始。

  为了跟包文档区分开，`// +build`行后面必须跟一个空白行。

{% highlight go %}

/*错误: +build下没有用空白行分隔，build tag无法生效*/
// +build enable_rpc 
func init(){
    go StartRPCForever()
}

/*正确*/
// +build enable_rpc

func init(){
    go StartRPCForever()    
}

func StartRPCForever(){
    // ...
}
{% endhighlight %}

在编译时，我们可以通过命令行: `go build -tags=enable_rpc`来启动RPC服务。

`+build`后面的tag也有讲究: 以

    // +build linux,386 darwin,!cgo 为例:

go build子命令将其解释成: 
    
    (linux AND 386) OR (darwin AND (NOT cgo))

当`+build`以多行出现时，这些`+build`之后的标签构成**AND**关系:

    // +build linux darwin
    // +build 386

go build子命令将其解释为: 

    (linux OR darwin) AND 386

