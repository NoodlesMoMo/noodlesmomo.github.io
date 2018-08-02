---
title: javascript note 
author: Noodles
layout: post
comments: true
permalink: /2018/06/javascript
categories:
  - js
tags:
  - js
---

<!--more-->

 ---------------------------------------------------

#### 变量

  可以把任意数据类型赋值给变量，同一个变量可以反复赋值，而且可以是不同类型的变量。但注意只能用`var`申明一次。

  {% highlight js %}
  var a = 123;
  a = 'ABC';
  {% endhighlight %}

#### Array

  数组可以包含任意数据类型。并通过索引来访问每一个元素。

  要获取Array长度，直接访问`length`属性。

  **直接给Array的lengths赋一个新的值就会导致Array大小的变化**


#### 比较运算

  js有两种比较运算符：

  - `==` : 自动转换数据类型再做比较
  - `===` : 比较过程不会进行数据类型转换
  - `NaN`: 这个特殊值与其它所有值都不相等，包括它自己。当需要判断是否为Nan时，可用`isNan()`函数。

  {% highlight js %}
  isNan(NaN);
  {% endhighlight %}

---------------------------------------------------

#### undefined 与 null

  undefined 表示一个变量自然的、最原始的状态值，而 null 则表示一个变量被人为的设置为空对象，而不是原始状态。所以，在实际使用过程中，为了保证变量所代表的语义，不要对一个变量显式的赋值 undefined，当需要释放一个对象时，直接赋值为 null 即可

---------------------------------------------------

#### 对象

  - js对象的键都是字符串类型。值可以是任意数据类型。

  - 既然键都是字符串，那么如果键的值包含特殊字符，则要想访问属性值，就无法使用`.`操作符，只能使用`object['xxx']`这种形式了

  - 如果访问不存在的属性，js不会报错，而是返回`undefined`

  - 判断对象是否存在某个属性，可用`in`来判断。

  **Array也是对象，它的属性就是它的索引**

{% highlight js %}
var a = ['A', 'B', 'C'];
for (var i in a) {
console.log(i);      
console.log(a[i]);
}
{% endhighlight %}

---------------------------------------------------

#### for ... in

{% highlight js %}
var o = {
    name: "jack",
    age: 20,
    city: "Beijing"
};

for (var key in o) {
    console.log(key);    
}

{% endhighlight %}

---------------------------------------------------

#### for ... of

{% highlight js %}

var a = ['A', 'B', 'C'];
var s = new Set(['A', 'B', 'C']);
var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);

for (var x of a) { // 遍历Array
    console.log(x);
}

for (var x of s) { // 遍历Set
    console.log(x);
}

for (var x of m) { // 遍历Map
    console.log(x[0] + '=' + x[1]);
}

{% endhighlight %}
