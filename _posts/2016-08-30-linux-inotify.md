---
title: Linux inotify mechanism
author: Noodles
layout: post
permalink: /2016/08/Linux-Inotify
categories:
  - Linux program
tags:
  - inotify

---

  Linux从2.6.13开始，提供了inotify机制，允许应用程序监视文件系统的事件。inotify机制取代了
之前的dnotify。这种机制在某些场合比较有用：例如，一个特定目录增加或者移除了图片时，图片管理
软件可以自动的加载显示和移除展示图片；当配置文件被修改时，相关的服务软件可以重新加载配置。
另外，我们可以用这个机制来开发一个日志收割分发器：监视特定的一系列log文件，当log文件有变动时，
可以将log增量及时的读取并分发下去。inotify的有一个类似网络套接口的listen文件描述符，此文件
描述符可以被添加到select/poll/epoll等函数中。上篇我们曾分析过tornado的自动加载机制，tornado的
自动加载机制依靠定时来存储、遍历对比文件修改时间，效率低下。另外，有一个pyinotify的第三方模块，
方便python程序的开发。

  本文只分析linux的inotify机制及其中的一些我在使用过程中遇到的注意事项。

<!--more-->

在使用inotify机制时，主要涉及到如下几个API和数据类型：

- `inotify_init()`和`inotify_init1()`: 用来创建一个inotify实例，是inotify机制后续继续的基础，调用成功返回一个文件描述符，
此文件描述符可以添加到select/poll/epoll中，可以类比于`epoll_create()`。`inotify_init1()`应该是后来加入的，
它带有一个flags参数，你可以让返回的描述符通过将flags设置`IN_NONBLOCK`和`IN_CLOEXEC`来影响其对应的描述符
属性。当flags为0时，它等同于`inotify_init()`。
- `inotify_add_watch(int fd, const char* pathname, uint32_t mask)`: 你可以将其等同于`epoll_ctl(EPOLL_CTL_ADD)`,
它用来将pathname指定的文件添加到监视描述符中，第一个参数fd就是`inotify_init()`返回的描述符。mask是事件类型
的掩码，它的详细定义你可以通过man inotify来查看其详细说明及完整定义。如果成功，此函数返回一个*watch descriptor*
Linux使用此描述符与pathname相关联。注意这个描述符与inotify_init()返回描述符之间的不同。你需要为此描述符建立一个
类似map<int, const char* pathname>的结构。因为你极有可能在必要时对其修改或将其移除，届时，你需要知道这个描述符
是跟那个文件名称相关联的。
- `inotify_rm_watch()`: 同样，此函数可类比`epoll_ctl(EPOLL_CTL_DEL)`。它用来把wd从监听描述符队列中移除。
- `inotify_event`: 这是一个与以上函数相关的事件结构。它包含了*watch descriptor*, *mask*, *name*, *len*等事件
属性。它是通过从inotify_init()返回的描述符中读取得到的。当有相关的事件发生时，你可以使用read从描述符中读取。

以上说了两种不同的描述符，一类是`inotify_init()`函数返回的，另一类是调用`inotify_add_watch()`返回的，它们之间
的关系如下图所示：(可类比网络中的监听描述符与已链接的普通描述符)

  <center><img src="/images/inotify/inotify_watch_fd.png"></center>

接下来说一下`inotify_event`这个数据类型,它的定义如下:

```c
/* Structure describing an inotify event.  */
struct inotify_event
{
  int wd;               /* Watch descriptor.  */
  uint32_t mask;        /* Watch mask.  */
  uint32_t cookie;      /* Cookie to synchronize two events.  */
  uint32_t len;         /* Length (including NULs) of name.  */
  char name __flexarr;  /* Name.  */
};
```

**注意： len字段用来指定从name字段开始到整个inotify_event结束的长度，它并不一定等于strlen(name)。因为inotify_event
结构会有对齐， 所以导致name之后会填充不定长的无效字节。**

`char name __flexarr`： 从定义可以看出， name字段为柔性数组(char name[])，它并不占用内存空间，所以一个inotify_event的
内存大小计算方法为: **sizeof(struct inotify_event)+inotify_event.len**

