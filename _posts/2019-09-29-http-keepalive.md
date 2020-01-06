---
title: HTTP keep-alive
author: Noodles
layout: post
comments: true
permalink: /2019/09/http-keepalive
categories:
  - http
tags:
  - keep-alive
---

<!--more-->

 ---------------------------------------------------

  HTTP的keep alive相关的概念比较有迷惑性，本文通过实际例子，一步一步探索整理下相关的知识。


### HTTP connection首部

  HTTP从客户端发出请求，到接收到服务发来的响应，在这过程中会经过一连串的HTTP中间实体（代理，高速缓存）等。
  相应的，HTTP首部字段将定义成缓存代理和非缓存代理的行为，分成两种类型:

  **端到端首部（end-to-end Header）**: 首部会转发给请求/响应对应的最终接收目标，且保存在由缓存生成的响应中，另外规定它被发。

  **逐跳首部（hop-by-hop header）**: 首部只对单次转发有效，会因通过缓存或者代理而不再转发。

  HTTP/1.1和之后的版本中，如果要使用`hop-by-hop`首部，需要提供`Connection`首部字段。

  除以下8个首部字段外，其它所有字段都属于端到端首部:

  **hop-by-hop header**
  ```bash
  Connection
  Keep-Alive
  Proxy-Authenticate
  Proxy-Authorization
  Trailer
  TE
  Transfer-Encoding
  Upgrade
  ```

  Connection 头（header） 决定当前的事务完成后，是否会关闭网络连接。如果该值是“keep-alive”，网络连接就是持久的，不会关闭，使得对同一个服务器的请求可以继续在该连接上完成。

  除去标准的逐段传输（hop-by-hop）头（Keep-Alive, Transfer-Encoding, TE, Connection, Trailer, Upgrade, Proxy-Authorization and Proxy-Authenticate），任何逐段传输头都需要在 Connection 头中列出，这样才能让第一个代理知道必须处理它们且不转发这些头。标准的逐段传输头也可以列出（常见的例子是 Keep-Alive，但这不是必须的）。


  **语法**

  ```bash
  Connection: keep-alive
  Connection: close
  ```

  **指令**
  
  `close`

  表明客户端或服务器想要关闭该网络连接，这是HTTP/1.0请求的默认值


  `以逗号分隔的HTTP头 [通常仅有 keep-alive]`

  表明客户端想要保持该网络连接打开，HTTP/1.1的请求默认使用一个持久连接。
  这个请求头列表由头部名组成，这些头将被第一个非透明的代理或者代理间的缓存所移除：这些头定义了发出者和第一个实体之间的连接，而不是和目的地节点间的连接。


### Nginx keep alive
---------------------

  首先梳理下nginx中关于HTTP keep alive有关配置。  


#### HTTP core module
---------------------

  `keepalive_disable none | browser ...`

  > Disables keep-alive connections with misbehaving browsers. The browser parameters specify which browsers will be affected. The value msie6 disables keep-alive connections with old versions of MSIE, once a POST request is received. The value safari disables keep-alive connections with Safari and Safari-like browsers on macOS and macOS-like operating systems. The value none enables keep-alive connections with all browsers.

  > 这个选项用来禁用某些浏览器keepalive请求。实际应用的场景不多见。默认对早期的IE6版本关闭keep-alive。

----------------------

  `keepalive_requests number`

  > Sets the maximum number of requests that can be served through one keep-alive connection. After the maximum number of requests are made, the connection is closed.

  > 设置某个开启keepalive的连接上的http请求数。默认值是100。nginx会统计此开启keepalive连接上的请求数，当达到这个值时，nginx关闭这个连接。

----------------------

  `keepalive_timeout timeout [header_timeout]`

  > The first parameter sets a timeout during which a keep-alive client connection will stay open on the server side. The zero value disables keep-alive client connections. The optional second parameter sets a value in the “Keep-Alive: timeout=time” response header field. Two parameters may differ.  
  The “Keep-Alive: timeout=time” header field is recognized by Mozilla and Konqueror. MSIE closes keep-alive connections by itself in about 60 seconds.

  > 第一个参数指示nginx对连接保持多长时间，为0则禁用client的keep-alive连接。可选参数是设置响应头部"Keep-Alive: timeout=time"

接下来我们做下实验, nginx 配置如下:

