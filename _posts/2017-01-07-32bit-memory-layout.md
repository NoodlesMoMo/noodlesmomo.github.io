---
title: Linux进程内存分布
author: Noodles
layout: post
permalink: /2017/01/32bit_memory_layout
categories:
  - 语言
tags:
  - GAS
  
---

<!--more-->

 ---------------------------------------------------

#### Linux 进程内存分布

 在网上找到一张经典32位IA32架构下程序虚拟内存布局图:
  
 <center><img src="/images/study/asm32/memory_layout.png"></center>

 32位下，操作系统为每个进程划分了4G虚拟内存空间:
 高1G的内存空间划分给内核，这部分空间称为Kernel Space。
 低3G的空间供各个应用程序使用，称为User Space。
 内核空间由系统内所有进程共享，每个进程不能直接访问这个空间，它运行在高保护级别上(ring 0)。用户进程如果
 想与内核进行数据交换，需要经过内核提供的系统调用间接完成。

 接下来是环境变量，它是用户程序运行的上下文。
 紧接着是程序的命令行参数。
 在用户空间的高区，是操作系统分配给应用程序的堆栈。由图看到，栈与命令行参数之间还有一个`Random stack offset`
 这是Linux为了防止堆栈缓冲区溢出攻击而随机产生的一个gap。这个开关在`/proc/sys/kernel/randomize_va_space`.
 它也影响下面的`Random mmap offset`和`Random brk offset`。执行:

    cat /proc/sys/kernel/randomize_va_space

 正常情况下，它的默认值为2. 这个内核属性由以下取值:

  - 0: 关闭进程地址随机化
  - 1: 表示将mmap的基址，stack和vdso页面随机化。
  - 2: 表示在1的基础上增加heap的随机化。

 `Memory Mapping Segment`是指操作系统为应用程序划分的特殊虚拟地址空间，之所以特殊，是因为这块地址空间是直接
 映射到某些文件(或文件的一部分)或类似文件的资源，而不是与物理内存相对应。但是，应用程序操作这块地址空间就像跟
 内存打交道一样。这些资源可以是典型的存储在磁盘上的文件，也可以是一个设备，一个共享内存对象或其他一些操作系统
 可以用一个文件描述符来引用的资源。比较常见的动态库就是映射到这个区。

 接下来是堆(heap)区。
 
 BSS段(Block Started by Symbol Segment): 这个段主要为应用程序中未初始化的全局变量或未初始化的静态局部变量分配
内存空间。它不真正存在于应用程序的二进制文件中，应用程序从磁盘加载到内存之后，由操作系统从ELF文件中读取其大小
信息之后再分配并初始化为0。

数据段: 存放应用程序中定义并初始化的全局变量或初始化的静态变量。

代码段: 通常用来存放程序执行代码的内存区域。这部分区域通常是只读的。在这其中，也会存放一些只读的常数变量，像
字符串等。

简单验证:

{% highlight C %}

#include <stdio.h>
#include <unistd.h>

extern char** environ;

int global_init_var = 0;
int global_uninit_var;

int main(int argc, char** argv)
{
    printf("address of environ values: %p\n", &(*environ));
    printf("address of global_init_var: %p\n", &global_init_var);
    printf("address of global_uninit_var: %p\n", &global_uninit_var);

    int i = 100;
    printf("address i: %p\n", &i);

    char **env = environ;
    while(*env){
        printf("%s\n", *env);
        env++;
    }

    getchar();

    return 0;
}

{% endhighlight %}

 <center><img src="/images/study/asm32/demo_address.png"></center>

 查看进程proc信息:
 <center><img src="/images/study/asm32/demo_maps.png"></center>

 关于proc的字段，可执行`man 5 proc`查看。

 从程序的内存映射和变量的打印地址看出，进程的内存分布确实与第一幅图一致。
