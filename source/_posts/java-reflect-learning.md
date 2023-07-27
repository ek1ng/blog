---
title: 重学 Java 反射机制
date: 2023-07-25 20:48:00
updated: 2023-07-25 20:10:00
tags: ["security","java security"]
category: Security
---

> 近期跟一些java的最新漏洞，发现自己的语言基础太差了，跟着p牛的java安全漫谈重新学一下反射，p牛的文章确实是讲复杂的东西讲的浅显易懂。

## 反射的定义

**对象可以通过反射获取对应的类，类可以通过反射获取所有方法，拿到的方法可以调用，这种机制就是反射。**

## 反射机制在安全方面的意义

例如我们要完成RCE，但代码中绝大多数时候并没有Runtime，ProcessBuilder等常见的用于命令执行的类来让我们调用。因此，比如说通过反射获取一个Runtime类的对象，调用它的exec方法就是完成RCE的重要利用手段之一。

## 如何通过反射执行命令
> 以反射的几种技巧为例子，将讲如何通过反射执行命令。
> 
> 主要学的是代码中如何使用反射的**思路**。

- 获取类的⽅法： forName
- 实例化类对象的⽅法： newInstance
- 获取函数的⽅法： getMethod
- 执⾏函数的⽅法： invoke

### Runtime

从一段代码来看
```java
Class clazz = Class.forName("java.lang.Runtime");  
Object runtimeObject = clazz.newInstance();  
Method execMethod = clazz.getMethod("exec", String.class);  
execMethod.invoke(runtimeObject,"open -a Calculator");
```

这段代码的思路是获取Runtiem类，实例化一个Runtime对象，再到获取exec方法，传参调用exec方法。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230725153245.png)

但是实际上会在第二行报错，因为Runtime类的构造方法是私有方法，并且给出了一个静态方法getRuntime方法来获取当前的runtime。

部分源码：

```
private static Runtime currentRuntime = new Runtime();  
  
/**  
* Returns the runtime object associated with the current Java application.  
* Most of the methods of class <code>Runtime</code> are instance  
* methods and must be invoked with respect to the current runtime object.  
*  
* @return the <code>Runtime</code> object associated with the current  
* Java application.  
*/  
public static Runtime getRuntime() {  
return currentRuntime;  
}  
  
/** Don't let anyone else instantiate this class */  
private Runtime() {}
```

在单例模式中，构造方法为私有比较常见。比如数据库连接的建立只需要一次，不能每次使用这个类，构造方法都去连接一次数据库，而是希望能够获取那个唯一的数据库连接。那么就可以将构造方法设为私有，并且通过一个静态方法来获取这个数据库连接，这就是单例模式。

```java
Class clazz = Class.forName("java.lang.Runtime");  
Method getRuntimeMethod = clazz.getMethod("getRuntime");  
Object runtimeObject = getRuntimeMethod.invoke(clazz);  
Method execMethod = clazz.getMethod("exec",String.class);  
execMethod.invoke(runtimeObject,"open -a Calculator");
```
修改代码，获取getRuntime方法，调用方法获取runtime对象，再通过这个runtime对象来调用exec方法，就可以成功执行命令啦。

Runtime的构造方法是私有的，但我们可以通过getRuntime去获取runtime对象。那么假设我们的目标是一个非单例模式设计的类，而他的构造方法又是私有的时，我们就需要getDeclaredMethod/getDeclaredConstructor来获取私有函数/构造方法。

getMethod获取的是当前类中所有公共方法，包括从父类继承的方法。而getDeclaredMethod获取的是当前类中“声明”的方法，包含私有方法在内所有写在这个类的方法，但获取不到从父类继承的方法。类似的，也有getDeclaredConstructor。

通过getDeclaredConstructo获取构造方法后，需要用setAccessible修改作用域，接着通过构造函数调用newInstance获取runtime对象。

```
Class clazz = Class.forName("java.lang.Runtime");  
Constructor runtimeConstructor = clazz.getDeclaredConstructor();  
runtimeConstructor.setAccessible(true);  
Method execMethod = clazz.getMethod("exec",String.class);  
Object runtimeObject = runtimeConstructor.newInstance();  
execMethod.invoke(runtimeObject,"open -a Calculator");
```

### ProcessBuilder

> ProcessBuilder是另一种通过反射完成RCE的常见方式，不同的是，ProcessBuilder类没有无参构造方法，也没有类似单例模式的静态方法来获取对象。

对于没有无参构造方法，也没有类似单例模式的静态方法来获取对象的情况，可以用getConstructor获取构造方法，再通过newInstance实例化对象。

getConstructor也是一个反射方法，它类似getMethod，接收的参数为构造函数列表类型，因为构造函数也是支持重载的，因此需要传入类型来确认唯一的构造函数。

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");  
Method startMethod = clazz.getMethod("start");  
Constructor processBuilder = clazz.getConstructor(List.class);  
Object command = processBuilder.newInstance(Arrays.asList("open","-a","Calculator"));  
startMethod.invoke(command);
```

ProcessBuilder有两个构造函数：

```java
public ProcessBuilder(List<String> command)
public ProcessBuilder(String... command)
```

这里用了第一个构造函数，getConstructor时传入List.class。

整体就是获取ProcessBuilder类，获取start方法，再获取构造方法并且通过构造方法实例化对象，最后通过invoke执行start方法。

那么如果是用第二个构造函数，参数是一个变长参数。

变长参数在编译时和数组是等效的，也不能重载。

```java
public void hello(String[] names) {}
public void hello(String...names) {}
```

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");  
Method startMethod = clazz.getMethod("start");  
Constructor processBuilder = clazz.getConstructor(String[].class);  
Object command = processBuilder.newInstance(new String[][]{{"open","-a","Calculator"}});  
startMethod.invoke(command);
```
