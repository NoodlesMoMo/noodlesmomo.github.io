---
title: go使用反射实现路径到方法的映射
author: Noodles
layout: post
comments: true
permalink: /2018/11/reflect-pathinfo
categories:
  - golang
tags:
  - golang
---

<!--more-->

 ---------------------------------------------------

  本文介绍如何使用reflect实现简单的路径到方法的映射方法。

  这里使用的框架是fasthttp。fasthttp并没有提供灵活的路由方法。
  这里借助[fasthttp-routing]("github.com/qiangxue/fasthttp-routing")。


  {% highlight go %}

    // router.go 中增加 ProjectHandler
  var (
	configService = service.NewConfigService()
    )

    // configService底层实际是个map,所以可以使用`[]`运算符。
    func ConfigureHandler(ctx *routing.Context) error {
        action := string(ctx.Param("action"))
        if handler, ok := (*configService)[action]; ok {
            return handler(ctx)
        }

        return service.NotFound(ctx)
    }

    authRouter := router.Group("")
	authRouter.Use(AuthMiddleHandler)

	authRouter.Any("/config/<action>/<res>", ConfigureHandler)

  {% endhighlight %}

  在 config 的 handler中，实现如下:

  {% highlight go %}

  type ConfigService map[string]func(ctx *routing.Context) error

  // 这里将驼峰式的明明方法转换成小写并且在期间添加`_`
  func (c *ConfigService) transFuncName(origin string) string {
	xname := make([]byte, 0)

	for idx, x := range origin {
		if byte(x) >= 'A' && byte(x) <= 'Z' && idx != 0 {
			xname = append(xname, '_')
		}
		xname = append(xname, byte(x))
	}

	return string(xname)
}

// 利用反射将可导出的method存到底层的map中。
func (c *ConfigService) initialization() {
	tye := reflect.TypeOf(c)
	for i := 0; i < tye.NumMethod(); i++ {
		method := tye.Method(i)

		fn, ok := method.Func.Interface().(func(*ConfigService, *routing.Context) error)
		if !ok {
			continue
		}

		(*c)[strings.ToLower(c.transFuncName(method.Name))] = func(ctx *routing.Context) error {
			return fn(c, ctx)
		}
	}
}

func NewConfigService() *ConfigService {
	inst := &ConfigService{}
	inst.initialization()

	return inst
}

// 如果想要通过 `/config/page/1` 这种方式请求到page方法，则接下来就很简单了

func (c *ConfigService) Hello(ctx *routing.Context) error {

    id := ctx.Param("res")

    ctx.WriteString(fmt.Sprintf("page %s", id))
    
    return nil
}

  {% endhighlight %}
