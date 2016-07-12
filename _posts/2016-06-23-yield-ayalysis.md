---
title: Python yield 浅析
author: Noodles
layout: post
permalink: /2016/06/Python-yield
categories:
  - 语言
tags:
  - Python 
  - Yield
  
---

### Python yield 浅析

有以下这段程序:

{% highlight python linenos %}
#!/usr/bin/env python

def countdown(n):
    print 'Count down from', n
    while n>=0:
        newvalue = (yield n)
        if newvalue is not None:
            n = newvalue
        else:
            n -= 1

c = countdown(5)
for x in c:
    print x
    if x == 5:
        print c.send(3)
{% endhighlight %}

函数执行结果为： 5 2 1 0

<!--more-->

在python里，含有yield表达式的函数构成一个生成器，当这个生成器函数被调用时，它返回一个我们所熟知的迭代器。
当生成器启动之后（可通过generator.next()或generator.send(None)启动），程序执行过程遇到第一个yield
表达式便返回，这时这个生成器的函数的执行流程被暂停，当再次调用generator.next()或generator.send(params)
时，函数从刚才暂停的地方重新开始执行。

对于一个生成器的函数体内可能含有如下形式的代码：

**[ LEFT_OPRAND = ] yield [ EXPRESSION ]**

  *注意*:
  1. 左右方括号中的内容均可以为空，例如如下代码构造一个无实际用途的定时器：

  ```python
  import time

  def one_second_clock(t=10):
    while t > 0:
      yield
      time.sleep(1)
      t -= 1

  clock = one_second_clock()
  for _ in clock:
    print 'one second has gone'
  ```
  2. **如果代码中含有形如：`val = yield 3` 形式的代码，千万不要理解成 `val = 3`!**
  val的赋值与yield表达式没有直接关系！val的值是由生成器的next()或send(params)来决定的：
  另外，第一次触发生成器时，必须调用next()或send(None)。不能调用send(X)将X传递进去，否则
  会引发TypeError错误！

  > TypeError: cant't send non-None value to a just-started generator

  另外，当我们使用for循环来迭代生成器创建的迭代器时，实际上for循环中是通过不断调用生成器的next()
  函数来实现的。

  **val的值是由生成器的next()或send(X)方法来决定的，如果调用next()，则val值为None,如果
  调用send(X)，则val的值为X。而 yield 3 只是每次程序在调用next()或send(X)时，不停地执行类似
  return 3而已，yield表达式的执行结果由next()或send(X)函数返回值接收。你需要在yield后面跟上有实际意义的表达式**
  例如：

```python
  def printHelloWorld():
      print 'Hello world!'
      return 1234

  def mygen():
      val = yield printHelloWorld()
      print val

  def test_mygen():
      gen = mygen()
      print gen.next() # start mygen
      try:
          print gen.next()
      except StopIteration as e:
          print "mygen has been stop!"

  test_mygen()
  ```
 
 ---------------------------------------------------

 回到开始的程序，借助Pycharm，单步调试以下这个程序，跟踪一下程序的主要执行流程。

**step1**
  <center><img src="/images/study/python/python_yield/step1.png"></img></center>

  程序开始运行时，首先执行第13行，创建一个生成器对象。需要注意的是，第13行执行过后
  并没有去执行countdown()函数中的print语句，这时候，生成器还没有开始运行。
  
  
**step2**
  <center><img src="/images/study/python/python_yield/step2.png"></img></center>

  当单步执行完第14行之后，for循环会调用生成器的next()触发生成器，这时，程序的执行流程
  并没有执行第15行print x而是跳转到countdown()中执行。实际上此时的x还为None，因为next()
  还并没有返回。


**step3**
  <center><img src="/images/study/python/python_yield/step3.png"></img></center>i

  程序执行到第7行时，先执行`=` 右边的yield表达式，执行完yield表达式之后立马返回。
  此处可看成将yield n 看成 return n。(此时n=5)
    
    
**step4**
  <center><img src="/images/study/python/python_yield/step4.png"></img></center>
 
  接下来，到了关键的一步：程序执行第17行之后，程序流程又转向了countdown()中，在上一步中，
  程序已经执行过`=`右边的yield表达式了，这次会执行`=`左边的表达式：也即将send()的参数3
  赋值给newvalue。


**step5**
  <center><img src="/images/study/python/python_yield/step5.png"></img></center>
 
  如上分析，newvalue已经变为send()的参数3.


**step6**
  <center><img src="/images/study/python/python_yield/step6.png"></img></center>

  程序主动权还在countdown()中，执行过第9行之后，进入下一次循环。又遇到yield语句，将3返回。


**step7**
  <center><img src="/images/study/python/python_yield/step7.png"></img></center>

  可以看到程序流程回到14行（此时14行还未执行）。上面分析yield返回3给send，但程序并没有
  用x来接收send的返回值，所以x还是刚迭代时的值5。再进行next()时，程序又一次进入countdown（
  的=右侧赋值，此时newvalue是next()的参数None，然后继续进行接下来的while循环。执行else分支，
  n的值为3，执行n-=1之后成为2。然后在遇到yield之后将值返回给next()的接受者x,所以程序会从
  5直接打印出2。接下来的过程就比较简单了。
