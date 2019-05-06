---
title: 使用kubeadm搭建kubernetes集群 
author: Noodles
layout: post
comments: true
permalink: /2019/05/kubeadm-install-k8s
categories:
  - kubernetes
tags:
  - kubernetes
---

<!--more-->

 ---------------------------------------------------

  本文使用vagrant + virtualbox + kubeadm来搭建具有三个节点的k8s节点。


### 安装vagrant
----------------
  
  [vagrant下载地址](https://www.vagrantup.com/downloads.html)

  可根据平台选择预编译的安装文件。安装过程比较简单。

  然后创建一个vagrant的工作目录。我这里创建目录如下 `/Users/guijieman/sandbox/ubuntu1804`

  这里我使用ubuntu18.04作为我集群的系统。大家可根据自己的喜好，在[vagrant box](https://app.vagrantup.com/boxes/search)
  中选择适合自己的box镜像。

  然后创建`Vagrantfile`, 内容如下:
  
  {% highlight ruby %}
    # -*- mode: ruby -*-
    # vi: set ft=ruby :

    Vagrant.configure("2") do |config|

      # 一块网卡设置成 NAT 模式。因为后续需要访问外网下载安装包
      config.vm.network "public_network", type: "dhcp"

      # 将三个静态IP写入hosts中。
      config.vm.provision "shell", inline: <<-SHELL
        echo '192.168.56.200 master' >> /etc/hosts
        echo '192.168.56.201 node1' >> /etc/hosts
        echo '192.168.56.202 node2' >> /etc/hosts
      SHELL

      # master节点配置
      config.vm.define :master do |master|
        master.vm.provider "virtualbox" do |v|
              v.customize ["modifyvm", :id, "--name", "k8s-master", "--memory", "2048", "--cpus", "2"]
        end
        master.vm.box = "ubuntu/bionic64"
        master.vm.hostname = "master"
        # 增加一块 host-only 模式的静态地址
        master.vm.network "private_network", ip: "192.168.56.200"
      end

      # worker1 节点配置
      config.vm.define :node1 do |node1|
        node1.vm.provider "virtualbox" do |v|
              v.customize ["modifyvm", :id, "--name", "k8s-node1", "--memory", "1024", "--cpus", "1"]
        end
        node1.vm.box = "ubuntu/bionic64"
        node1.vm.hostname = "node1"
        node1.vm.network "private_network", ip: "192.168.56.201"
      end

      # worker2 节点配置
      config.vm.define :node2 do |node2|
        node2.vm.provider "virtualbox" do |v|
              v.customize ["modifyvm", :id, "--name", "k8s-node2", "--memory", "1024", "--cpus", "1"]
        end
        node2.vm.box = "ubuntu/bionic64"
        node2.vm.hostname = "node2"
        node2.vm.network "private_network", ip: "192.168.56.202"
      end

    end

  {% endhighlight %}


  **注意:**
  1. 这里每个节点都有多个网卡。一个设置成NAT模式，一个是host-only模式。NAT模式是因为我们后续需要访问外网下载安装包。
  host-only模式的网卡作为集群之间的网口。
  2. 每个节点都需要配置hosts。三个节点地址信息需要彼此知晓，后续安装`flannel`网络拓展时候需要告诉它集群内部互通的网口。
  3. master节点设置成2G内存，2核CPU。否则安装k8s组件时会因资源不足报错。

  配置妥当，执行`vagrant up`启动三个节点。
  
  另外，每个节点需要关闭swap。否则kubelet组件可能无法正常工作。可执行`swapoff --all`。编辑`/etc/fstab`，将swap一行注释掉，将其永久关闭。


### 安装docker及容器运行环境
----------------------------

  [docker install](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

  参照docker官网，对master节点和worker节点分别安装docker环境。安装过程从略。

  安装完成后，可执行`docker --version`查看安装版本。我这里输出如下:

  > Docker version 18.09.5, build e8ff056

  安装完成后，

  1.将docker相关的k8s容器运行环境服务加入systemd管理中:

    # Setup daemon.
    cat > /etc/docker/daemon.json <<EOF
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF

    mkdir -p /etc/systemd/system/docker.service.d

    # Restart docker.
    systemctl daemon-reload
    systemctl restart docker

  2.设置containerd

    # Configure containerd
    mkdir -p /etc/containerd
    containerd config default > /etc/containerd/config.toml
    将config.toml中，plugins.cri一节中的systemd_cgroup设置成true。

    systemctl restart containerd

#### 安装kubeadm, kubelet, kubectl 
----------------------------------

    apt-get update && apt-get install -y apt-transport-https curl
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl

#### 启动kubeadm
----------------

  这里，我们使用`flannel`来作为我们集群pod网络管理插件。这里指定跟官网一样，指定我们的pods子网网段为
    `10.244.0.0/16`。并且，指定kube-apiserver监听在`192.168.56.200:8080`.

    kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.200

    成功之后，将集群相关的配置信息拷贝到当前用户home目录下，以便使用kubectl管理集群。

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

#### 安装flannel
----------------

    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml

    
  至此，master节点工作完成。可使用:

    kubectl get pods --all-namespaces -o wide

  查看pods运行情况:

    NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
    kube-system   coredns-fb8b8dccf-nrznc          1/1     Running   0          37m
    kube-system   coredns-fb8b8dccf-xgx22          1/1     Running   0          37m
    kube-system   etcd-master                      1/1     Running   0          36m
    kube-system   kube-apiserver-master            1/1     Running   0          36m
    kube-system   kube-controller-manager-master   1/1     Running   0          36m
    kube-system   kube-flannel-ds-amd64-j4fnx      1/1     Running   0          17m
    kube-system   kube-proxy-cdp95                 1/1     Running   0          37m
    kube-system   kube-scheduler-master            1/1     Running   0          36m

#### join 工作节点
------------------

  如果`kubeadm init`执行成功, 最后会给出`kubeadm join`子命令的提示。例如:

    kubeadm join 192.168.56.200:6443 --token 13xlzm.kzwijb46c1ieocac \
    --discovery-token-ca-cert-hash sha256:ac6a94688dad8258238e9aac96d6f9d4877f0729f8306653139a1b09c14b3d5a

  在剩下的两个节点都执行一下即可。其中，生成的token 24小时一改变。
  可通过 `kubeadm token list`命令查看基本信息， 通过`kubeadm token create`生成新的token。

  最后可通过 `kubectl get nodes`查看当前集群节点信息。


