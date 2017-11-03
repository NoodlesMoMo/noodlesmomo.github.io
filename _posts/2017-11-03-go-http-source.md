---
title: go http source剖析
author: Noodles
layout: post
permalink: /2017/11/go-http-source
categories:
  - source
tags:
  - go
  - http
  
---

<!--more-->

 ---------------------------------------------------

使用go自带的http包写一个最简单的http服务可以这样:
  {% highlight go %}
  import (
    "net/http"
    )

  func main() {
        http.HandleFunc("/hello",
                func(writer http.ResponseWriter, request *http.Request) {
                                writer.Write([]byte("hello, go http"))
                                        })

            http.ListenAndServe("0.0.0.0:9981", nil)
  }
  {% endhighlight %}

从端口的监听到路由的注册，逐步看go http是如何实现的。


  {% highlight go %}
  // net/http/server.go

  // ListenAndServe always returns a non-nil error.
  func ListenAndServe(addr string, handler Handler) error {
        server := &Server{Addr: addr, Handler: handler}
            return server.ListenAndServe()
  }
  {% endhighlight %}

  {% highlight go %}
  // net/http/server.go
  type Server struct {
    Addr string
    Handler Handler

    /* 省略其他字段，暂不做分析 */
  }

  func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()

    // 用来针对监听时临时性故障时，重新尝试时的sleep时间
    var tempDelay time.Duration

    // 与HTTP/2有关，暂不分析
    if err := srv.setupHTTP2_Serve(); err != nil {
        return err    
    }

    // 将监听对象加入到srv对象中
    srv.trackListener(l, true)
    // 服务结束时，将监听对象从srv对象中移除
    defer srv.trackListener(l, false)

    // 创建空白上下文对象
    baseCtx := context.Background()

    // 将`ServerContextKey` - `srv`和 `LocalAddrContextKey` - `l.Addr()` 键值对加入上下文中
    // `ServerContextKey`和`LocalAddrContextKey`实际分别是`http-server`和`local-addr`字符串
    // 对象的简单包装
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    ctx = context.WithValue(ctx, LocalAddrContextKey, l.Addr())

    for {

        // 开始监听
        rw, e := l.Accept()
        if e != nil {
            select {
                // 如果服务已经关闭，则直接返回错误
                case <- srv.getDoneChan():
                    return ErrServerClosed
                default:
            }    

            // 检查错误
            if ne, ok := e.(net.Error); ok && ne.Temporary() {
                // 如果是临时性的错误，则尝试等待tempDelay时间后重试
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond    
                } else {
                    tempDelay *= 2    
                }    
                
                // 最长不得超过1秒
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max    
                }

                time.Sleep(tempDelay)
                continue
            }
            
            // 如果是非临时性故障，则返回错误
            return e
        }

        tempDelay = 0
        
        // 将连接返回的套接字加入新的连接结构中
        c := srv.newConn(rw)
        // 设置连接状态为`StatNew`
        c.setState(c.rwc, StateNew)

        // 开启go routine，处理返回的连接
        go c.serve(ctx)
    }
  }

  func (c *conn)serve(ctx context.Context){
    // 获取客户端地址
    c.remoteAddr = c.rwc.RemoteAddr().String()
    // 处理连接异常，打印堆栈信息
    defer func(){
        if err := recover(); err != nil && err != ErrAbortHandler {
            const size = 64 << 10
            buf := make([]byte, size)
            buf = buf[:runtime.Stack(buf, false)]
            c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
        }    

        // 如果连接没有被劫持，则结束时关闭连接并设置连接状态
        if !c.hijacked(){
            c.close()
            c.setState(c.rwc, StateClosed)
        }
    }()

    // 有关HTTPS相关处理
    if tlsConn, ok := c.rwc.(*tls.Conn); ok {
        if d := c.server.ReadTimeout; d != 0 {
            c.rwc.SetReadDeadline(time.Now().Add(d))
        }   
        if d := c.server.WriteTimeout; d != 0 {
            c.rwc.SetWriteDeadline(time.Now().Add(d))
        }
        if err := tlsConn.Handshake(); err != nil {
            c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)    
            return
        }
        c.tlsState = new(tls.ConnectionState)
        *c.tlsState = tlsConn.ConnectionState()
        if proto := c.tlsState.NegotiatedProtocol; validNPN(proto) {
            if fn := c.server.TLSNextProto[proto]; fn != nil {
                h := initNPNRequest{ tlsConn, serverHandler{ c.server }}    
                fn(c.server, tlsConn, h)
            }    
            return
        }
    }

    // HTTP/1.x from here on.
    ctx, cancelCtx := context.WithCancel(ctx) // 为ctx加入`done` chan，返回具有取消属性的ctx,和取消的调用对象
    c.cancelCtx = cancelCtx
    defer cancelCtx() // 结束时，调用可取消调用对象，使与之有关的子上下文对象都依次被取消 

    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)
  }

  {% endhighlight %}
