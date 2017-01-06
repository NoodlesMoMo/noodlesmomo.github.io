---
title: Tornado IOLoop 1
author: Noodles
layout: post
permalink: /2016/07/Python-IOLoop1
categories:
  - 语言
tags:
  - Python 
  - Tornado
  - IOLoop
  
---

### Tornado IOLoop 源码阅读 ###

Tornado中，IOLoop实现了底层的事件循环。在Tornado4.3+Linux环境下，当你简单很简单的写下如下代码：

```python
import tornado.ioloop

tornado.ioloop.IOLoop.instance().start()
```

Tornado对采用的是epoll()系统调用的水平触发模式。


在IOLoop类(`tornado/ioloop.py`)
里有如下类似宏定义：

<!--more-->

```python
# Constants from the epoll module
 _EPOLLIN = 0x001
 _EPOLLPRI = 0x002
 _EPOLLOUT = 0x004
 _EPOLLERR = 0x008
 _EPOLLHUP = 0x010
 _EPOLLRDHUP = 0x2000
 _EPOLLONESHOT = (1 << 30)
 _EPOLLET = (1 << 31)

 # Our events map exactly to the epoll events
 NONE = 0
 READ = _EPOLLIN
 WRITE = _EPOLLOUT
 ERROR = _EPOLLERR | _EPOLLHUP

```
Tornado部分定义了epoll()的事件类型，这些常量与epoll()系统调用的事件类型是严格一致的：可以通过
打开`/usr/include/x86_64-linux-gnu/sys`下的epoll.h EPOLL_EVENTS枚举及宏定义做对比。

```c
enum EPOLL_EVENTS
  {
    EPOLLIN = 0x001,
#define EPOLLIN EPOLLIN
    EPOLLPRI = 0x002,
#define EPOLLPRI EPOLLPRI
    EPOLLOUT = 0x004,
#define EPOLLOUT EPOLLOUT
    EPOLLRDNORM = 0x040,
#define EPOLLRDNORM EPOLLRDNORM
    EPOLLRDBAND = 0x080,
#define EPOLLRDBAND EPOLLRDBAND
    EPOLLWRNORM = 0x100,
#define EPOLLWRNORM EPOLLWRNORM
    EPOLLWRBAND = 0x200,
#define EPOLLWRBAND EPOLLWRBAND
    EPOLLMSG = 0x400,
#define EPOLLMSG EPOLLMSG
    EPOLLERR = 0x008,
#define EPOLLERR EPOLLERR
    EPOLLHUP = 0x010,
#define EPOLLHUP EPOLLHUP
    EPOLLRDHUP = 0x2000,
#define EPOLLRDHUP EPOLLRDHUP
    EPOLLWAKEUP = 1u << 29,
#define EPOLLWAKEUP EPOLLWAKEUP
    EPOLLONESHOT = 1u << 30,
#define EPOLLONESHOT EPOLLONESHOT
    EPOLLET = 1u << 31
#define EPOLLET EPOLLET
  };
```
Tornado 对epoll的常用事件重新定义为 READ、WRITE、ERROR。

Tornado使用纯python语言实现了一个事件反应堆模型，这个反应堆模型对python内置的select模块进行了封装，
当你只是简单的如开始那段程序运行时，整个事件反应堆就开始等待网络IO事件发生了。背后的调用过程是：

  1. 调用IOLoop对象的instance()静态函数，此函数返回一个IOLoop类型的单例,在执行之前添加全局锁。
在构造IOLoop对象时，首先会使用父类的`__new__()`来创建类。
  `Configurable`-->`Configurable.__new__()`

  2. 在`Configurable.__new__()`中会调用`cls.configurable_base()`，此时cls代表IOLoop这个类，
所以调用的是IOLoop.configurable_base()。IOLoop.configurable_base()返回IOLoop类对象(python中
  类也是对象)。

  3. 接着调用`Configurable.configured_class()`-->`IOLoop.configurable_default()`。在IOLoop中，有

```python
@classmethod
def configurable_default(cls):
    if hasattr(select, "epoll"):
        from tornado.platform.epoll import EPollIOLoop
        return EPollIOLoop
    if hasattr(select, "kqueue"):
        # Python 2.6+ on BSD or Mac
        from tornado.platform.kqueue import KQueueIOLoop
        return KQueueIOLoop
    from tornado.platform.select import SelectIOLoop
    return SelectIOLoop
```

Tornado使用Configurable类实现平台的一致性(select/poll/epoll/kqueue)，在Linux下，configurable_default
函数返回EPollIOLoop类对象，此类对PoolIOLoop类做了简单的包装，
它是真正的背后英雄。至此，IOLoop.instance()会接着调用EPollIOLoop类的initialize()。

```python
class EPollIOLoop(PollIOLoop):
    def initialize(self, **kwargs):
        super(EPollIOLoop, self).initialize(impl=select.epoll(), **kwargs)
```

最终调用到
`PollIOLoop.initialize()`，此函数定义如下：

