[TOC]


# 跨域资源共享(CORS)


## #1 什么是CORS

CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。

它允许浏览器向跨源服务器，发出XMLHttpRequest请求，从而克服了AJAX只能同源使用的限制。

整个CORS通信过程，都是浏览器自动完成，不需要用户参与。对于开发者来说，CORS通信与同源的AJAX通信没有差别，代码完全一样。浏览器一旦发现AJAX请求跨源，就会自动添加一些附加的头信息，有时还会多出一次附加的请求，但用户不会有感觉。

因此，实现CORS通信的关键是服务器。只要服务器实现了CORS接口，就可以跨源通信。

## #2 两种请求

浏览器将CORS请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request）。

1. 简单请求 

只要同时满足以下两大条件，就属于简单请求。

- 请求方法是以下三种方法之一：
    - HEAD
    - GET
    - POST
- HTTP的头信息不超出以下几种字段：
    - Accept
    - Accept-Language
    - Content-Language
    - Last-Event-ID
    - Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain

凡是不同时满足上面两个条件，就属于非简单请求。

2. 复杂请求 

不满足上面简单的,都属于复杂请求


## #3 请求过程

### #3.1 简单请求 

> 当浏览器发现跨域的Ajax请求时简单请求,会走如下流程 : 

- 浏览器 : 兄弟,你这是需要跨域请求吧 ! 得问一下服务器大哥同不同意吧, 我在你的header里加上Origin信息, 让服务器大哥知道你是哪来的
- 服务器 : 来者何人,亮出你的Origin给我瞧瞧(服务器大哥拿着Origin和自己手上的Origin比较),在我的允许范围内,允许进入服务器

![](https://raw.githubusercontent.com/Coxhuang/yosoro/master/20200323031039.png)


> 服务器配置CORS

```
//指定允许其他域名访问
'Access-Control-Allow-Origin:http://admin.minhung.me:19700'//一般用法（*，指定域，动态设置），3是因为*不允许携带认证头和cookies
//是否允许后续请求携带认证信息（cookies）,该值只能是true,否则不返回
'Access-Control-Allow-Credentials:true'
```


> Access-Control-Allow-Origin有多种设置方法 : 

- 设置*是最简单粗暴的，但是服务器出于安全考虑，肯定不会这么干，而且，如果是*的话，游览器将不会发送cookies，即使你的XHR设置了withCredentials
- 动态设置为请求域，多人协作时，多个前端对接一个后端，这样很方便


### #3.2 复杂请求

最常见的情况为,当我们使用`PUT`或者`DELETE`请求时,浏览器会先发送`option`（预检）请求

> 与简单请求不同的是，option请求多了2个字段：

- Access-Control-Request-Method：该字段是必须的，用来列出浏览器的CORS请求会用到哪些HTTP方法。
- Access-Control-Request-Headers：该字段是一个逗号分隔的字符串，指定浏览器CORS请求会额外发送的头信息字段

Access-Control-Request-Method，Access-Control-Request-Headers返回的是满足服务器要求的所有请求方式，请求头，不限于该次请求,服务器收到"预检"请求以后，检查了Origin、Access-Control-Request-Method和Access-Control-Request-Headers字段以后，确认允许跨源请求，就可以做出回应。



```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://api.minhung.me:19800
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```

上面的HTTP回应中，关键的是Access-Control-Allow-Origin字段，表示http://api.minhung.me:19800可以请求数据。该字段也可以设为星号，表示同意任意跨源请求。


如果浏览器否定了"预检"请求，会返回一个正常的HTTP回应，但是没有任何CORS相关的头信息字段。这时，浏览器就会认定，服务器不同意预检请求，因此触发一个错误，被XMLHttpRequest对象的onerror回调函数捕获。控制台会打印出如下的报错信息。

一旦服务器通过了"预检"请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个Origin头信息字段。服务器的回应，也都会有一个Access-Control-Allow-Origin头信息字段。





