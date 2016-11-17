---
title: Python源码分析1
author: Noodles
layout: post
permalink: /2016/11/python_src_reading_p1
categories:
  - 语言
tags:
  - Python
  - SourceCode
  
---

### Python源码学习

<!--more-->

最近阅读python源码。跟随《Python源码剖析》一书开始学习。书写得很棒。
作者以其惊人的耐心和透彻的分析，向我们阐述了python背后的运作原理。阅读
下来已是不易，更何况写书。拾人牙慧，我也是跟着书一起对2.5版本的源码进行
学习，以期对python有更深的认识。

 ---------------------------------------------------

 PyObject分析

 PyObject是Python对象的基石。其定义在`Include/object.h`中。
 
{% highlight C %}
    typedef struct _object {
        PyObject_HEAD
    }PyObject;
{% endhighlight %}

 `PyObject_HEAD`是个宏，为了阅读方便，我们可以使用gcc将这个头文件预编译一下。

    gcc -E -P Include/object.h -o ~/py_src/object.i

 参数-E 是只预编译，不进行汇编和连接过程。在预编译时，预处理器会为文件打上行号标记(`linemarks`), 
 加上`-P`可以抑制编译器这种行为。这样，我们可以得到一个预编译之后的文件。
 可以看到，预编译后的_object定义如下:

{% highlight c linenos %}

    typedef struct _object {
        Py_ssize_t ob_refcnt;   // 引用计数
        struct _typeobject *ob_type; // 类型指针
    } PyObject;

    typedef struct {
        Py_ssize_t ob_refcnt; // 引用计数
        struct _typeobject *ob_type; //_typeobject 真正记录类型信息
        Py_ssize_t ob_size; /* Number of items in variable part */
    } PyVarObject;

    typedef struct _typeobject {
        Py_ssize_t ob_refcnt; // 引用计数
        struct _typeobject *ob_type; // 类型对象的类型！
        Py_ssize_t ob_size; // 容纳元素的个数，并非字节数
        const char *tp_name; // 类型名称
        Py_ssize_t tp_basicsize, tp_itemsize; // 类型对象分配内存大小信息

        /* Methods of implement standard operations */
        destructor tp_dealloc;
        printfunc tp_print;
        getattrfunc tp_getattr;
        setattrfunc tp_setattr;
        cmpfunc tp_compare;
        reprfunc tp_repr;

        /* Method suites for standard classes */
        PyNumberMethods *tp_as_number;
        PySequenceMethods *tp_as_sequence;
        PyMappingMethods *tp_as_mapping;

        /* More standard operations (here for binary compatibility)*/
        hashfunc tp_hash;
        ternaryfunc tp_call;
        reprfunc tp_str;
        getattrofunc tp_getattro;
        setattrofunc tp_setattro;

        /* Functions to access object as input/output buffer */
        PyBufferProcs *tp_as_buffer;

        /* Flags to define presence of optional/expanded features */
        long tp_flags;

        /* Documentation string*/
        const char *tp_doc;

        /* call function for all accessible objects */
        traverseproc tp_traverse;

        /* delete references to contained */
        inquiry tp_clear;

        /* rich compaisons (2.1 later)*/
        richcmpfunc tp_richcompare;

        /* weak reference enabler */
        Py_ssize_t tp_weaklistoffset;

        /* Iterators*/
        getiterfunc tp_iter;
        iternextfunc tp_iternext;

        /* Attribute descriptor and subclassing stuff */
        struct PyMethodDef *tp_methods;
        struct PyMemberDef *tp_members;
        struct PyGetSetDef *tp_getset;
        struct _typeobject *tp_base;
        PyObject *tp_dict;
        descrgetfunc tp_descr_get;
        descrsetfunc tp_descr_set;
        Py_ssize_t tp_dictoffset;
        initproc tp_init;
        allocfunc tp_alloc;
        newfunc tp_new;
        freefunc tp_free; /* Low-level free-memory routine */
        inquiry tp_is_gc; /* For PyObject_IS_GC */
        PyObject *tp_bases;
        PyObject *tp_mro; /* method resolution order */
        PyObject *tp_cache;
        PyObject *tp_subclasses;
        PyObject *tp_weaklist;
        destructor tp_del;
    } PyTypeObject;
    
    // 省略其它 ...
{% endhighlight %}

要提醒的是，在object.h中定义了其它类似`Py_DEBUG`的宏，这类宏从命名也可以看出，这是为了编译调试版
条件编译的，我们如果想要将条件编译的宏展开，可使用`-D`选项:

    gcc -E -P -D Py_DEBUG Include/object.h -o ~/py_src/object_debug.i

这样预处理之后的文件中就会展开有关`Py_DEBUG`选项的宏定义。

  **在python中，一切皆为对象，包括类型。比如，int, string, list, dict 等类型也是python的对象，这些是
python内部预先定义好的，称之为`内建对象`。对于类型这种特殊的对象，我们姑且称之为 `类型对象`， 对于由
这些类型创建的实例或我们通过class创建的对象，我们称之为`实例对象`。**

  如果按照对象占用内存空间是否固定大小分类，可以有固定大小和可变大小两种情况。python内部使用`PyObject`