```python
def initialize(self, impl, time_func=None, **kwargs):
    super(PollIOLoop, self).initialize(**kwargs)
    self._impl = impl # _impl 代表 select.epoll()
    if hasattr(self._impl, 'fileno'):  # 如果当前的 epoll 对象中有打开文件描述符，设置 CLOSE_ON_EXEC
        set_close_exec(self._impl.fileno())
    self.time_func = time_func or time.time
    self._handlers = {} # 存储所有的 Handler
    self._events = {} # 存储事件
    self._callbacks = [] # 存储回调
    self._callback_lock = threading.Lock()
    self._timeouts = [] #存储_Timeout对象，包含超时时间戳和与之相关的超时回调函数
    self._cancellations = 0
    self._running = False # 启动标志
    self._stopped = False # 停止标志
    self._closing = False # 关闭标志
    self._thread_ident = None
    self._blocking_signal_threshold = None
    self._timeout_counter = itertools.count()

    # Create a pipe that we send bogus data to when we want to wake
    # the I/O loop when it is idle
    self._waker = Waker()
    self.add_handler(self._waker.fileno(),
                     lambda fd, events: self._waker.consume(),
                     self.READ)

```
函数最后创建了一个pipe,并将读一端加入回调中，这个管道只是发送无用数据(发送‘x’)。这样做可以在Tornado
进入事件堵塞时，在没有网络IO事件或超时发生时，调用stop()时，主动发送数据可以唤醒堵塞，快速结束IOLoop.

```python
def add_handler(self, fd, handler, events):
    fd, obj = self.split_fd(fd) # 从文件对象中分离出文件描述符
    self._handlers[fd] = (obj, stack_context.wrap(handler)) # 将文件描述符添加到 _handlers 中
    self._impl.register(fd, events | self.ERROR) # 注册这个IO
```

add_handler中，使用stack_context.wrap对回调处理器进行了包装，使处理器携带所在线程调用栈信息，这样当有
异常出现时，Tornado可以倾泄调用栈信息。(StackContext类实现以后单独剖析)

至此，IOLoop已经前戏准备就绪，接下来进入IOLoop.start().

