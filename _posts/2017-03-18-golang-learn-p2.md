---
title: golang学习笔记
author: Noodles
layout: post
permalink: /2017/03/golang_learn_p2
categories:
  - 语言
tags:
  - go
  
---

<!--more-->

### range

  range是go中的一个关键字.在go中,我们使用`range`结合`for`循环可以方便的对string, slice, map等类型进行方便的迭代.
  对于像string,slice等顺序结构,range顺序返回元素的索引和数据.对于map复合类型,range返回无序的key-value对.

 几种常见的写法:

**正常写法:**

{% highlight go %}

var array = []int{5, 4, 3, 2, 1}

for i, v := range array {
    fmt.Printf("index=%d, value=%d\n", i, v)
}

{% endhighlight %}

**缺少某一项的写法:**

{% highlight go %}

var array = []int{5, 4, 3, 2, 1}

// 如果不关心value, 则可以写成
for i := range array {
    fmt.Printf("index=%d", i)
}

// 如果不关心index
for _, value := range array {
    fmt.Printf("value=%d", value)
}

{% endhighlight %}
