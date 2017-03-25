---
title: golang学习笔记
author: Noodles
layout: post
permalink: /2017/03/golang_learn_p1
categories:
  - 语言
tags:
  - go
  
---

<!--more-->

  go变量定义的语法一般如下:

    var 变量名称 类型 = 表达式

  如果是局部变量,则通常可以写成:(简短方式)

    var_name := 表达式

  go变量定义时,如果初始化表达式被省略,那么将用`零`值初始化该变量.
  
  - 数值类型为0
  - 布尔类型对应为false
  - 字符串类型对应为空字符串
  - 接口或引用类型(包括slice, map, chan和函数)变量为nil
  - 任何类型的指针的零值都为nil

  **go语言中不存在未初始化的变量.** 这个特性可以帮助我们省略一些工作: 比如,在C++中,
有时你不得不为类定义一个合适的构造函数以便对象可以正确的得到初始化.
再比如,

{% highlight go %}
    func Foo(){
        content_counter := make(map[string] int)
        content_connter["hello"]++
    }
{% endhighlight %}

content_content这个字典, 不会因为一开始没有"hello"这个键而报错. 

另外, **全局变量在定义时,可以使用函数来初始化这个全局变量.**
{% highlight go %}
package main

import "fmt"

var Extern_int_val = initValue()

func initValue() int {
	fmt.Println("Call before main")
	return 100
}

func main() {
	fmt.Println("Call main")
}
{% endhighlight %}

从执行结果可知: 全局变量的初始化函数在main函数调用之前被调用.

**如果使用简短声明方式,则声明语句中必须至少有一个是新的变量,否则会报语法错误**

错误实例:

{% highlight go %}
f, err := os.Open(file)
// ...
f, err := os.Open(file)
{% endhighlight %}

### 指针

  go中:

  **返回函数中的局部变量的地址也是安全的.**
  **go在语法上不支持指针运算.**(也许是指针的加减运算会使自动回收机制难以实现)

{% highlight go %}
var p = f()

func f() *int {
    v:=1
    return &v
}
{% endhighlight %}
以上代码,调用f函数创建局部变量v, 返回局部变量的地址也是安全的.因为指针p依然引用这个变量.
每次调用f函数都将返回不同的结果.

### 变量生命周期

  **包级别变量的生命周期与整个程序的运行周期一致.**
  **局部变量的生命周期是动态的: 从每次创建一个新变量的语句开始,直到该变量不被引用为止,然后
  变量的存储空间可能会被回收.函数的参数变量和返回值都是局部变量,它们在函数每次被调用时候创建.**

### 包初始化函数
  **go为了解决包中某些数据需要特殊初始化的问题,它支持在每个文件中可以定义一个或者多个init初始化函数.也只有
  这个函数可以在同一个文件中被定义多个.每个包在解决依赖的前提下,以导入声明的顺序初始化,并且,每个包只会被初始化一次.**
