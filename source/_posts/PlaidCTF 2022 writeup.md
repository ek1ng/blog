---
title: PlaidCTF 2022 Amongst Ourselves Shipmate writeup
date: 2022-04-10 14:12:00
updated: 2022-04-10 14:14:00
tags: ctf
---

# PlaidCTF 2022 Amongst Ourselves: Shipmate writeup

一道挺有趣的以amongst游戏为背景出的综合ctf题目，比赛时候web差一点出，以web手的视角重新复盘理解一下题目。

>PPP is proud to announce a completely original, platform-exclusive game for Plaidiverse! With novel socially deceptive mechanics, Amongst Ourselves™ has been recognized with the "Best-In-Class"® and "Best Narrative Game"® awards at the 2022 Plaidies℗℗℗ Game Awards™. Available now at e-shops near you in the Plaidiverse!!
>
>机翻：PPP自豪地宣布一个完全原创的，平台专属的游戏为plaaidiverse !凭借新颖的社交欺骗机制，《between Ourselves™》获得了2022年Plaidies℗℗℗Game awards™的“最佳类”和“最佳叙事游戏”奖。现在可以在你附近的电子商店买到!!

题目一共有5个flag，对应web,misc,pwn,re,crypto5个方向各有一个小题。并且题目给出了源码，可以在本地用docker起环境。

题目就是amongst这个游戏为背景，在生存者视角出了5个小题，内奸视角也有2题但是我们没啥人做也没啥思路。生存者在amongst游戏中需要通过完成小游戏来获得胜利，题目固定了5个游戏并且每个游戏存在一个漏洞，通过漏洞可以拿到flag。

## Provide Credentials

Green starts teleporting wildly around The Shelld before suddenly disappearing. Meanwhile, Brown inspects the access logs on the ship's computer to piece together a timeline of the murders. (Web.)

首先是web题，题目描述就是说到飞船上的电脑，然后这个任务就是给你一个账号密码去输入，正确输入后就可以完成任务。

![image-20220410132818258](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220410132818258.png)

但是这里由于是把传入的语句拼接进了sql语句，那么就可以sql注入。

我们先来看看给出的源码，游戏本身想要做到的查询效果只是，首先他有三个账户，然后他会随机给你一个写在游戏的右边，你输入的是这个他给你的号，他就校验一下首先是不是查询出1行信息，然后这一行信息的id和他想要让你查的id是不是一样，比如说他想让你查zuck这个id是1的用户，但是你查询返回了randall这个id是2的用户这一行信息，那么也是会报错的，在后面构造sql查询的时候是需要注意这一点的，

```tsx
    //过滤部分
    // No sql injection
    username = username.replace("'", "''");
    password = password.replace("'", "''");

  	// Max length of username and password is 32
	username = username.substring(0, 32);
	password = password.substring(0, 32);

    pool.query<{ id: number; username: string }>(
      `SELECT id, username FROM users WHERE username = '${username}' AND password = '${password}'`
    )
      .then((result) => {
        if (this.complete) {
          throw new Error("Invalid state");
        }

        if (result.rowCount !== 1) {
          throw new Error("Invalid credentials");
        }

        if (result.rows[0].id !== this.credentials.id) {
          throw new Error("Invalid credentials");
        }

        this.complete = true;
        this.updatePlayers();
        player.pushUpdate(
          new GameUpdate.ProvideCredentialsResponse({
            success: true,
            message: `Successfully logged in as ${result.rows[0].username}`
          })
        );
      })
      .catch(() => {
        player.pushUpdate(
          new GameUpdate.ProvideCredentialsResponse({
            success: false,
            message: "Invalid credentials"
          })
        );
      });
```

那么sql语句是拼接了username和password的内容，然后sql注入的过滤首先把引号替换成两个引号，然后将长度截取32个字符，来做长度限制，再拼接进sql语句，需要注意的是这里这个pool.query，我们可以看出这个数据库是postgres。

一开始我的想法是，username:admin\，password:or 2>1 limit 1 #（这里password构造一个成真语句其实是不太正确的，即便\可以转义引号，这样的构造sql语句一定会返回users表的第一条信息，但是游戏想让你查询的不一定是第一个用户，那就也是有可能报错的，应该是要使用联合注入的方式去查询flag这个表的flag字段，并且联合注入刚好有两个字段所以第一个字段的id我们让他成为希望我们查询的这个用户的id，第二个字段查flag表的flag就可以了，这里只是我当时的想的构造），将sql语句变成`SELECT id, username FROM users WHERE username = 'admin\' AND password = 'or 2>1 limit 1 #'`，用\转义第二个引号，用#注释掉最后这个引号，然后就变成从users这个表里面满足`username ='admin\' AND password = '` or `2>1 limit 1 #'`条件的结果，而且有本地环境可以打印看执行了什么语句，所以说经过排查这个语句确实执行，但是就是无法从数据库中查询出东西，导致一直Invalid credentials，查询结果也是一直为空，对于sql语句的报错我们一直觉得很奇怪但是没有排查出什么。

后来xiaoyu学长做出来了，我才知道报错的原因是postgresql9之前的版本中可以用\进行转义，但是题目使用了postgres:13.6，在9之后的版本，\变成了普通字符，想用反斜杠来转义字符，要么在需要转义的字符串前面加上E，比如`E'ek\'1ng'`，这样就可以把引号转义变成`'ek'1ng'`，但是在题目的环境中我们显然做不到，另一个方法是用单引号来转义单引号`'ek''1ng'`，这样也可以得到`'ek'1ng'`，但是题目不是把单引号替换成两个单引号了么，我们还是没法输入一个单引号来转义后面那个单引号？

这里就需要用到题目给出的截取32个字符的部分了，由于是先将一个单引号替换成两个单引号，然后再截取前32个字符，那么我们只需要拼凑31个字符然后末尾加个单引号，单引号变成两个单引号后，33个字符被截取掉末尾一个，来绕过对单引号的限制，从而达到和我们想转义username末尾单引号的目的。

此时username为`1234567890abcdef1234567890abcde'`，password就可以使任何我们想执行的sql语句了。

并且题目给出了初始化数据库的文件init.sql，

```sql
CREATE TABLE users (id serial primary key, username text, password text);
INSERT INTO users (username, password) VALUES ('zuck', 'hunter2');
INSERT INTO users (username, password) VALUES ('randall', 'correcthorsebatterystaple');
INSERT INTO users (username, password) VALUES ('richard', 'cowabunga');

CREATE TABLE flag (flag text);
INSERT INTO flag (flag) VALUES ('PCTF{SAMPLE_WEB_FLAG}');

CREATE USER amongst WITH PASSWORD 'amongst';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO amongst;
```

我们可以看到除了3个用来完成游戏的用户和密码外，还有一个flag表，flag表里面有个字段flag，flag的值就是我们要拿的东西。

因此就像上面说的一样，绕过引号然后使用联合注入查询flag表，就可以得到flag辣。

```sql
1234567890abcdef1234567890abcde'

UNION SELECT 1, flag from flag--
```

填1，2还是3要看他想让你查询的这个用户的id是几，不然会报错，本地环境成功打通。

![image-20220409205348855](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220409205348855.png)

接下来打远程环境，也是成功打通

![image-20220410141719151](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220410141719151.png)




