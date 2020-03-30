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

  本文使用vagrant + virtualbox + kubeadm来搭建具有三个节点的k8s节点。

<!--more-->

 ---------------------------------------------------


### 安装vagrant
----------------
  
  [vagrant下载地址](https://www.vagrantup.com/downloads.html)

  可根据平台选择预编译的安装文件。安装过程比较简单。

  然后创建一个vagrant的工作目录。我这里创建目录如下 `/Users/guijieman/sandbox/k8s`

  这里我使用ubuntu18.04作为我集群的系统。可在[vagrant box](https://app.vagrantup.com/boxes/search)
  中选择适合自己的box镜像。

  这里我的虚拟机配置是: master 2C2G， worker为1C2G。

  master节点IP: 192.168.56.200。两台worker节点分别为192.168.56.201, 192.168.56.202.

  在运行之前，vagrant需要安装host-manager插件。

  然后创建`Vagrantfile`, 内容如下:
  
  {% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.


BOX_IMAGE = "ubuntu/bionic64" # ubuntu 18.04 LTS
SETUP_MASTER = true
SETUP_WORKERS = true
WORKER_NODE_COUNT = 2
MASTER_IP = "192.168.56.200"
WORKER_IP_NETWORK = "192.168.56."
POD_NETWORK_CIDR = "10.244.0.0/16"
KUBE_TOKEN = "0x0000.0123456789abcdef" # dummy token


$kube_worker_script = <<EOF
sudo swapoff -a
sudo kubeadm join --token #{KUBE_TOKEN} #{MASTER_IP}:6443 --discovery-token-unsafe-skip-ca-verification
EOF


$kube_master_script = <<EOF
sudo swapoff -a
sysctl net.bridge.bridge-nf-call-iptables=1
sudo kubeadm init --apiserver-advertise-address=#{MASTER_IP} --pod-network-cidr=#{POD_NETWORK_CIDR} --token #{KUBE_TOKEN} --token-ttl 0
mkdir -p $HOME/.kube
sudo cp -Rf /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f kube-flannel.yaml
kubectl taint nodes --all node-role.kubernetes.io/master-
EOF


Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  # config.vm.box = "ubuntu/bionic64"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
  #


  config.vm.box = BOX_IMAGE
  config.vm.box_check_update = false

  config.vm.provider "virtualbox" do |v|
    v.cpus = 1
    v.memory = "1024"
  end

  config.vm.provision :shell, :path => "ubuntu_bootstrap.sh"

  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true
  #config.vm.network "public_network", type: "dhcp", bridge: "en0: Wi-Fi (AirPort)"

  if SETUP_MASTER
    config.vm.synced_folder "./master_home", "/vagrant"
    config.vm.define "master" do |v|
      v.vm.hostname = "master"
      v.vm.network :private_network, ip: MASTER_IP
      v.vm.provider :virtualbox do |vb|
        vb.name = "master"
        vb.customize ["modifyvm", :id, "--cpus", "2"]
        vb.customize ["modifyvm", :id, "--memory", "2048"]
      end

      #v.vm.provision :shell, inline: $kube_master_script

    end
  end


  if SETUP_WORKERS
    (1..WORKER_NODE_COUNT).each do |i|
      config.vm.define "worker#{i}" do |v|
        v.vm.hostname = "worker#{i}"
        v.vm.network :private_network, ip: WORKER_IP_NETWORK + "#{i+200}"

        #v.vm.provision :shell, inline: $kube_worker_script

        v.vm.provider :virtualbox do |vb|
          vb.name = "worker#{i}"
          vb.customize ["modifyvm", :id, "--memory", "2048"]
        end

      end
    end
  end

end

  {% endhighlight %}

  其中，`ubuntu_bootstrap.sh`脚本内容如下:

  {% highlight shell %}
#!/bin/bash
# Source: http://kubernetes.io/docs/getting-started-guides/kubeadm/

# disable swap
sudo swapoff -a
sudo sed -i '/swap/s/^/#/g' /etc/fstab

# use iptables legacy
#update-alternatives --set iptables /usr/sbin/iptables-legacy
#update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
#update-alternatives --set arptables /usr/sbin/arptables-legacy
#update-alternatives --set ebtables /usr/sbin/ebtables-legacy

# Install Docker CE
## Set up the repository:
### Install packages to allow apt to use a repository over HTTPS
sudo apt-get update &&  sudo apt-get install \
  apt-transport-https ca-certificates curl software-properties-common

### Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

### Add Docker apt repository.
add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

## Install Docker CE.
sudo apt-get update && sudo apt-get install -y \
  containerd.io=1.2.10-3 \
  docker-ce=5:19.03.4~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.4~3-0~ubuntu-$(lsb_release -cs)

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


### install kubelet, kubadm, kubectl

sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

  {% endhighlight %}

  **⚠️注意:**

  1. k8s master的最低配置为2C2G。worker内存强烈建议配置成2G，否则非常容易造成因资源受限导致node not-ready
  2. 这里每个节点都有多个网卡。一个设置成NAT模式，一个是host-only模式。NAT模式是因为我们后续需要访问外网下载安装包。
  host-only模式的网卡作为集群之间的网口。
  3. 每个节点都需要配置hosts。三个节点地址信息需要彼此知晓，后续安装`flannel`网络拓展时候需要告诉它集群内部互通的网口。

  配置妥当，执行`vagrant up`启动三个节点。

#### 启动kubeadm
----------------

##### 关闭swap

    sudo swapoff -a

##### 打开内核网络转发

    sysctl net.bridge.bridge-nf-call-iptables=1

##### 启动kubeadm

    kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.200

  
  成功之后，将集群相关的配置信息拷贝到当前用户home目录下，以便使用kubectl管理集群。

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

#### 安装flannel
----------------
    
    wget https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
  
  大约在192行左右开始稍作修改:
  {% highlight yaml %}
 # ...
  containers:
  - name: kube-flannel
    image: quay.io/coreos/flannel:v0.11.0-amd64
    command:
    - opt/bin/flanneld
    args:
    - --ip-masq
    - --kube-subnet-mgr
    - --iface=enp0s8 # 指定flanneld工作的的网卡
    resources:
      requests:
        # ...
  {% endhighlight %}

  **⚠️: 这里一定要指定正确，否则不同node的pod很可能因为默认使用第一个网卡（NAT）而导致不互通。**

    kubectl apply -f kube-flannel.yaml 
    
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

    kubeadm join 192.168.56.200:6443 --token 0x0000.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:ac6a94688dad8258238e9aac96d6f9d4877f0729f8306653139a1b09c14b3d5a

  在剩下的两个节点都执行一下即可。其中，默认生成的token 24小时一改变。这里指定ttl为0，设置成永不过期。
  可通过 `kubeadm token list`命令查看基本信息， 通过`kubeadm token create`生成新的token。

  最后可通过 `kubectl get nodes`查看当前集群节点信息。


