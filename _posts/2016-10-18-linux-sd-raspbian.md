---
title: Linux SD 卡安装Raspbian
author: Noodles
layout: post
permalink: /2016/10/Linux-SD-Raspbian
categories:
  - RaspberryPi
tags:
  - RaspberryPi

---

### SD卡写入Raspbian ###

  本文介绍Linux下写入Raspbian操作系统。

<!--more-->

  - **Setp1.** 下载RaspberryPi操作系统镜像文件。
   [http://www.raspberrypi.org/downloads] (http://www.raspberrypi.org/downloads)
  - **Setp2.** 下载下来之后可以做散列校验。
  - **Setp3.** 解压下载下来的压缩包。里面含有Raspbian镜像。
  - **Setp4.** 执行`df -h`查看你的SD卡所在的分区。
  - **Setp5.** 执行`umount /dev/sdb4`
    执行此步骤的目的是在接下来的写入镜像过程中确保其它操作对这个SD卡进行读写。
    避免写入的镜像文件不正确。**注意:如果在以上的df命令中你看到你的SD卡显示多个分区，
    需要将所有分区一并卸载。**
  - **Setp6.** 使用`dd`将镜像写入SD卡。
  > sudo dd bs=4M status=progress if=~/Raspbian/2016-09-23-raspbian-jessie.img of=/dev/sdb
   
    **1. bs指定写入块大小。**

    **2. if指定输入文件，of指定输出设备，注意，这里的输出设备为/dev/sdb而不是sdb4。你需要将
    镜像写入整个SD卡**
    
    **3. 一旦执行以上命令，整个过程比较常而且没有进度显示。在此过程中尽量不要进行其他操作，避免写入失败**

  

