---
title: Alpine 修改源
author: Noodles
layout: post
comments: true
permalink: /2019/09/alpine-source
categories:
  - kubernetes
tags:
  - kubernetes
---

<!--more-->

 ---------------------------------------------------

  alpine默认源往往比较慢，这里推荐使用阿里和科大的国内源。速度很快。

### 阿里源
----------

```bash
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
```
  

### 科大源
----------

```bash
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
```