```python
def start(self):
        if self._running:
            raise RuntimeError("IOLoop is already running")
        self._setup_logging()
        if self._stopped:
            self._stopped = False
            return
        old_current = getattr(IOLoop._current, "instance", None)
        IOLoop._current.instance = self
        self._thread_ident = thread.get_ident() # 获得当前线程标识符
        self._running = True

        old_wakeup_fd = None
        if hasattr(signal, 'set_wakeup_fd') and os.name == 'posix':
            # requires python 2.6+, unix.  set_wakeup_fd exists but crashes
            # the python process on windows.
            # 这段对信号的处理没有看明白为什么要这么做。根据 python.signal模块描述：
            # 将描述符设置为signal的唤醒描述符，当一个信号被接收时,signal.set_wakeup_fd(fd)
            # 会向管道发送'\0',这样可以用来唤醒select或poll，允许信号被完全处理。
            try:
                old_wakeup_fd = signal.set_wakeup_fd(self._waker.write_fileno())
                if old_wakeup_fd != -1:
                    # Already set, restore previous value.  This is a little racy,
                    # but there's no clean get_wakeup_fd and in real use the
                    # IOLoop is just started once at the beginning.
                    signal.set_wakeup_fd(old_wakeup_fd)
                    old_wakeup_fd = None
            except ValueError:
                # Non-main thread, or the previous value of wakeup_fd
                # is no longer valid.
                old_wakeup_fd = None

        try:
            while True: # 事件循环开始
                # 加锁，保护self._callbacks,因为self._callbacks可能会跨线程，IOLoop.add_callback()
                # 可能会在别的线程中调用，所以Tornado在这里只是尽快的将self._callback中的回调函数转移进来，
                # 并不急于处理回调，然后释放掉对self._callback的占有权。
                with self._callback_lock:
                    callbacks = self._callbacks
                    self._callbacks = []

                # Add any timeouts that have come due to the callback list.
                # Do not run anything until we have determined which ones
                # are ready, so timeouts that call add_timeout cannot
                # schedule anything in this iteration.
                # self._timeouts经过最小堆排过序，以下处理也是将self._timeouts中的超时
                # 对象尽快的转移进 due_timeouts。并不急于处理超时回调
                due_timeouts = []
                if self._timeouts:
                    now = self.time()
                    while self._timeouts:
                        # 如果超时无回调，则只是从self._timeouts移出。注意，后续对self._timeouts
                        # 的处理要保证其存储堆排有效
                        if self._timeouts[0].callback is None:
                            # The timeout was cancelled.  Note that the
                            # cancellation check is repeated below for timeouts
                            # that are cancelled by another timeout or callback.
                            heapq.heappop(self._timeouts)
                            self._cancellations -= 1
                        elif self._timeouts[0].deadline <= now: # 有超时发生
                            due_timeouts.append(heapq.heappop(self._timeouts))
                        else:
                            break
                    # 当 self._cancellations 记录过大，则重新整理 self._timeouts，清除
                    # 其中没有设置回调处理的超时对象，并使用 heap.heapify使之最小堆化
                    if (self._cancellations > 512
                            and self._cancellations > (len(self._timeouts) >> 1)):
                        # Clean up the timeout queue when it gets large and it's
                        # more than half cancellations.
                        self._cancellations = 0
                        self._timeouts = [x for x in self._timeouts
                                          if x.callback is not None]
                        heapq.heapify(self._timeouts)

                # 开始对收集的回调和超时事件进行处理
                for callback in callbacks:
                    self._run_callback(callback)
                for timeout in due_timeouts:
                    if timeout.callback is not None:
                        self._run_callback(timeout.callback)
                # Closures may be holding on to a lot of memory, so allow
                # them to be freed before we go into our poll wait.
                # 释放内存。（看python GC 心情吧）
                callbacks = callback = due_timeouts = timeout = None

                # 检查之前清空的self._callbacks是否有新的回调。
                # 如果有，则将poll_timeout设为0，告诉接下来的poll(poll_timeout)不要等待，
                # 这样可快速进行下次循环时处理刚添加的回调
                if self._callbacks:
                    # If any callbacks or timeouts called add_callback,
                    # we don't want to wait in poll() before we run them.
                    poll_timeout = 0.0
                elif self._timeouts:
                    # If there are any timeouts, schedule the first one.
                    # Use self.time() instead of 'now' to account for time
                    # spent running callbacks.
                    poll_timeout = self._timeouts[0].deadline - self.time()
                    poll_timeout = max(0, min(poll_timeout, _POLL_TIMEOUT))
                else:
                    # No timeouts and no callbacks, so use the default.
                    poll_timeout = _POLL_TIMEOUT # _POLL_TIMEOUT为3600(秒)

                if not self._running:
                    break

                if self._blocking_signal_threshold is not None:
                    # clear alarm so it doesn't fire while poll is waiting for
                    # events.
                    signal.setitimer(signal.ITIMER_REAL, 0, 0)

                try:
                    event_pairs = self._impl.poll(poll_timeout)
                except Exception as e:
                    # Depending on python version and IOLoop implementation,
                    # different exception types may be thrown and there are
                    # two ways EINTR might be signaled:
                    # * e.errno == errno.EINTR
                    # * e.args is like (errno.EINTR, 'Interrupted system call')
                    # 与UNIX socket API处理方式一样，如果遇到EINTR错误，表示此系统调用因信号打断，
                    # select/poll/epoll等有些系统不会主动继续监听I/O事件，忽略此信号，再次投入即可
                    if errno_from_exception(e) == errno.EINTR:
                        continue
                    else:
                        raise

                if self._blocking_signal_threshold is not None:
                    signal.setitimer(signal.ITIMER_REAL,
                                     self._blocking_signal_threshold, 0)

                # Pop one fd at a time from the set of pending fds and run
                # its handler. Since that handler may perform actions on
                # other file descriptors, there may be reentrant calls to
                # this IOLoop that update self._events
                # self._events中存放{fd: IO_EVENTS}
                # self._handers中存放{fd: (fd_obj, statck_context.wrap(handler))}
                # 当网络IO发生时，select.epoll()返回描述符和在此描述符上发生的事件，
                # 然后根据描述符可以在self._handlers中找到此描述符对应的处理器。
                # 接下来就可以将描述符对象和与之相关的网络事件传给处理器处理就行了。
                self._events.update(event_pairs)
                while self._events:
                    fd, events = self._events.popitem()
                    try:
                        fd_obj, handler_func = self._handlers[fd]
                        handler_func(fd_obj, events)
                    except (OSError, IOError) as e:
                        if errno_from_exception(e) == errno.EPIPE:
                            # Happens when the client closes the connection
                            pass
                        else:
                            self.handle_callback_exception(self._handlers.get(fd))
                    except Exception:
                        self.handle_callback_exception(self._handlers.get(fd))
                fd_obj = handler_func = None

        finally:
            # reset the stopped flag so another start/stop pair can be issued
            self._stopped = False
            if self._blocking_signal_threshold is not None:
                signal.setitimer(signal.ITIMER_REAL, 0, 0)
            IOLoop._current.instance = old_current
            if old_wakeup_fd is not None:
                signal.set_wakeup_fd(old_wakeup_fd)
```

Tornado的IOLoop事件循环机制跟libevent是何其相似！
最后，顺便说一下stop()函数：

```python
def stop(self):
    self._running = False
    self._stopped = True
    self._waker.wake()
```

    设置与循环相关的标志，然后调用self._waker.wake()发送一个'x'，唤醒epoll()，当再次执行循环时，
    条件不在成立，结束循环。

还有一些知识点还没有搞清楚，另外，还有一些重要函数，像.`_run_callback(callback)`还没有跟下去。
这是协程的关键。后续会继续分析，争取将Tornado整个框架模块分析明白透彻。
