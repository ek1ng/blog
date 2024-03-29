---
title: 浅谈cors
date: 2021-11-19 12:19:00
updated: 2022-05-12 20:14:00
tags: [front-end]
description: 聊一聊开发中的cors限制
category: Develop
---

最近有用 vue 然后调 face++的 api 做一个前端人脸识别的需求，其中使用了 axios 作为 http 请求库，配置浏览器 cors 限制时遇到了一些不太一样的问题，写篇博客记录一下。

## 什么是 cors

`跨源资源共享` ([CORS](https://developer.mozilla.org/zh-CN/docs/Glossary/CORS))（或通俗地译为跨域资源共享）是一种基于 [HTTP](https://developer.mozilla.org/zh-CN/docs/Glossary/HTTP) 头的机制，该机制通过允许服务器标示除了它自己以外的其它[origin](https://developer.mozilla.org/zh-CN/docs/Glossary/Origin)（域，协议和端口），这样浏览器可以访问加载这些资源。跨源资源共享还通过一种机制来检查服务器是否会允许要发送的真实请求，该机制通过浏览器发起一个到服务器托管的跨源资源的"预检"请求。在预检中，浏览器发送的头中标示有 HTTP 方法和真实请求中会用到的头。

![cors_principle](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/cors_principle.png)

如果服务器不同源，那么浏览器就会存在 cors 限制，这样的话我就没法从 localhost:8080 请求到 face++ api 的这个服务器了，所以我们需要一些措施去解决 cors 限制。

### CORS 的意义是什么？难道是为了给开发者增加麻烦？

主要是为了防 CSRF，有了 cors 之后，假设用户不小心点击了恶意站点，也无法从 B 向站点 A 发送请求，因为站点 A 不会配置对站点 B 的跨域，因此从 B 站点发起一个向 A 站点的请求是不被浏览器允许的，浏览器会检测到 A 站点接口的响应头中没有配置对 B 站点的跨域，从而拦截响应。

![图 1](https://s2.loli.net/2022/06/14/2VAzY3XZf6FlJ49.png)  

## 简单请求和非简单请求

### 简单请求和非简单请求是什么？

我在开发过程中不只是遇到了 cors 限制的问题，我也同样很奇怪，为什么会先发送一个 option 请求，option 请求是什么，我明明是发送的 post 请求。那这个的话其实是因为浏览器将 CORS 请求分为两类：简单请求（simple request）和非简单请求（not-simple-request）,简单请求浏览器不会预检，而非简单请求会预检。这两种方式怎么区分？

1.请求方式只能是：GET、POST、HEAD

2.HTTP 请求头限制这几种字段：Accept、Accept-Language、Content-Language、Content-Type、Last-Event-ID

3.Content-type 只能取：application/x-www-form-urlencoded、multipart/form-data、text/plain

由于我发送请求时，Content-type 是 json 格式，这样的话就会触发非简单请求。

**非简单请求是对那种对服务器有特殊要求的请求，比如请求方式是 PUT 或者 DELETE，或者 Content-Type 字段类型是 application/json。都会在正式通信之前，增加一次 HTTP 请求，称之为预检。浏览器会先询问服务器，当前网页所在域名是否在服务器的许可名单之中，服务器允许之后，浏览器会发出正式的 XMLHttpRequest 请求，否则会报错。**

那这样看来，face++的 api 端口设计咱们又不能修改，所以我们肯定是需要添加 Content-Type 字段，同时仔细看 face++的文档中我们也可以发现，确实是需要 Content-Type = multipart/form-data 的设置，我们给 axios 添加上这个请求头后，就会变成 POST 请求啦，但是我们发现 POST 请求还是被拦截了，因为不论是简单请求还是非简单请求，都是收到 cors 限制的。

### 对非简单请求做预检的意义是什么？

简单来说应该是节约资源，非简单请求就是普通 HTML Form 无法实现的请求。比如 PUT 方法、需要其他的内容编码方式、自定义头之类的。

对于服务器来说，第一，许多服务器压根没打算给跨源用。当然你不给 CORS 响应头，浏览器也不会使用响应结果，但是请求本身可能已经造成了后果。所以最好是默认禁止跨源请求。

第二，要回答某个请求是否接受跨源，可能涉及额外的计算逻辑。这个逻辑可能很简单，比如一律放行。也可能比较复杂，结果可能取决于哪个资源哪种操作来自哪个 origin。对浏览器来说，就是某个资源是否允许跨源这么简单；对服务器来说，计算成本却可大可小。所以我们希望最好不用每次请求都让服务器劳神计算。

CORS-preflight 就是这样一种机制，浏览器先单独请求一次，询问服务器某个资源是否可以跨源，如果不允许的话就不发实际的请求。注意先许可再请求等于默认禁止了跨源请求。如果允许的话，浏览器会记住，然后发实际请求，且之后每次就都直接请求而不用再询问服务器否可以跨源了。于是，服务器想支持跨源，就只要针对 preflight 进行跨源许可计算。本身真正的响应代码则完全不管这个事情。并且因为 preflight 是许可式的，也就是说如果服务器不打算接受跨源，什么事情都不用做。

但是这机制只能限于非简单请求。在处理简单请求的时候，如果服务器不打算接受跨源请求，不能依赖 CORS-preflight 机制。因为不通过 CORS，普通表单也能发起简单请求，所以默认禁止跨源是做不到的。

既然如此，简单请求发 preflight 就没有意义了，就算发了服务器也省不了后续每次的计算，反而在一开始多了一次 preflight。

## 错误配置跨域的结果

经典的错误配置`Access-Control-Allow-Origin = *`。

首先，跨域本身是一种安全措施，这种错误的跨域配置相当于跨域防 CSRF 防了个寂寞。

其次，chromium 内核也对后端配置跨域错误时做出了很严格的限制，这也会导致你在开发时遇到诸多困难，比如后端的鉴权接口通过 set-cookie 响应头返回了 session，你想从请求头里面拿 session 但是你拿不到，然后一查发现如果想要从 set-cookie 里面拿 session 那就需要配置 Access-Control-Allow-Credentials 这个响应头，那么比如说你写前端你用 axios，有 with credential 这个属性可以开启，但是开启后由于后端错误配置跨域，你的请求会在 with credential 开启后被跨域拦截，原因是 chromium 发现后端错误配置了跨域，总之，错误配置跨域的本质问题是无法防御 CSRF 攻击，因此浏览器在请求错误配置跨域的接口后对响应头的字段做检查，并且拦截响应，也会导致开发上也很难继续工作。

## CORS 的解决方案

cors 的解决方案本质上都是通过代理服务器来解决的

### 正确配置 CORS 请求头

后端接口正确配置 cors 的请求头即可，但是我这里是调用的 api，所以说我得想办法在前端上解决这个问题。

### webpack 的 devServer

那我们现在发起的是一个简单请求。

**对于简单请求，浏览器直接请求，会在请求头信息中，增加一个 origin 字段，来说明本次请求来自哪个源（协议+域名+端口）。服务器根据这个值，来决定是否同意该请求，服务器返回的响应会多几个头信息字段。**

这个时候 face++的 api 接口仍然没有同意此次 http 请求，那么是因为他服务器并没有许可 localhost:8080 这个客户端的访问，这时候我们需要给 vue 配置 proxy，也就是代理请求，首先 localhost:8080 会将请求发给代理服务器，然后代理服务器是可以获取接口返回的信息的，这时候就可以解决跨域了，下面我们来说说为什么配置代理可以解决跨域问题。

![image-20220304120613627](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220304120613627.png)

并不是网页服务访问代理，而是代理检测网页服务内部的接口服务，当符合条件的服务出现的时候，代理服务器拦截请求并且以代理服务器的身份请求网页后端服务，服务端之间的请求不受跨域限制，因为跨域是浏览器的一种安全策略，那么这个时候代理服务器将返回的接口返回给客户端，客户端就不会收到 cors 的限制啦。

下面是给 vue.config.js 添加 devServer 的具体代码

```js
devServer: {
    // development server port 8000
    port: 8000,
    proxy: {
        '/facepp/v3': {
            target: 'https://api-cn.faceplusplus.com',     // 拦截到'/facepp/v3'的，将axios中baseURL替换成target
            ws: true,                                 // proxy websockets
            // logLevel: 'debug',
            changeOrigin: true,                       // 是否跨域
        },
    }
  },
```

### nginx 反向代理

在本机调试下，webpack 帮你把脏活累活干了，那么打包之后，服务器上没有 webpack 了，代理怎么办呢？

这时候可以使用 nginx，配置一下 server 就可以啦

config.conf 是 nginx 的配置文件，在其中 location /api 就是 nginx 的代理。意思与测试环境的意思相同，我们就能成功解决开发和生产环境下的 cors 问题了。
