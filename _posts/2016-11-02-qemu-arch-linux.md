---
title: qemu虚拟机安装ArchLinux
author: Noodles
layout: post
permalink: /2016/11/ArchLinux
categories:
  - 语言
tags:
  - ArchLinux
  - qemu-kvm
  
---

<!--more-->

网上有大量的关于ArchLinux的安装，配置介绍。但用qemu虚拟机的不多。ArchLinux是一款致力于轻量级,简单,
可定制的优秀操作系统。目前诸多的发行版本中，都或多或少捆绑集成了很多我们不需要的软件模块，也许不见得
我们都可以将ArchLinux配置得像RedHat，CentOS那样通用和稳定，也不见得像Ubuntu那样易用和流行，但我们可以
在使用过程中更多的了解Linux，组装出适合自己的版本。结合自己在qemu中安装ArchLinux的实践，介绍一下在
qemu中应该注意的地方。如果想完整的了解ArchLinux，需要首先到wiki中学习一下。

 ---------------------------------------------------

#### 准备工作
 1. 安装好qemu-kvm, virt-manager等必要的虚拟机环境。
 2. 首先到Arch官网上下载iso镜像。

 ---------------------------------------------------

#### 创建一个虚拟机

 1. `Use ISO image`中选择下载好的ArchLinux镜像。
 `OS type`选择Linux, 版本可选择通用2.6.25之后的内核版本。
  
 <center><img src="/images/arch_install/1.png"></center>

 2. 内存, CPU核数, 磁盘大小等。根据实际情况。我这里设置内存1G, 1核, 磁盘大小为32G。
 
 设置妥当之后就可以启动了。启动之后会以root用户自动登陆。这类似于Windows安装盘中
 常带的PE操作系统。这个操作系统集成了必要的工具: pacman, fdisk, mount, vi等等。我们
 需要借助这个操作系统将远在网络那端的kernel，bootloader，runtime-lib，工具包等组件
 安装到我们的磁盘上。所以网络是必需的。

 ---------------------------------------------------

#### 磁盘分区

 1. 执行`fdisk -l`可以查看目前的磁盘设备和分区情况。目前我的虚拟机可以看到有`/dev/vda`
 和`/dev/loop0`两个磁盘设备。其中,`/dev/loop0`表示我们的CD设备,`/dev/vda`表示我们真正
 需要接下来分区的硬盘设备。当然，目前还没有任何分区。
 
 <center><img src="/images/arch_install/2.png"></center>

 2. 执行`cfdisk /dev/vda` 命令，选择GPT分区模式。进入cfdisk程序交互界面。选中下面的
 `new`,按下`n`,划分第一块分区，手动输入`200M`,然后选中`type`，类型选择`EFI System`,
 这个分区作为我们的boot目录。
 
 <center><img src="/images/arch_install/3.png"></center>
 
 3. 紧接着划分2G大小的，类型为`linux swap`的第二个分区，此分区接下来会作为交换分区。

 4. 接下来同样的方式创建两个类型为`linux filesystem`分区,分别作为我们的根目录和/home目录.
 根目录可以分12G，余下的作为/home目录。
 
 <center><img src="/images/arch_install/4.png"></center>

 5. 设置完成后，选择`write`，然后退出。

 ---------------------------------------------------

#### 分区格式化

  1. 执行以下命令,将1,3,4分区格式化成ext4类型分区:   
    `mkfs.ext4 /dev/vda1`  
    `mkfs.ext4 /dev/vda3`  
    `mkfs.ext4 /dev/vda4`  
 
 <center><img src="/images/arch_install/5.png"></center>

  2. 执行 **mkswap /dev/vda2**，将/dev/vda2格式化成交换分区.

  3. 对于boot分区，我们需要做如下设置:  

    `parted /dev/vda`

    (parted) `set 1 boot on`
  
 <center><img src="/images/arch_install/6.png"></center>

 ---------------------------------------------------

#### 分区格式化

  1. 挂载

    `mount /dev/vda3 /mnt`  
    `mkdir /mnt/boot`  
    `mkdir /mnt/home`  
    `mount /dev/vda1 /mnt/boot`  
    `mount /dev/vda4 /mnt/home`  

  2. 启用交换分区

    `swapon /dev/vda2`

 ---------------------------------------------------

