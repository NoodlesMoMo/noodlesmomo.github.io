---
title: supervisord install
author: Noodles
layout: post
comments: true
permalink: /2018/09/supervisor-install
categories:
  - dev-ops
tags:
  - dev-ops
---

<!--more-->

 ---------------------------------------------------

 supervisord安装及配置。记录备忘。

如果使用apt,yum包管理工具，建议直接使用包管理工具来安装，这样可省去配置开机自启。

### install pip

  [install guide](https://pip.pypa.io/en/latest/installing/)

    
    $ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py

    # python get-pip.py

  安装成功后，执行:

    # pip install supervisor

  安装完成，直接执行`supervisorctl`往往会出错，`supervisord`守护进程并没有自动启动。为了机器重启后`supervisord`无法自启
  造成无法管理进程，需要将`supervisord`加入开机自启。


### systemd

  关于`systemd`详细介绍，可参考阮一峰的blog: [systemd介绍](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)

  简单说，systemd是用来管理系统资源的一组命令。它将系统资源分为多个单元(unit)，每个单元都有一个配置文件，用来指导unit管理系统资源。

  默认配置文件目录为`/etc/systemd/system/`，但里面存放的大都是软连接，真正的配置放到`/usr/lib/systemd/system/`中。

  在这个目录下新建`supervisord.service`配置文件:

    [Unit]
    Description=Process Monitoring and Control Daemon
    After=rc-local.service nss-user-lookup.target

    [Service]
    Type=forking
    ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf

    [Install]
    WantedBy=multi-user.target

  激活开机命令:

    # systemctl enable supervisord.service

  启动:

    # systemctl start supervisord.service

  关闭:
   
    # systemctl stop supervisord.service

  reload:

    # systemctl reload supervisord.service

### supervisor 配置

   [supervisor ducument](http://www.supervisord.org)

  通常，supervisor安装完成后，会读取`/etc/supervisor.conf`配置。
  如果文件不存在，有个命令用`echo_supervisord_conf`命令生成配置:

    # echo_supervisord_conf > /etc/supervisord.conf

  一个简单的配置如下:

    [program:demo_prog]
    autorestart=True
    autostart=True 
    redirect_stderr=True
    environment=PATH="/usr/bin:/usr/local/bin"
    command=/var/ime_etcd/start.sh
    user=root
    directory=/var/demo_prog
    stdout_logfile_maxbytes = 10MB
    stdout_logfile_backups = 10 
    stdout_logfile = /tmp/demo_prog_stdout.log
    stderr_logfile_maxbytes = 20MB
    stderr_logfile_backups = 10
    stderr_logfile = /tmp/demo_prog_stderr.log

  为了便于管理，通常将每个服务的配置放到特定目录下，这取决于`/etc/serpervisord.conf中include一节的配置`
  应用配置:

    # supervisorctl update // 更新新的配置到supervisord

    # supervisorctl reload // 重新启动配置中的程序


  另外，supervisor提供了一个web界面方便用户使用:
    
    [inet_http_server]          ; inet (TCP) server disabled by default
    port=*:9001                 ; (ip_address:port specifier, *:port for all iface)
    username=admin              ; (default is no username (open server))
    password=admin              ; (default is no password (open server))

  界面如下:

  ![supervisord](/images/dev_ops/supervisor/supervisor_web.png) 