cookie 字段用来将相关联的事件绑定到一起，目前它只当文件重命名时有用。

另外，对`inotify_event`结构特别需要注意的地方是：**如果调用inotify_add_watch()添加的是具体的文件名，而不是一个目录，那么此结构中
的name字段为NULL，所以你如果你为了效率只监视必要的文件，那么就需要在调用inotify_add_watch()时就将wd与name之间的关系建立保存好以备
必要时候查询。**

  以下是我在[stackoverflow][1]中查到的说明：
  [1]: http://stackoverflow.com/questions/7957132/inotify-inotify-event-event-name-is-empty

> The name field is only present when an event is returned for a file inside a watched directory; it identifies the file > pathname relative to the watched directory. This pathname is null-terminated, and may include further null bytes to align subsequent reads to a suitable address boundary.

一个简单的demo:

```c
#include <sys/inotify.h>
#include <limits.h>
static void
displayInotifyEvent(struct inotify_event *i)
{
  printf("wd =%2d; ", i->wd);

  if (i->cookie > 0)
    printf("cookie =%4d; ", i->cookie);

  printf("mask = ");
  if (i->mask & IN_ACCESS)        printf("IN_ACCESS");
  if (i->mask & IN_ATTRIB)        printf("IN_ATTRIB");
  if (i->mask & IN_CLOSE_NOWRITE) printf("IN_CLOSE_NOWRITE");
  if (i->mask & IN_CLOSE_WRITE)   printf("IN_CLOSE_WRITE");
  if (i->mask & IN_CREATE)        printf("IN_CREATE");
  if (i->mask & IN_DELETE)        printf("IN_DELETE");
  if (i->mask & IN_DELETE_SELF)   printf("IN_DELETE_SELF");
  if (i->mask & IN_IGNORED)       printf("IN_IGNORED");
  if (i->mask & IN_ISDIR)         printf("IN_ISDIR");
  if (i->mask & IN_MODIFY)        printf("IN_MODIFY");
  if (i->mask & IN_MOVE_SELF)     printf("IN_MOVE_SELF");
  if (i->mask & IN_MOVED_FROM)    printf("IN_MOVED_FROM");
  if (i->mask & IN_MOVED_TO)      printf("IN_MOVED_TO");
  if (i->mask & IN_OPEN)          printf("IN_OPEN");
  if (i->mask & IN_Q_OVERFLOW)    printf("IN_Q_OVERFLOW");
  if (i->mask & IN_UNMOUNT)       printf("IN_UNMOUNT");
  printf("\n");
}

#define BUF_LEN (10 * (sizeof(struct inotify_event) + NAME_MAX + 1))

int main(int argc, char *argv[])
{
  int inotifyFd, wd, j;
  char buf[BUF_LEN];
  ssize_t numRead;
  char *p;
  struct inotify_event *event;
  if (argc < 2 || strcmp(argv[1], "--help") == 0)
  usageErr("%s pathname... \n", argv[0]);

  inotifyFd = inotify_init();
  if (inotifyFd == -1)
    exit(EXIT_FAILURE);

  /* Create inotify instance */
  for (j = 1; j < argc; j++) {
    wd = inotify_add_watch(inotifyFd, argv[j], IN_ALL_EVENTS);
    if (wd == -1){
      exit(EXIT_FAILURE);
    }

    printf("Watching %s using wd %d\n", argv[j], wd);
  }

  for (;;) {
  /* Read events forever */
  numRead = read(inotifyFd, buf, BUF_LEN);
  if (numRead == 0)
  fwrite(stderr, "read() from inotify fd returned 0!");
  if (numRead == -1)
  exit(EXIT_FAILURE);

  printf("Read %ld bytes from inotify fd\n", (long) numRead);
  /* Process all of the events in buffer returned by read() */
  for (p = buf; p < buf + numRead; )
  {
      event = (struct inotify_event *) p;
      displayInotifyEvent(event);
      p += sizeof(struct inotify_event) + event->len;
  }
}
  exit(EXIT_SUCCESS);
}

```