#### 在挂载点安装Arch
 
  接下来就要使用包安装工具安装所需的基础包了。在安装之前，需要设置一下镜像的配置。

  编辑 `/etc/pacman.d/mirrorlist`
  将163的网址镜像源提升到配置文件开始处。ArchLinux包安装工具使用此配置文件开始处依次
  使用相应的镜像地址获取包，如果获取的网速太低，则尝试接下来的镜像地址。通常163的镜像
  网速非常快。所以将其作为首选地址。

    执行 `pacstrap /mnt base base-devel`  

  基础包和基础开发包安装完成之后，ArchLinux主体已经完成安装。但此时如果直接重启是无法
  进入系统的。首先需要告诉系统要挂载我们的磁盘分区。

    执行 `genfstab -p /mnt >> /mnt/etc/fstab`  

  系统启动时会读取/etc/fstab来决定如何挂载分区。
  
  <center><img src="/images/arch_install/7.png"></center>

 ---------------------------------------------------

#### 安装bootloader

  PC OS常见的bootloader主要是grub和syslinux。syslinux配置文件比较简单易懂，并且支持
  多种启动方式。这里，我们选择syslinux。

  1. 执行 `pacstrap /mnt syslinux`

  2. 执行 `syslinux-install_update -i -a -c /mnt`
  syslinux详细的使用方式和介绍请参考[wiki](https://wiki.archlinux.org/index.php/Syslinux)

  3. 执行 `arch-chroot /mnt` 切换到已经安装的arch linux环境

 ---------------------------------------------------

#### 初始配置

  1. chroot过后，修改 `/etc/locale.gen` 文件，选择**en_US.UTF-8**，将其前面注释符去掉。  
  
    执行 `locale-gen` 使之生效。  
    
  2. 设置本地语言  

    `echo LANG="en_US.UTF-8" > /etc/locale.conf`

  3. 设置区时  

    `ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`  

  4. 设置密码  

    `passwd root`

  5. 设置主机名

    `echo archlinux > /etc/hostname`

  6. 设置DHCP

    `systemctl enable dhcpcd.service`

  7. 修改 `/boot/syslinux/syslinux.cfg`  
  ** 注意: 在qemu下,需要对此配置文件做修改**
  vi /boot/syslinux/syslinux.cfg　将LABEL arch 和 LABEL archfallback
  下APPEND 一行的`root=/dev/sda3 rw`修改为`root=/dev/vda3 rw`。qemu虚拟机
  下的硬盘设备名称为vda而非sda。如果这里不修改，将无法进入系统。

  8. 执行 `cp /usr/lib/syslinux/bios/*.c32 /boot/syslinux`
　将syslinux工具下的相关模块拷贝到/boot/syslinux目录下，如果不这样也会导致
　boot阶段因syslinux找不到相关模块而启动失败。

  9. 执行 `extlinux --install /boot/syslinux`

  10. 执行 `dd conv=notrunc bs=440 count=1 if=/usr/lib/syslinux/bios/gptmbr.bin of=/dev/vda`
  此步骤将gpt信息写入到硬盘的开始440字节。

  11. 执行 `mkinitcpio -p linux`。

  至此，所有必须的工作都已完成。
  可执行:

  `umount -R /mnt`

  `swapoff /dev/vda2`
  
  `reboot`

  如果一切正常，那么就可以成功进入到ArchLinux了。

 ---------------------------------------------------
#### 安装ssh服务

    执行 `pacman -S openssh`  

    修改 /etc/ssh/sshd_config 配置文件，约在40多行，可看到
    `#PermitRootLogin prohibit-password`一行，可在相应位置
    添加 `PermitRootLogin yes` 。这样，就可以以root用户ssh连接了。  

  
  <center><img src="/images/arch_install/8.png"></center>

 ---------------------------------------------------
#### 其它
  
  如果你想安装图形环境，需要首先安装Xorg等相关服务。推荐一个轻量可定制的
  `herbstluftwm` 平铺式图形软件。

