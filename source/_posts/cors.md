---
title: 浅谈开发中的cors限制
date: 2021-11-19 12:19:00
tags: front-end
cover: https://miro.medium.com/max/1200/1*fFOeYqrSmQc5WCVqBazQKw.jpeg
top_img: false
---

# 浅谈开发中的cors限制

## 前言

最近有用vue然后调face++的api做一个前端人脸识别的需求，其中使用了axios作为http请求库，配置浏览器cors限制时遇到了一些不太一样的问题，写篇博客记录一下。

## 什么是cors

`跨源资源共享` ([CORS](https://developer.mozilla.org/zh-CN/docs/Glossary/CORS))（或通俗地译为跨域资源共享）是一种基于 [HTTP](https://developer.mozilla.org/zh-CN/docs/Glossary/HTTP) 头的机制，该机制通过允许服务器标示除了它自己以外的其它[origin](https://developer.mozilla.org/zh-CN/docs/Glossary/Origin)（域，协议和端口），这样浏览器可以访问加载这些资源。跨源资源共享还通过一种机制来检查服务器是否会允许要发送的真实请求，该机制通过浏览器发起一个到服务器托管的跨源资源的"预检"请求。在预检中，浏览器发送的头中标示有HTTP方法和真实请求中会用到的头。

![cors_principle](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/cors_principle.png)

如果服务器不同源，那么浏览器就会存在cors限制，这样的话我就没法从localhost:8080请求到face++ api的这个服务器了，所以我们需要一些措施去解决cors限制。

## 明明是post，为什么会发送option请求

我在开发过程中不只是遇到了cors限制的问题，我也同样很奇怪，为什么会先发送一个option请求，option请求是什么，我明明是发送的post请求。那这个的话其实是因为浏览器将CORS请求分为两类：简单请求（simple request）和非简单请求（not-simple-request）,简单请求浏览器不会预检，而非简单请求会预检。这两种方式怎么区分？

1.请求方式只能是：GET、POST、HEAD

2.HTTP请求头限制这几种字段：Accept、Accept-Language、Content-Language、Content-Type、Last-Event-ID

3.Content-type只能取：application/x-www-form-urlencoded、multipart/form-data、text/plain

由于我发送请求时，Content-type是json格式，这样的话就会触发非简单请求。

**非简单请求是对那种对服务器有特殊要求的请求，比如请求方式是PUT或者DELETE，或者Content-Type字段类型是application/json。都会在正式通信之前，增加一次HTTP请求，称之为预检。浏览器会先询问服务器，当前网页所在域名是否在服务器的许可名单之中，服务器允许之后，浏览器会发出正式的XMLHttpRequest请求，否则会报错。**

那这样看来，face++的服务器咱们又不能修改，所以我们肯定是需要添加Content-Type字段，同时仔细看face++的文档中我们也可以发现，确实是需要Content-Type = multipart/form-data的设置，我们给axios添加上这个请求头后，就会变成POST请求啦，但是我们发现POST请求还是被拦截了。

## 修改成简单http请求后为什么还是被cors拦截

那我们现在发起的是一个简单请求。

**对于简单请求，浏览器直接请求，会在请求头信息中，增加一个origin字段，来说明本次请求来自哪个源（协议+域名+端口）。服务器根据这个值，来决定是否同意该请求，服务器返回的响应会多几个头信息字段。**

这个时候face++的api接口仍然没有同意此次http请求，那么是因为他服务器并没有许可localhost:8080这个客户端的访问，这时候我们需要给vue配置proxy，也就是代理请求，首先localhost:8080会将请求发给代理服务器，然后代理服务器是可以获取接口返回的信息的，这时候就可以解决跨域了，下面我们来说说为什么配置代理可以解决跨域问题。

![image-20220304120613627](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220304120613627.png)

并不是网页服务访问代理，而是代理检测网页服务内部的接口服务，当符合条件的服务出现的时候，代理拦截住，并替代网页服务返回结果。

下面是给vue.config.js添加devServer的具体代码

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

## 解决了本地开发环境问题后，以后上线生产环境，又如何解决代理问题呢？

在本机调试下，webpack帮你把脏活累活干了，那么打包之后，服务器上没有webpack了，代理怎么办呢？

这时候可以使用ngix，配置一下server就可以啦

confnginx.conf是ngix的配置文件，在其中location /api 就是ngnix的代理。意思与测试环境的意思相同。

至此我们就能成功解决开发和生产环境下的cors问题了。

