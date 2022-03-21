---
title: HCTF 2018 WarmUP writeup
date: 2022-03-7 22:47:00
tags: [ctf,security]
---
# HCTF 2018 WarmUp writeup

我在adworld上发现有这个题目的环境，然后觉得还挺有意思的于是仔细做了做，顺带写一篇博客记录一下踩坑过程。

题目是php的代码审计

考察的是一个文件包含漏洞(phpmyadmin 4.8.1任意文件包含)

注释中可以看到source.php，访问后简单审计一下代码，还发现了hint.php，除此之外的文件都是无法访问的，自然我们也不能访问hint.php中告诉我们的flag文件来直接获取，而是要找到一种方式间接的访问到flag文件。

![image-20220307223132800](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220307223132800.png)

![image-20220307223148815](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220307223148815.png)

```php
 if (! empty($_REQUEST['file'])
        && is_string($_REQUEST['file'])
        && emmm::checkFile($_REQUEST['file'])
    ) {
        include $_REQUEST['file'];
        exit;
    } else {
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
    }  
```

主入口就是一个if函数，里面要求传入的参数file不为空且是string，并且可以通过checkFile函数校验，所以我们需要仔细查看checkFile，漏洞肯定出现在这里。

我们来看看checkFile函数的主体

```php
$whitelist = ["source"=>"source.php","hint"=>"hint.php"];
            if (! isset($page) || !is_string($page)) {
                echo "you can't see it";
                return false;
            }

            if (in_array($page, $whitelist)) {
                return true;
            }

            $_page = mb_substr(
                $page,
                0,
                mb_strpos($page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }
			//下面这段在做啥呢？？？？？？？？
            $_page = urldecode($page);
            $_page = mb_substr(
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }
            echo "you can't see it";
            return false;
```

定义了一个白名单，如果file不存在或者不是string，那么就 you can‘t see it啦

如果file在白名单中，那么就返回true，到这里逻辑上看起来还挺正常的没啥问题。

但是我们发现下面有这么一段和上面好像做了一样的事情，那这段做了什么呢？

下面这段多了一句`$_page = urldecode($page);`其他部分相同，是为了防止用户传参时符号被url encode了，导致无法解析对应文件。

但是这里解析url访问对应page的过程，恰好是我们访问到这个已知名字的flag文件的漏洞产生的地方，这里截取访问url中第一个问号处后面的字符串的这么一个文件（如果没有就加一个’?'），看在不在白名单中，我们假如说传参file=source.php?，那这个时候他其实解析两次，前面解析出来source.php，后面这段出来是个空，也就是相当于可以自由输入来访问一个路径，然后就是因为我们在访问source.php，这个时候返回了source.php的源代码，但是只返回了一份，传参进去的内容其实是没有返回的，我们如果传参file=source.php，本身我们访问到文件有一份源代码，代码里面又return true，然后又能看到一份，会是两份，这是差异产生的地方，因此我们可以借助file=source.php?/xxxxxxxxxxxxxxx/

然后就去想办法访问到flag文件，但是我们肯定不在根目录，需要加很多/../

```
source.php?file=source.php?/../../../../../../../../../ffffllllaaaagggg
```

这样我们就能访问到flag文件啦

![image-20220307224544354](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220307224544354.png)