和`PyVarObject`两种结构来分别表示固定大小和可变大小两种对象。这两种结构体中都包含`PyTypeObject`结构指针，
正是这个类型，描述了Python世界中的万千对象。

  PyTypeObject结构有着足够的信息来描述一个对象的类型，Python用这个结构来实现面向对象中的`类`的概念。
  这个结构成员代表的含义主要分为以下几种情况,先来简要说明一下，随着后续的阅读和学习，我们再来反复研究
  这个重要的数据结构。
  - tp_name: 类型名称
  - tp_basicsize, tp_itemsize: 类型对象所占内存大小信息
  - 函数族: 这个结构体有许多的函数指针成员，它们代表了一系列该类型可以进行的操作，这是面向对象理论中的
  成员函数概念。(包括C++中的构造/析构/操作符等等)
  - 其它: 包括python中特有的文档字符串，MRO多重继承时继承顺序的解决方式，等等。

  值得注意的是: 在这个结构体中，同样存在着一个自身类型的_ob_type指针成员。这个成员就是Python中类也是对象
  (**类型对象**)的表示！那么类型对象的类型是什么呢？Python中还有一个特殊的类型: `PyType_Type`。
  
  它的定义在Object/typeobject.c里：
{% highlight c linenos %}

PyTypeObject PyType_Type = {
	PyObject_HEAD_INIT(&PyType_Type)
	0,					/* ob_size */
	"type",					/* tp_name */
	sizeof(PyHeapTypeObject),		/* tp_basicsize */
	sizeof(PyMemberDef),			/* tp_itemsize */
	(destructor)type_dealloc,		/* tp_dealloc */
	0,					/* tp_print */
	0,			 		/* tp_getattr */
	0,					/* tp_setattr */
	type_compare,				/* tp_compare */
	(reprfunc)type_repr,			/* tp_repr */
	0,					/* tp_as_number */
	0,					/* tp_as_sequence */
	0,					/* tp_as_mapping */
	(hashfunc)_Py_HashPointer,		/* tp_hash */
	(ternaryfunc)type_call,			/* tp_call */
	0,					/* tp_str */
	(getattrofunc)type_getattro,		/* tp_getattro */
	(setattrofunc)type_setattro,		/* tp_setattro */
	0,					/* tp_as_buffer */
	Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
		Py_TPFLAGS_BASETYPE,		/* tp_flags */
	type_doc,				/* tp_doc */
	(traverseproc)type_traverse,		/* tp_traverse */
	(inquiry)type_clear,			/* tp_clear */
	0,					/* tp_richcompare */
	offsetof(PyTypeObject, tp_weaklist),	/* tp_weaklistoffset */
	0,					/* tp_iter */
	0,					/* tp_iternext */
	type_methods,				/* tp_methods */
	type_members,				/* tp_members */
	type_getsets,				/* tp_getset */
	0,					/* tp_base */
	0,					/* tp_dict */
	0,					/* tp_descr_get */
	0,					/* tp_descr_set */
	offsetof(PyTypeObject, tp_dict),	/* tp_dictoffset */
	0,					/* tp_init */
	0,					/* tp_alloc */
	type_new,				/* tp_new */
	PyObject_GC_Del,        		/* tp_free */
	(inquiry)type_is_gc,			/* tp_is_gc */
};

{% endhighlight %}

  可以看到`PyType_Type`的类型也是`PyTypeObject`!， 你可以使用预编译的方法将其宏展开，
届时你会发现，这个所谓的特殊类型也没啥特殊的，仅仅是它的ob_type成员指向自身而已。

  可以看到，PyTypeObject这个结构实现了Python世界中的类型描述大一统。

  为了接下来的学习，我将尝试编译这个2.5版本的python。
  解压之后，源文件目录下有一个configure配置脚本，很熟悉的Linux三板斧: configure & make & makeinstall
  执行`configure --help`可以查看这个脚本支持的命令，

    configure --prefix=/opt/python25 --disable-shared

  执行完毕，生成Makefile。我在make时候编译Module下的getbuildinfo.c文件时失败了，原因是Makefile中开启了
版本控制，我的代码不是用svnversion版本控制工具来管理的，所以执行svnversion时会因为`Unversiond directory`
失败。需要对Makefile做少许改动:
 将`SVNVERSION`变量这行注释掉， 另外，将`Compiler options`做如下改动:

    OPT=		-DNDEBUG -g -O3 -Wall -Wstrict-prototypes
    OPT=		-DNDEBUG -g -Wall -Wstrict-prototypes --save-temp

 去掉原来的优化选项，增加`--save-temp`。尽量避免我们调试时候遇到因为优化而难以理解的代码；同时将编译过程
 中的临时文件保存下来，起始我们主要对预处理文件感兴趣。(*注意，开启--save-temp选项之后会有大量的中间过程
         文件和临时文件*)
 我使用的操作系统是ubuntu，gcc版本是4.8。在我的环境下很快编译出了Python可执行程序。

 我们可以查程序帮我们展开的`PyType_Type`结构定义:
 **typeobject.i**

{% highlight c linenos %}
PyTypeObject PyType_Type = {
 1, &PyType_Type,
 0,
 "type",
 // ...
};
{% endhighlight %}