```bash
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    tcp_nopush    on;
    keepalive_timeout  65; # nginx为连接保持65s
    keepalive_requests 7;  # 每个连接只能发7次http请求
    server {
        listen       9090;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

为了方便验证，我们可以借助`python requests`库看下实际效果. 完整脚本如下:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import logging
import requests
import time
from httplib import HTTPConnection

# HTTPConnection.debuglevel = 1

logging.basicConfig(level=logging.DEBUG)
requests_log = logging.getLogger("requests")
requests_log.setLevel(logging.DEBUG)
requests_log.propagate = True

logging.basicConfig(lelel=logging.DEBUG)
session = requests.Session()

while True:
    resp = session.get('http://10.143.47.227:9090/index.html')
    # print resp
    time.sleep(0.3)

```

输出结果(从结果看出确实是每7个请求重新建立连接):

```bash
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:Resetting dropped connection: 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:Resetting dropped connection: 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:Resetting dropped connection: 10.143.47.227
```
以下是抓包结果:

  ![keepalive1](/images/2019/0930/keepalive.png)

  **⚠️注意: 此处并不是nginx主动关闭，而是由客户端发起的FIN包。**

  <center><img src="/images/2019/0930/keepalive2.png" width="400"/></center>

  nginx在第7个包的响应中设置了`Connection: close`。


接下来，我们把nginx配置http块中的`keepalive_timeout`改成:

```bash
http {
    ...
    keepalive_timeout 5s 5s;
    ...
}
```

输出结果（将python脚本中的`HTTPConnection.debuglevel = 1`一行注释去掉)，可以看到，响应头部增加了`Keep-Alive: timeout=5`

```bash
reply: 'HTTP/1.1 200 OK\r\n'
header: Server: nginx/1.17.4
header: Date: Mon, 30 Sep 2019 05:47:03 GMT
header: Content-Type: text/html
header: Content-Length: 612
header: Last-Modified: Tue, 24 Sep 2019 15:08:48 GMT
header: Connection: keep-alive
header: Keep-Alive: timeout=5
header: ETag: "5d8a3180-264"
header: Accept-Ranges: bytes
```
**⚠️**response中设置`keep-alive`，需要客户端的支持。

接下来，把nginx `keepalive_timeout`设置一个较短的时间，脚本请求间隔设置较大时间，来看下结果:

```bash
http {
    ...
    keepalive_timeout  1s;
    keepalive_requests 100;
    ...
}
```
结果输出(python 请求间隔为3秒):

```bash
DEBUG:urllib3.connectionpool:Starting new HTTP connection (1): 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:Resetting dropped connection: 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:Resetting dropped connection: 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:Resetting dropped connection: 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
DEBUG:urllib3.connectionpool:Resetting dropped connection: 10.143.47.227
DEBUG:urllib3.connectionpool:http://10.143.47.227:9090 "GET /index.html HTTP/1.1" 200 612
```

可以看到每次都需要重建连接。但这种情况与到达`keepalive_requests`预设值不同，前者主动断开一方是客户端，
而这次是服务端。下面是抓包情况：

<center><img src="/images/2019/0930/keepalive3.png"/></center>

<center><img src="/images/2019/0930/keepalive4.png"/></center>

总结一下:

  1. `keepalive_disable`这个参数目前实际当中不多见，保持默认即可。
  2. `keepalive_requests`这个值根据实际情况，如果nginx下游是用户，则这个值默认值通常来说够用了。但如果面向的是为其它server提供服务，这个值太过保守了，应该设置更大更大些。
  3. `keepalive_timeout`这个值通常保持默认即可，不应设置过小，如果服务QPS比较高，过小会导致nginx产生大量的`TIME_WAIT`，效率不高。


### 常被误解的Connection首部

  这部分内容来自《http权威指南》。

#### http/1.0 + keep-alive

  `keep-alive`已经不再使用了，而且在当前的`HTTP/1.1`规范中也没有对它的说明了。但浏览器和服务器对`keep-alive`握手支持仍然相当广泛。这里看下`HTTP/1.0`时代是如何处理的:

  实现`HTTP/1.0 Keep-alive`连接的客户端可以通过包含`Connection: Keep-Alive`首部请求将一条连接保持在打开状态。

  如果服务器愿意为下一条请求维持这个连接，就在响应中包含相同的首部`Connection: Keep-Alive`。如果没有，则客户端认为服务器不支持Keep-Alive，会在发回ACK后关闭连接。



  
