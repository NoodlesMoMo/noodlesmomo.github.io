---
title: Tornado autoreload
author: Noodles
layout: post
permalink: /2016/08/Python-Autoreload
categories:
  - 语言
tags:
  - Python
  - Tornado
  - Autoreload

---

在开发Tornado(v4.3)时，如果设置了Application的debug参数，Tornado会自动启用autoreload
机制: 当项目中有脚本修改时，Tornado会自动重启并且reload所有相关的模块，不用修改
完成之后再手动停止-运行了，比较方便调试。如何实现的？其实很简单: Tornado会将所有
已加载的模块存入一个字典，记录了文件名称和文件修改时间，然后将其添加到IOLoop中每
隔一段时间检查一下所有文件事件有没有被修改，如果某个文件修改时间与字典中的时间不一致，
则Tornado会自动停止-重新加载运行。这可以通过touch某个脚本来验证。

下面简单分析一下。

<!--more-->

Web.py的Application构造式中，有如下代码。


```python
class Application(httputil.HTTPServerConnectionDelegate):
        def __init__(self, handlers=None, default_host="", transforms=None,
                                 **settings):
            # ... ...
            self.settings = settings

            # ... ...
            if self.settings.get('debug'):
                self.settings.setdefault('autoreload', True)
                self.settings.setdefault('compiled_template_cache', False)
                self.settings.setdefault('static_hash_cache', False)
                self.settings.setdefault('serve_traceback', True)

            # Automatically reload modified modules
            if self.settings.get('autoreload'):
                from tornado import autoreload
                autoreload.start()
```

如果debug为True，则Tornado会导入autoreload模块，并执行其中的start()函数。

```python
# tornado/autoreload.py

def start(io_loop=None, check_time=500):
    io_loop = io_loop or ioloop.IOLoop.current()
    if io_loop in _io_loops:
        return
    _io_loops[io_loop] = True
    if len(_io_loops) > 1:
        gen_log.warning("tornado.autoreload started more than once in the same process")
    if _has_execv:
        add_reload_hook(functools.partial(io_loop.close, all_fds=True))
    modify_times = {}
    callback = functools.partial(_reload_on_update, modify_times)
    scheduler = ioloop.PeriodicCallback(callback, check_time, io_loop=io_loop)
    scheduler.start()
```

在start()中，首选获取IOLoop实例，然后判断_io_loops有没有此实例，如果有，则返回，保证start()
只对某个IOLoop实例操作一次。其中，`_io_loops`是一个模块级的`weakref.WeakKeyDictionary()`对象，
用来存储`_io_loops`的弱引用。如果你熟悉C++ Boost库的weak_ptr，一定对此不陌生，它并不增加对象
的引用计数，主要功能就是为了检测对象是否存活。我们知道，python依靠引用计数来实现回收:当对象引用计数
为0或只剩下弱引用时，GC会将内存回收。所以，如果我们使用普通的容器（比如map）存储对象引用显然不合适,
而weakref.WeakKeyDictionary，weakref.WeakValueDictionary等内部使用弱引用来帮助我们解决这个
问题，详情请参考python文档。

  接下来调用`add_reload_hook`，在加载时挂载`io_loop.close`,保证IOLoop停止时关闭所有描述符，
防止描述符泄露。

  接下来将`_reload_on_update`回调添加到IOLoop.PeriodicCallback中。PeriodicCallback将
`_reload_on_update`添加到IOLoop超时管理中达到定时执行的目的。这里超时时间是500ms.
*(如果我们在实际生产环境部署时，要记得关闭debug选项，这样可以提升系统性能)*

  看一下`_reload_on_update`实现：

```python
def _reload_on_update(modify_times):
    # 如果已经尝试加载过了，那就不再纠缠了
    if _reload_attempted:
        return
    if process.task_id() is not None:
        return
    # sys.modules是个全局字典，记录着python已经导入到内存中的模块，它起到缓冲作用，当首次导入
    # 某个模块时，会在这个字典里记录一下，当第二次导入这个模块时，python会先到这里面查找。
    for module in list(sys.modules.values()):
        if not isinstance(module, types.ModuleType):
            continue
        path = getattr(module, "__file__", None)
        if not path:
            continue
        if path.endswith(".pyc") or path.endswith(".pyo"):
            path = path[:-1]
        # 检查modify_times中的path项，如果没有，则在modify_times中增加一项检查项
        _check_file(modify_times, path)
    for path in _watched_files:
        _check_file(modify_times, path)
```

```python
def _check_file(modify_times, path):
    try:
        # 获取文件修改时间
        modified = os.stat(path).st_mtime
    except Exception:
        return
    # 如果modify_times中没有此文件，则增添一项
    if path not in modify_times:
        modify_times[path] = modified
        return
    # 如果文件的修改时间改变了，则调用_reload
    if modify_times[path] != modified:
        gen_log.info("%s modified; restarting server", path)
        _reload()
```

```python
def _reload():
    # 将_reload_attempted设为True. _reload_on_update会根据这个值判断是否已经重新加载过了
    global _reload_attempted
    _reload_attempted = True
    # 将_reload_hooks中注册的hooks调用一遍
    for fn in _reload_hooks:
        fn()
    if hasattr(signal, "setitimer"):
        # 清除alarm信号，防止在接下来的execv时产生alarm信号
        signal.setitimer(signal.ITIMER_REAL, 0, 0)

    path_prefix = '.' + os.pathsep
    if (sys.path[0] == '' and
            not os.environ.get("PYTHONPATH", "").startswith(path_prefix)):
        os.environ["PYTHONPATH"] = (path_prefix +
                                    os.environ.get("PYTHONPATH", ""))
    # 此处针对 windows 平台的特殊处理
    if not _has_execv:
        subprocess.Popen([sys.executable] + sys.argv)
        sys.exit(0)
    else:
        try:
            # 重新执行Tornado项目工程
            # sys.executable为python解释器， sys.argv为项目的起始脚本
            os.execv(sys.executable, [sys.executable] + sys.argv)
        except OSError:
            # Mac OS X versions prior to 10.6 do not support execv in
            # a process that contains multiple threads.  Instead of
            # re-executing in the current process, start a new one
            # and cause the current process to exit.  This isn't
            # ideal since the new process is detached from the parent
            # terminal and thus cannot easily be killed with ctrl-C,
            # but it's better than not being able to autoreload at
            # all.
            # Unfortunately the errno returned in this case does not
            # appear to be consistent, so we can't easily check for
            # this error specifically.
            os.spawnv(os.P_NOWAIT, sys.executable,
                      [sys.executable] + sys.argv)
            # At this point the IOLoop has been closed and finally
            # blocks will experience errors if we allow the stack to
            # unwind, so just exit uncleanly.
            os._exit(0)
```

  这里，整个自动加载机制脉络基本已经清楚了，这其中引入了另一个比较重要的知识点，python的导入机制。
找个时间，好好研究以下～
