---
title: printf hex 
author: Noodles
layout: post
comments: true
permalink: /2018/07/printf-x
categories:
  - c
tags:
  - c
---

<!--more-->

 ---------------------------------------------------

### char* or unsigned char*

  《深入理解计算机系统》第二章讲述数据存储时，有一下程序。

{% highlight c %}

#include <stdio.h>

typedef unsigned char *byte_pointer;

void show_byte(byte_pointer start, size_t len)
{
	for (size_t i = 0; i < len; i++)
		printf("%.2x ", start[i]);

	printf("\n");
}

void show_int(int x)
{
    show_byte((byte_pointer)&x, sizeof(int));
}

int main()
{
    int a = 12345, b = 54321;

    show_int(a);
    show_int(b);

    return 0;
}

{% endhighlight %}

  对某种类型数据获取存储内容16进制时，通常会采取这种方法。

  以上程序打印结果：

    39 30 00 00
    31 d4 00 00

  将`unsigned char*`改为`char*`后，程序结果为：

    39 30 00 00
    31 ffffffd4 00 00

**char类型占有8位，除去最高位符号位，最大可以表示到127. `d4`的8位字节表示时，第8bit位为1。**
**printf("%.2x", start[i])会将start[i]提升为int类型，出现上述情况，是因为`d4`符号位进行了扩展。**

可通过一下简单验证：

{% highlight c %}
    char c = 0xd4;
    int i = c;
    assert(i == 0xffffffd4);
{% endhighlight %}
