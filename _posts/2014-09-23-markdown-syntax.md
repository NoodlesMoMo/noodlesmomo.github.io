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
  - Pygments
  
---
  
  * [区块元素](#blockelement)
    * [标题](#title)
	* [区块引用](#blockref)
	* [列表](#list)
	* [代码区块](#codeblock)
	* [分割线](#splitline)
  * 区段元素
    * [链接](#reference)
	* [强调](#strong)
	* [代码](#code)
	* [图片](#picture)

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

<!--more-->

### 常用语法

------------------------------------------------------------
#### <a name="blockelement">区块元素</a>
  一个Markdown段落是由一个或多个连续的文本行组成，**它的前后要有一个以
  上的空行（空行的定义是显示上看起来像空的，便会被视为空行，比如，若
  某一行只包含空格和制表符，则该行也会被视为空行）。**普通段落不该使用
  空格或制表符来缩进。
  Markdown允许段落内强迫换行（插入换行符），这个特性和其他大部分的text-
  to-HTML格式不同，其他的格式会把每行都转换成`<br/>`标签。
  如果你确实想要依赖Markdown来插入 `<br/>` 标签，在插入处先输入两个空格
  然后回车。

------------------------------------------------------------
#### <a name="title">标题</a>
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

------------------------------------------------------------
#### <a name="blockref">区块引用 Blockquotes</a>
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

------------------------------------------------------------
#### <a name="list">列表</a>
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

  **如果列表项目中间用空行分开，在输出HTML时，**
  **Markdown项目会将项目内容用 `<p>` 标签包起来，例如：**

    *  Bird
	*  Magic
 
  会被转换成：
    
    <ul>
	<li>Bird</li>
	<li>Magic</li>
	</ul>

  但是这个:

    *  Bird

	*  Magic
	
  会被转换成：
    <ul>
	<li><p>Bird</p></li>
	<li><p>Magic</p></li>
	</ul>

  **列表项目可以包含多个段落，每个项目下的段落都必须缩进4个空格或者1个制表符：**

    1.  This is a blockquotes with two paragraphs.
	    I like for you to be still,it is as though you were absent
		you hear me from far away,and my voice does not touch you.

		another paragraphs ...

  它的显示：

  1.  This is a blockquotes with two paragraphs.
	  I like for you to be still,it is as though you were absent
      you hear me from far away,and my voice does not touch you.
	  
	  another paragraphs ...
	
  如果你每行都有缩进，这样会好看很多。

  如果要在列表项目内放进引用，那 `>` 就需要缩进：

    *  A list item with a blockquote:
	
		> This is a blockquote
		> inside a list item.
	
  它显示起来如下：
  *  A list item with a blockquote:

	  > This is a blockquote
	  > inside a list item.

  **如果要放代码区块的话，该区块就需要缩进两次，也就是8个空格或两个制表符。**
  
    *  一个列表项包含一个列表区块：
	  <代码写在这>
	
  项目列表有可能会一不小心产生，例如：
    
    1986. What a great season!
 
  换句话说，也就是在行首出现 `数字-句点-空白` ，要避免这样的状况，你可以
  在句点前面加上反斜杠。
  
    1986\. What a great season!

------------------------------------------------------------
#### <a name="codeblock">代码区块 Code Block</a>

  如果你尝试着单纯的使用HTML语言来写一个具有代码规范的代码片段就知道有多麻烦。
  但在Markdown中，一切变得很简单：**要在Markdown中建立代码区块，只需要简单地**
  **缩进4个空格或一个制表符就可以了。**
    
    这是一个普通代码段落：
	  
	  这是一个代码区块。

  Markdown会转换成：

    <p>这是一个普通段落：</p>
	<pre><code>这是一个代码区块</code></pre>

  显示结果并不包含开始的4个空格或1个制表符。例如：

    Here is an example of AppleScript:

	  tell application "Foo"
	  	beep
	  end tell

  会被转换为：

    <p>Here is an example of AppleScript:</p>
	
	<pre><code>tell application "Foo"
		beep
	end tell
	</code></pre>

  **一个代码区块会一直持续到没有缩进的那一行(或文件结尾)**
  **在代码区块中的 `&`, `<` 和 `>` 会自动转换成HTML实体，这样的**
  **方式让你非常容易使用Markdown插入范例用的HTML原始码，只需要**
  **复制粘贴上，然后再增加缩进就可以了，剩下的Markdown会自动处理。**

------------------------------------------------------------
#### <a name="splitline">分割线 </a>
  你可以在一行中用三个以上的 `*`,`-`或下划线来建立分割线。行内不能
  有其他东西。你也可以在 `*` 或 `-` 中间插入空格：

    * * *
	***
	*****
	- - -
	------------------------------------------------------------

------------------------------------------------------------
#### <a name="reference">链接</a>
  Markdownz支持两种形式的连接语法：**行内式** 与 **参考式** 两种形式。
  不管是哪一种，链接文字都用 `[方括号]` 来标记。

  要建立一个`行内式`的链接，只要在方括号后面紧跟着圆括号并插入网址链接即可。
  如果你还想加上链接的title文字，只要在网址后面，用双引号把title文字包起来
  即可，例如：

    This is [an example](http://example.com/ "Title") inline link.
	[This link](http://example.net/) has no title attribute.

  如果你要链接到同样主机的资源，你可以使用相对路径:

    See my [About](/about/) page for details.

  `参考式` 的链接是在链接文字的括号后面再接上另一个方括号，而在第二个方括号
  里面填入用以辨识链接的标记：

    This is [an example][id] reference-style link.

  你也可以选择性地在两个方括号中间再加一个空格。
    
    This is [an example] [id] reference-style link.
 
  接着，在文件的任意处，你可以把这个标记的链接内容定义出来。

    [id]: http://example.com/ "Optional title here"

  链接内容定义的形式为：

    * 方括号(前面可以选择性的加上一个或之多三个空格来缩进)，里面输入
	  链接文字
	* 接着一个以上的空格或制表符
	* 接着链接的网址
	* 选择性的接着title内容，可以用单引号，双引号或者括弧号包着
	
  下面这三种链接的定义都是相同：

    [foo]: http://example.com/ "optional title here"
    [foo]: http://example.com/ 'optional title here'
    [foo]: http://example.com/ (optional title here)

  **链接辨别标签可以有字母，数字，空白和标点符号，但并不区分大小写**
  **以下两种写法是一样的**
    [link text][a]
	[link text][A]

  `隐式链接标记`功能让你可以省略指定链接标记，这种情形下，链接标记会被视为
  等于链接文字。要用隐式链接标记只要在链接文字后面加上一个空方括号。例如：
  
    [google][]
    
  然后定义链接内容：

    [google]: http://google.com/

  下面是一个参考链接的范例：

    I get 10 times more traffic from [google][1] than from 
	[Yahoo] [2] or [MSN] [3].

	[1]: http://google.com/ "Google"
	[2]: http://search.yahoo.com/ "Yahoo search"
	[3]: http://search.msn.com/ "MSN"

  如果改成链接名称方式：

    I get 10 times more traffic from [google][] than from 
	[Yahoo] [] or [MSN] [].

	[google]: http://google.com/ "Google"
	[Yahoo]: http://search.yahoo.com/ "Yahoo search"
	[MSN]: http://search.msn.com/ "MSN"

#### <a name="strong">强调</a>
  Markdown 使用 `*` 和 `_` 作为标记强调字词的符号。例如：

    *single asterisks*
	_single underscores_
	**double asterisks**
	__double underscores__

  **注意：如果你的 * 和 _ 两边都有空白的话，它们只会被当成普通的符号**
  **如果要在文字前后直接插入普通的星号或下划线，你可以用反斜线：**
    
    \*This text is surround by literal asterisks\*

#### <a name="code">代码</a>
  关于插入代码，这里我使用 `Pygments` 来使代码高亮。
  代码片段要放在标签对\{ % highlight language % \}和\{ % endhighlight % \}之间。
  其中的 language 为 [Pygments](http://www.pygments.org/docs/lexers/) 
  页面中的`short name`.

  举一个C hello world 程序：

    {% highlight c %}
	/* hello world demo */
	#include <stdio.h>
	int main(int argc, char** argv)
	{
		printf("Hello world!\n");
		return 0;
	}
    {% endhighlight %}
 
#### <a name="picture">图片</a>
  Markdown使用一种跟链接很相似的语法来标记图片，同样分为：`行内式`和`参考式`。

  写法为：
  * 一个感叹号
  * 紧接着一个方括号，里面放入图片的替代文字
  * 紧接着一个普通括号，里面放上图片的网址，最后还可以用引号包住并加上选择性的
    'title' 文字。

  行内式方式：
    
    ![Alt text](/path/to/image.jpg)
	![Alt text](/path/to/image.jpg "optional title")

  参考式方式：
    
    ![Alt text][id]
	[id]: url/to/image "optional title attribute"

  当然，还有一个很常用的方式就是直接用html的 `<img>` 标签。


### 参考文章

  [Markdown语法说明(简体中文版)] [1]

  [Jekyll中的语法高亮：Pygments] [2]

  [1]: http://www.wowubuntu.com/markdown/
  [2]: http://www.havee.me/internet/2013-08/support-pygments-in-jekyll.html

