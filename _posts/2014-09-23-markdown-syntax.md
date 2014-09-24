---
title: markdown学习笔记
author: Noodles
layout: post
permalink: /2014/09/markdown-syntax/
categories:
  - 语言
tags:
  - markdown
  - syntax
  - win7
  - Pygments
  
---

### Markdown简介

  前几天刚搭建了github pages,初次学习Markdown语法，一边记录一边实践。
  关于Markdown语法介绍，你可以到网上搜索相关的语法介绍。这里有一篇
  Markdown语法中文说明。说一说我的认识：

  用Markdown写Blog很方便。因为它的目的就是实现易写易读(easy-to-read
  and easy-to-write)。它吸收了一些text-to-html格式的精华，精心挑选
  一些符号作为标记供Markdown生成器生成HTML发布。它兼容HTML，但Markdown
  并非想取代HTML，并且Markdown的写法与HTML也并没有多少相似之处。Mark-
  down的理念是使文档更加容易读写改。个人认为：HTML是写给浏览器的，而
  Markdown是面向Blog作者的。

### 常用语法

#### 区块元素
  一个Markdown段落是由一个或多个连续的文本行组成，**它的前后要有一个以
  上的空行（空行的定义是显示上看起来像空的，便会被视为空行，比如，若
  某一行只包含空格和制表符，则该行也会被视为空行）。**普通段落不该使用
  空格或制表符来缩进。
  Markdown允许段落内强迫换行（插入换行符），这个特性和其他大部分的text-
  to-HTML格式不同，其他的格式会把每行都转换成`<br/>`标签。
  如果你确实想要依赖Markdown来插入 `<br/>` 标签，在插入处先输入两个空格
  然后回车。

#### 标题
  Markdown支持两种标题语法，类 Setext 和类 Alt 形式。
  
  类 Setext 形式使用底线形式，并利用 = (最高标题) 和 - (第二标题)。例如，

	This is an H1 title
	===================
	This is an H2 title
	-------------------

  **注意：不能有空格或制表符。否则会原样输出。**

  类Atx形式则是在行首插入1到6个`#`，对应标题H1~H6.
 
    #This is H1 titile.
    ## This is H2 title.
    ###### This is H6 title.

#### 区块引用 Blockquotes
  Markdown标记区块引用使用类似Email中的 `>` 引用形式。
  首先先断好行，然后再在每行的前面加上 `>` :

    > This is a blockquotes with two paragraphs.
    > I like for you to be still,it is as though you were absent
    > you hear me from far away,and my voice does not touch you.

  区块引用可以嵌套（引用内的引用），只要根据层次不同加上
  不同的 `>`。

    > This is a blockquotes with two paragraphs.
	>
    >> I like for you to be still,it is as though you were absent
    >> you hear me from far away,and my voice does not touch you.

  它看起来是这样：  
  > This is a blockquotes with two paragraphs.
  >
  >> I like for you to be still,it is as though you were absent
  >> you hear me from far away,and my voice does not touch you.

  引用块内也可以使用其他的Markdown语法，包括标题，列表，代码区块等：

    > #### 这是一个标题
	>
	> 1.  这是第一行列表项
	> 2.  这是第二行列表项
	>
	> 给出一个代码例子
	>
	> `return shell_exec("echo $input | $markdown_script");`

  > #### 这是一个标题
  >
  > 1.  这是第一行列表项
  > 2.  这是第二行列表项
  >
  > 给出一个代码例子
  >
  >```return shell_exec("echo $input | $markdown_script");```

#### 列表
  Markdown支持有序列表和无序列表。
  无序列表使用 `*`,`+` 或者 `-` 作为标记列表。

    * Red
	+ Green
	- Blue

  它显示出来如下：

  * Red
  + Green
  - Blue

  有序列表则使用数字接着英文句点：
  
    1. Bird
	2. Polly

  **注意：你在列表中标记的数字并不会影响输出HTML结果。**
  **例如，如果你写成如下形式**
  
    9. Bird
    4. Polly

  或者：
    
    1. Bird
	3. Polly

  则显示出来都为：

  1. Bird
  2. Polly
