+++
date = '2025-08-21T18:46:47+08:00'
title = '兴趣八股之计算机网络EP03——CORS浅探'
categories = ["计算机网络"]
tags = ["CORS","八股"]
+++

## 引子

在之前的部署私有镜像仓库项目中，我们遇到过一次跨域请求问题，在 nginx 中添加了有关 CORS 的限制内容，那么这篇文章就来填个坑，梳理一下跨域的原理。

从下面这段简单的跨域中间件代码开始吧

```go
// CORSMiddleware 跨域中间件
func CORSMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
		c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
		c.Writer.Header().Set("Access-Control-Allow-Headers", "Origin, Content-Type, Content-Length, Accept-Encoding, X-CSRF-Token, Authorization")
		
		if c.Request.Method == "OPTIONS" {
			c.AbortWithStatus(204)
			return
		}
		
		c.Next()
	}
}
```

## 为什么会有跨域限制

CORS 全称是跨域资源共享，它是一个设定好的**标准**，起源于浏览器实现的一项安全策略——同源策略。

- 两个 URL 的协议、域名、端口必须完全相同
- 例如 `http://example.com:8080`和`http://example.com:8080/users`是同源的

那么他限制了什么呢？**不同源的脚本不能直接读取或者操作另一个源的 DOM 或者网络请求。**

目的呢？防止 **CSRF（跨站请求伪造）和 XSS（跨站脚本攻击）**等。

### CSRF

拿一个 AI 的举例来看：

假设登录了银行网站并且登录状态存在了 cookie 中，此时如果浏览另一个恶意网站，这个网站页面包含了一行访问银行网站并取钱的请求。

浏览器就会发送这个请求带上以之前的 cookie 内容，这样银行就会把钱转给攻击者。

所以浏览器就会保证同源访问策略，有了同源访问策略，恶意网站就拿不到银行网站的 cookie 信息了，自然也无法取到钱。

实际上还使用到了 X-CSRF-Token 来防护 CSRF，这个不是今天的重点。

### 问题

这是一个很好地策略，为什么我们还需要来进行跨域呢？因为在日常开发中，前后端分离环境下，前端和后端开发出的内容并不是同源的，所以需要跨域；

又或者在微服务架构中，服务 A 调用服务 B 的 API，二者不同源的话，需要在 B 的网关处配置 CORS 策略，允许服务 A 的域名。

而既然要跨域，那必须得保证安全，所以 CORS 就**允许服务器主动声明哪些外部源可以访问它的资源**，这样既能够保证安全又能够放宽同源策略。

主要是通过 **HTTP 响应头**来告知。

### 分类

具体来说，服务器通过设置一系列`Access-Control-*`开头的响应头，告知浏览器，“我允许这个源，用这种方法，带这些头信息来访问我”，否则拦截响应。

回到上面的代码中，我们一行一行来看

`c.Writer.Header().Set("Access-Control-Allow-Origin", "*")` 设置允许访问的源为全部，这里是开发环境下，生产环境更加严格。

`c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, OPTIONS")`设置允许的请求方法。

`c.Writer.Header().Set("Access-Control-Allow-Headers", "Origin, Content-Type, ... Authorization")`允许浏览器的实际请求允许带哪些自定义的请求头。

```go
if c.Request.Method == "OPTIONS" {
    c.AbortWithStatus(204)
    return
}
```

处理预检请求。

这里我们就要对 CORS 请求进行分类：**简单请求+预检请求**。

- 简单请求：例如一个简单的 GET，POST,且 `Content-Type` 为 `text/plain、multipart/form-data、application/x-www-form-urlencoded`
- 预检请求(OPTIONS)：PUT，DELETE，PATCH，`Content-Type`为`application/json`，带有自定义的请求头等。

### 本中间件的预检流程

浏览器向服务器发送一个OPTIONS请求。这个请求会带上两个特殊的头：

- Access-Control-Request-Method: 实际请求将要使用的方法 (例如: POST)。
- Access-Control-Request-Headers: 实际请求将要携带的自定义头 (例如: Authorization, Content-Type)

服务器捕获预检请求：

- 服务器在响应中设置好上面提到的 Allow-Methods 和 Allow-Headers 等许可头
- `c.AbortWithStatus(204)`:服务器返回一个 `204 No Content` 的状态码，表示“许可信息已在头部，没有正文内容”。Abort会中断后续的中间件和处理函数，因为这只是一个许可问询，不需要执行真正的业务逻辑

浏览器 (决策)：收到204响应后，检查响应头里的许可信息。

- 如果当前请求的方法和头部都在许可范围内，浏览器就会发送真正的、实际的业务请求（比如那个POST请求）
- 否则，就在控制台报CORS错误

## 总结

所以说到这里，跨域资源保护很简单，核心就是从服务器的角度出发，让其规定谁可以如何访问它的资源，保证安全的同时做到跨域访问。

最后再回顾一下构建私有镜像仓库时我们用到了命令,其中就指定了具体的源可以访问。

```bash
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v /etc/docker/registry/auth:/auth \
  -v /var/lib/registry:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin='["https://registryui.bfsmlt.top"]' \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Allow-Methods='["GET", "DELETE", "PUT", "POST", "OPTIONS"]' \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Allow-Headers='["Authorization", "Accept", "Cache-Control", "Content-Type"]' \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Allow-Credentials='["true"]' \
  --restart=always \
  registry:2
```

在实际后端开发中，通常会从配置文件或者环境变量中加载`allowedOrigins`列表。

而 CORS 通常也会和一些认证机制 JWT，OAuth2 结合，下一篇文章我打算谈谈 JWT 等认证机制。

**这里是LTX，感谢您阅读这篇博客，人生海海，和自己对话，像只蝴蝶纵横四海。**