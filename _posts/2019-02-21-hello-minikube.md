---
title: hello minikube
author: Noodles
layout: post
comments: true
permalink: /2019/02/hello-minikube
categories:
  - kubernetes
tags:
  - kubernetes
---

<!--more-->

 ---------------------------------------------------

 文章记录minikube运行一个简单demo的流程，总结体会使用k8s的整体部署流程。


### 准备一个docker image
  
-------------------  
#### 简单的http服务

  {% highlight go %}

package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func HelloServer(w http.ResponseWriter, req *http.Request) {
    io.WriteString(w, "hello, world!\n")
}

func main() {
    http.HandleFunc("/hello", HelloServer)

    fmt.Println("will listen on 12345")
    log.Fatal(http.ListenAndServe(":12345", nil))
}

  {% endhighlight %}
 
-------------
#### Dockfile

  {% highlight dockerfile %}

FROM golang:1.11.5-alpine3.8 AS build
RUN mkdir -p $GOPATH/src/hello && mkdir -p /root/myapp
COPY . $GOPATH/src/hello
WORKDIR $GOPATH/src/hello
RUN go build -o /root/myapp/hello
CMD /root/myapp/hello
  {% endhighlight %}

------------------------------
#### 生成镜像并推送到docker hub

  生成镜像

    docker build -t myapp .

  为镜像打tag, 事先需要在docker hub上创建kbtest的项目。

    docker tag myapp guijiedocker/kbtest:myapp-v1
    
  将镜像推送到docker hub的kbtest项目下

    docker push guijiedocker/kbtest:myapp-v1


### setup minikube

1.创建minikube cluster
 
    minikube start

2.打开minikube dashboard
    
    minikube dashboard

3.创建deployment
    
    kubectl create deployment myapp --image=guijiedocker/kbtest:myapp-v1

4.查看deployment

    kubectl get deployments

  输出:
  > 
    NAME    READY   UP-TO-DATE   AVAILABLE   AGE
    myapp   1/1     1            1           24h

5.查看pod

    kubectl get pods

  输出：
  > 
    Name:               myapp-579c5846dc-drmj5
    Namespace:          default
    Priority:           0
    PriorityClassName:  <none>
    Node:               minikube/10.0.2.15
    Start Time:         Thu, 21 Feb 2019 17:18:58 +0800
    Labels:             app=myapp
                        pod-template-hash=579c5846dc
    Annotations:        <none>
    Status:             Running
    IP:                 172.17.0.2
    Controlled By:      ReplicaSet/myapp-579c5846dc
    Containers:
      kbtest:
        Container ID:   docker://0a725a3f5cf57c1eb23fb6b967e24e313593e5cc9034f3c85e4ec263692db283
        Image:          guijiedocker/kbtest:myapp-v1
        Image ID:       docker-pullable://guijiedocker/kbtest@sha256:79a4855a6fc532b9b194cbcccbaa55963dd13466e53b0d027eb44931b71a75e6
        Port:           <none>
        Host Port:      <none>
        State:          Running
          Started:      Fri, 22 Feb 2019 16:31:15 +0800
        Last State:     Terminated
          Reason:       Error
          Exit Code:    255
          Started:      Thu, 21 Feb 2019 17:18:59 +0800
          Finished:     Fri, 22 Feb 2019 16:30:49 +0800
        Ready:          True
        Restart Count:  1
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from default-token-hw6bk (ro)
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      ContainersReady   True
      PodScheduled      True
    Volumes:
      default-token-hw6bk:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-hw6bk
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                     node.kubernetes.io/unreachable:NoExecute for 300s
    Events:          <none>

  可以看出: 启动的pod的IP是172.17.0.2。(或者执行`kubectl get pods -o wide`) 但这个IP目前无法直接访问。它是集群内部的一个IP，需要进入到kubernate 节点中进行访问。
  可运行：
    
    minikube ssh

  进入kubernates Node之后，`ping 172.17.0.2`，测试到pod的连通性。然后执行:

    curl 'http://172.17.0.2:12345/hello'

6.创建service
  
  外部如何访问到myapp服务？kubernetes默认方案是在deployment基础之上又抽象出了`service`这个组件。执行:

    kubectl expose deployment myapp --type=LoadBalancer --port 12345

  以上命令中的port要跟我们服务的监听端口一致。


7.查看外部访问端口

    kubectl get service myapp

输出:
    NAME    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
    myapp   LoadBalancer   10.100.146.178   <pending>     12345:30173/TCP   5m50s

  或者,更直观一点:

    minikube service myapp --url

输出:
    http://192.168.99.100:30173


  测试请求：
    
    curl 'http://192.168.99.100:30173/hello'



