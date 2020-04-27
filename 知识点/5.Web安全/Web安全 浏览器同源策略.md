[TOC]


# Web安全(一) --- 浏览器同源策略


## #1 什么是浏览器同源策略

**浏览器的同源策略一直是开发中经常遇到的问题,它是浏览器最核心也是最基本的安全功能,如果缺少了同源策略,则浏览器的正常功能都会受到影响**

### #1.1 什么是同源 ? 

同源是指同 ` 协议 ` 、`同域名`、`同端口`。



![](https://raw.githubusercontent.com/Coxhuang/yosoro/master/20200322164321.png)

> 注：IE 未将`端口号`加入到同源策略的组成部分之中


在浏览器中, `<script>` 、`<img>`、`<iframe>`、`<link>`等标签都可以跨域加载,而不受浏览器的同源策略的限制, 这些带`src`属性的标签每次加载的时候,实际上都是浏览器发起一次GET请求, 不同于普通请求(XMLHTTPRequest)的是,通过src属性加载的资源,浏览器限制了JavaScript的权限,使其不能读写src加载返回的内容

浏览器同源策略中,除了上述的几个标签可以跨域加载外,其他出现跨域请求时,请求会发到跨域的服务器,并且会服务器会返回数据,只不过浏览器"拒收"返回的数据

### #1.2 同源策略的限制

浏览器的同源策略目的是为了保护用户的信息安全,为了防止恶意网站窃取用户在浏览器上的数据,如果`不是同源`的站点,将不能进行如下操作 : 

- Cookie、LocalStorage 和 IndexDB 无法读写 
- DOM 和 Js对象无法获得
- AJAX请求不能发送


#### #1.2.1 不能读写Cookie、Session Storage、Local Storage、Cache、Indexed DB

用户登录某个站点,站点后端服务器验证账号密码正确之后,会返回Cookie、Token 或者是用户名和密码给客户端浏览器,浏览器会将这些个人数据保存到Cookie、Session Storage、Local Storage、Cache、Indexed DB其中的一个(具体怎么保存,取决网站开发人员),如果浏览器没有同源策略,当用户访问恶意网站时,恶意网站就可以通过脚本获取用户的数据,这是极其不安全的行为

所以在不是同源的情况下,不能读写其他站点设置的Cookie、Session Storage、Local Storage、Cache、Indexed DB


#### #1.2.2 DOM

来自一个源的js只能读写自己源的DOM树不能读取其他源的DOM树

#### #1.2.3 异步请求 

一般而言来自一个源的js只能向自己源的接口发送请求,不能向其他源的接口发送请求。当然其实本质是，一方面浏览器发现一个源的js向其他源的接口发送请求时会自动带上Origin头标识来自的源，让服务器能通过Origin判断要不要向应；另一方面，浏览器在接收到响应后如果没有发现Access-Control-Allow-Origin允许发送请求的域进行请求那也不允许解析


## #2 跨域

不同域之间的访问就叫跨域,因为浏览器同源策略的限制,导致我们在不同源之间通信,出现了浏览器接受不到服务端返回数据的问题,这也是目前前后端分离的项目必须要解决的问题

### #2.1 解决跨域的方法  

- 通过jsonp跨域
- document.domain + iframe跨域
- location.hash + iframe
- window.name + iframe跨域
- postMessage跨域
- 跨域资源共享(CORS)
- Nginx反向代理
- nodejs中间件代理跨域
- WebSocket协议跨域


**下面主要讲两个平时我常用的解决跨域的方法 CORS 和 Nginx反向代理** 


### #2.2 跨域资源共享(CORS)

> 只服务端设置Access-Control-Allow-Origin即可，前端无须设置，若要带cookie请求：前后端都需要设置。


- 服务端需要设置以下响应头


```python 
Access-Control-Allow-Origin:http://www.admin.minhung.me // 一般用法（*，指定域，动态设置），*不允许携带认证头和cookies 
Access-Control-Allow-Credentials: true  // 是否允许后续请求携带认证信息(如:cookies),该值只能是true,否则不返回 
Access-Control-Max-Age: 1800 //预检结果缓存时间,也就是上面说到的缓存，单位：秒 
Access-Control-Allow-Methods:GET,POST,PUT,POST // 允许的请求动词, GET|POST|PUT|DELETE
Access-Control-Allow-Headers:x-requested-with,content-type //允许的请求头字段
```

#### # CORS方法如何携带Cookie 

> 如果使用CORS解决跨域问题,除了后端服务器需要配置以上信息外,前端也需要进行如下配置 : 

```
// 表示跨域请求时是否需要使用凭证
axios.defaults.withCredentials = true // Vue.js框架
```

==并且,后端服务器不能配置Access-Control-Allow-Origin: *,一定要记住,如果配置为任意,不管withCredentials有没有设置，cookie也带不过去==

### #2.3 Nginx反向代理 

通过nginx配置一个代理服务器（域名与端口号和客户端不同）做跳板机，反向代理访问api.minhung.me接口，并且可以顺便修改cookie中admin.minhung.me信息，方便当前域cookie写入，实现跨域登录。

前端项目(admin.minhung.me:19700)

后端项目(api.minhung.me:19800)

在前端项目nginx配置中添加proxy_pass,将请求的接口代理到 api.minhung.me:19800

```
# 前端nginx配置 
server {
    listen       19700;
    server_name  admin.minhung.me;

    location / {
        proxy_pass   http://api.minhung.me:19800;  #反向代理
        proxy_cookie_domain api.minhung.me:19800 admin.minhung.me:19700; #修改cookie里域名
        index  index.html index.htm;
    }
}
```








