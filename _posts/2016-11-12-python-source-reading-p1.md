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

最近打算阅读python源码。跟随<<Python源码剖析>>一书开始学习。目前Python3.X已经
大行其道了，为了省力，我跟书上的版本保持一致，阅读的版本是2.5.0版本源码。

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
        struct _typeobject *ob_type;
        Py_ssize_t ob_size;
    } PyVarObject;

    typedef struct _typeobject {
        Py_ssize_t ob_refcnt; // 引用计数
        struct _typeobject *ob_type;
        Py_ssize_t ob_size;
        const char *tp_name;
        Py_ssize_t tp_basicsize, tp_itemsize;
        destructor tp_dealloc;
        printfunc tp_print;
        getattrfunc tp_getattr;
        setattrfunc tp_setattr;
        cmpfunc tp_compare;
        reprfunc tp_repr;
        PyNumberMethods *tp_as_number;
        PySequenceMethods *tp_as_sequence;
        PyMappingMethods *tp_as_mapping;
        hashfunc tp_hash;
        ternaryfunc tp_call;
        reprfunc tp_str;
        getattrofunc tp_getattro;
        setattrofunc tp_setattro;
        PyBufferProcs *tp_as_buffer;
        long tp_flags;
        const char *tp_doc;
        traverseproc tp_traverse;
        inquiry tp_clear;
        richcmpfunc tp_richcompare;
        Py_ssize_t tp_weaklistoffset;
        getiterfunc tp_iter;
        iternextfunc tp_iternext;
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
        freefunc tp_free;
        inquiry tp_is_gc;
        PyObject *tp_bases;
        PyObject *tp_mro;
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
