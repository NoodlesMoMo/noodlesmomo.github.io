---
title: network namespace
author: Noodles
layout: post
comments: true
permalink: /2020/03/21/network-namespace
photos:
- http://q61qnv6m4.bkt.clouddn.com/2020/0316/istio.png
---

<!--more-->

 ---------------------------------------------------

 Linux2.6之后引入了`network namespace`，实现了网络空间隔离。像网络设备，IP协议栈，路由表，防火墙规则以及`/proc/net`, `/sys/class/net`目录树以及`socket`等都可以在不同的网络空间中，相互隔离。
 
 网络设备, socket等资源同时只能属于一个命名空间。

network namespace涉及的三个核心Linux API: `clone(CLONE_NEWNET)`, `unshare`, `setns`。为了用户操作方便，linux 提供了虚拟的以太网pair对（veth）设备及ip netns的子命令来操作命名空间。

  network namespace在逻辑上是网络协议栈的一个副本，默认情况下，子进程继承父进程的network namespace。如果不显式创建新的network namespace,所有进程都从init进程继承相同的默认network namespace.

  根据约定，命名的network namespace是可以打开的`/var/run/netns`目录下的一个对象。比如一个名为net1的network namespace对象，则可以由打开`/var/run/netns/net1`对象产生的文件描述符引用net1。通过这个文件描述符，可以操作进程的network namespace.

### ip netns help

    Usage: ip netns list
       ip netns add NAME
       ip netns set NAME NETNSID
       ip [-all] netns delete [NAME]
       ip netns identify [PID]
       ip netns pids NAME
       ip [-all] netns exec [NAME] cmd ...
       ip netns monitor
       ip netns list-id

### 查看当前进程的namespace ID

    # readlink /proc/$$/ns/net

### 创建namesapce
   
    # ip netns add ns1 # 创建一个ns1的network namespace

### 在network namespace下执行操作

    # ip netns exec ns1 <commands...>

  eg: 查看ns1下所有网卡信息

    # ip netns exec ns1 ifconfig -a

    1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

  可以看到创建的network namespace下有个默认的`DOWN`状态的lo设备。

    # ip netns exec ns1 ip link set dev lo up

    # ip netns exec ns1 ping -c 3 127.0.0.1

### 创建veth pair

    # ip link add veth0 type veth peer name veth1
    # ip link show

    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
        link/ether 08:00:27:ee:39:eb brd ff:ff:ff:ff:ff:ff
    3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
        link/ether 08:00:27:b6:05:5f brd ff:ff:ff:ff:ff:ff
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:b8:c9:88:02 brd ff:ff:ff:ff:ff:ff
    5: br-139c21686be3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
        link/ether 02:42:2a:cb:75:c4 brd ff:ff:ff:ff:ff:ff
    6: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether ca:e7:a4:9a:14:4f brd ff:ff:ff:ff:ff:ff
    7: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether fe:6d:5e:33:e8:fe brd ff:ff:ff:ff:ff:ff

  6,7有两个`DOWN`状态的`veth1@veth0`和`veth0@veth1` veth虚拟网卡。

### 将veth1加入到ns1中
    
    # ip link set veth1 netns ns1

### 配置IP，让这两个peer互通
   
    # ip addr add 10.0.0.1/24 dev veth0
    # ip link set dev veth0 up

    # ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1
    # ip netns exec ns1 ip link set dev veth1 up

    # ping -c 3 10.0.0.2
    # ip netns exec ns1 ping -c 10.0.0.1
