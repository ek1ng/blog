---
title: Java 动态加载字节码机制
date: 2023-06-15 20:48:00
updated: 2023-06-18 20:10:00
tags: ["security","java security"]
category: Security
---

> 参考：《JAVA 安全漫谈》
## 什么是Java字节码

Java 是一门跨平台的编译型语言。在运行Java代码的整个过程中，代码会被编译成字节码在JVM中运行。而字节码其实就是JVM所支持的指令。

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230915110121.png)

## Java加载字节码机制的安全意义

在很多场景，我们可以有控制一些能够加载字节码函数的能力，那么可以通过加载远程的恶意类，来完成RCE。

## 如何加载类

### 一些概念的区
### 用 URLClassLoader 从远程加载类

ClassLoader是JVM的一个子系统，负责动态加载Java类和资源文件，通俗地说它告诉JVM如何加载这个类。而默认加载类的方式是根据类名加载，例如`java.lang.runtime`。

```java
ClassLoader.getSystemClassLoader().loadClass("java.lang.runtime");
```

URLClassLoader是ClassLoader的实现类，顾名思义，主要用于从URL路径加载类和资源，包括
从本地或者远程加载两种方式。

URLClassLoader是默认使用的AppClassLoader的父类，也就是说URLClassLoader的加载类流程，就是Java默认的类加载流程，接下来我们看一下URLClassLoader是怎么加载类的。

> 参考文章：https://juejin.cn/post/7057728820270333966

核心逻辑在函数`URLClassPath#getLoader(URL url)`
```java
private Loader getLoader(final URL url) throws IOException {
    try {
        return AccessController.doPrivileged(
                new PrivilegedExceptionAction<>() {
                    public Loader run() throws IOException {
                        String protocol = url.getProtocol();
                        String file = url.getFile();
                        if (file != null && file.endsWith("/")) {
                            if ("file".equals(protocol)) {
                                return new FileLoader(url);
                            } else if ("jar".equals(protocol) &&
                                    isDefaultJarHandler(url) &&
                                    file.endsWith("!/")) {
                                // extract the nested URL
                                URL nestedUrl = new URL(file.substring(0, file.length() - 2));
                                return new JarLoader(nestedUrl, jarHandler, lmap, acc);
                            } else {
                                return new Loader(url);
                            }
                        } else {
                            return new JarLoader(url, jarHandler, lmap, acc);
                        }
                    }
                }, acc);
    } catch (PrivilegedActionException pae) {
        throw (IOException)pae.getException();
    }
}
```

根据配置项 `sun.boot.class.path` 和 `java.class.path` 中列举到的基础路径（这些路径是经过处理后的 `java.net.URL`类）来寻找.class文件来加载，而这个基础路径有分为三种情况：

- URL 以`/`结尾，那么就是一个JAR，用`JarLoader`寻找类
- URL 不以`/`结尾，是`file`协议，就用`FileLoader`寻找类
- URL 不以`/`结尾，不是`file`协议，就创建一个`Loader`来寻找类

这里我们尝试用`http`协议从远程加载类，也就是第三种情况。
```java
public class Hello {  
    public Hello(){  
        System.out.println("Hello, world!");  
    }}
```

```java
package com.govuln.test;

import java.net.URL;
import java.net.URLClassLoader;

public class ClassLoaderTest {
    public static void main(String[] args) throws Exception {
        URL[] urls = {new URL("http://localhost:8000/")};
        URLClassLoader loader = URLClassLoader.newInstance(urls);
        Class c = loader.loadClass("Hello");
        c.newInstance();
    }
}
```

![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230918160309.png)

### 用 ClassLoader#defineClass 加载类

不论是从远程加载类还是从本地加载，类加载的过程都是：
ClassLoader#loadClass -> ClassLoader#findClass -> ClassLoader#defineClass。

其中：
- loadClass：主要有两个作用，一是运行时动态加载指定的类，加载过程会读取字节码文件，验证正确性，解析类的依赖关系等等。二是检测类是否重复加载，包括对类加载器的继承关系，已加载累的缓存进行检测。使用ClassLoader的loadClass方法加载类时，如果类缓存、父加载器等位置找不到类，就会传入url，调用findClass去加载类。
- findClass：根据传入的url，以对应的方式从本地class文件、jar包、远程http服务器等地方加载类的字节码，并且将字节码传给defineClass。
- defineClass：根据传入的已经加载的类的字节码，将其转换成对应的Class对象，并且返回。

接下来我们尝试使用ClassLoader#defineClass加载字节码，其中由于Class#defineClass是一个保护属性，需要通过反射来调用。

defineClass就像名字，定义类，它只做把一个类定义出来的工作，而不会初始化类对象。类对象还是需要通过显式调用构造函数，初始化代码才会被执行。所以如果我们的目标是任意代码执行，还需要想办法调用构造函数。

```java
package com.govuln.bytes;

import org.apache.commons.codec.binary.Base64;

import java.lang.reflect.Method;

public class HelloDefineClass {  
    public static void main(String[] args) throws Exception {  
        Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);  
        defineClass.setAccessible(true);  
  
        // source: bytecodes/Hello.java  
        byte[] code = Base64.decodeBase64("yv66vgAAADQAHAoABgAOCQAPABAIABEKABIAEwcAFAcAFQEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAAg8Y2xpbml0PgEAClNvdXJjZUZpbGUBAApIZWxsby5qYXZhDAAHAAgHABYMABcAGAEAC0hlbGxvIFdvcmxkBwAZDAAaABsBAAVIZWxsbwEAEGphdmEvbGFuZy9PYmplY3QBABBqYXZhL2xhbmcvU3lzdGVtAQADb3V0AQAVTGphdmEvaW8vUHJpbnRTdHJlYW07AQATamF2YS9pby9QcmludFN0cmVhbQEAB3ByaW50bG4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYAIQAFAAYAAAAAAAIAAQAHAAgAAQAJAAAAHQABAAEAAAAFKrcAAbEAAAABAAoAAAAGAAEAAAACAAgACwAIAAEACQAAACUAAgAAAAAACbIAAhIDtgAEsQAAAAEACgAAAAoAAgAAAAQACAAFAAEADAAAAAIADQ==");  
        Class hello = (Class)defineClass.invoke(ClassLoader.getSystemClassLoader(), "Hello", code, 0, code.length);  
        hello.newInstance();  
    }}
```


其中的字节码生成自如下文件:
```java
public class Hello {  
    public Hello() {  
    }  
  
    static {  
        System.out.println("Hello World");  
    }  
}
```


### 用 TemplatesImpl 加载类

TemplatesImpl 是很多反序列化链中都会使用到的类。

在 com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl 这个类中定义了一个内部类 TransletClassLoader :
```java
static final class TransletClassLoader extends ClassLoader {  
    private final Map<String,Class> _loadedExternalExtensionFunctions;  
  
     TransletClassLoader(ClassLoader parent) {  
         super(parent);  
        _loadedExternalExtensionFunctions = null;  
    }  
    TransletClassLoader(ClassLoader parent,Map<String, Class> mapEF) {  
        super(parent);  
        _loadedExternalExtensionFunctions = mapEF;  
    }  
    public Class<?> loadClass(String name) throws ClassNotFoundException {  
        Class<?> ret = null;  
        // The _loadedExternalExtensionFunctions will be empty when the  
        // SecurityManager is not set and the FSP is turned off        if (_loadedExternalExtensionFunctions != null) {  
            ret = _loadedExternalExtensionFunctions.get(name);  
        }        if (ret == null) {  
            ret = super.loadClass(name);  
        }        return ret;  
     }  
    /**  
     * Access to final protected superclass member from outer class.     */    Class defineClass(final byte[] b) {  
        return defineClass(null, b, 0, b.length);  
    }}
```

TransletClassLoader重写了defineClass，并且没有显式地声明其定义域，相当于用default声明。重写后本来的protected类型变成了default类型，使它可以被外部调用。

那么什么时候TransletClassLoader#defineClass()会被调用呢，这里推荐自己看一下源码，理解一下调用的过程

```java
TemplatesImpl#getOutputProperties()
-> TemplatesImpl#newTransformer()
-> TemplatesImpl#getTransletInstance()
-> TemplatesImpl#defineTransletClasses()
-> TransletClassLoader#defineClass()
```

其中getOutputProperties和newTransformer都是被public声明的，可以在外部调用，这里尝试用newTransformer来构造POC：

```java
package com.govuln.bytes;  
  
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;  
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;  
import org.apache.commons.codec.binary.Base64;  
  
import java.lang.reflect.Field;  
  
public class HelloTemplatesImpl {  
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {  
        Field field = obj.getClass().getDeclaredField(fieldName);  
        field.setAccessible(true);  
        field.set(obj, value);  
    }  
    public static void main(String[] args) throws Exception {  
        // source: bytecodes/HelloTemplateImpl.java  
        byte[] code = Base64.decodeBase64("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");  
        TemplatesImpl obj = new TemplatesImpl();  
        setFieldValue(obj, "_bytecodes", new byte[][] {code});  
        setFieldValue(obj, "_name", "HelloTemplatesImpl");  
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());  
  
        obj.newTransformer();  
    }}
```


上面的POC比较简单，setFieldValue用来设置私有属性，构造了`_bytecode`，`_name`，`_tfactory`三个属性来从newTransformer走到最终的defineClass，加载字节码调用了类HelloTemplatesImpl的构造方法

其中的字节码生成自如下文件，需要注意的是TemplatesImpl加载的字节码，需要是com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet的子类，所以HelloTemplatesImpl继承自AbstractTranslet类。
```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class HelloTemplatesImpl extends AbstractTranslet {
    public void transform(DOM var1, SerializationHandler[] var2) throws TransletException {
    }

    public void transform(DOM var1, DTMAxisIterator var2, SerializationHandler var3) throws TransletException {
    }

    public HelloTemplatesImpl() {
        System.out.println("Hello TemplatesImpl");
    }
}

```

### 用BCEL ClassLoader加载类

BCEL是Apache COmmons BCEL，提供了一系列用于分析、创建、修改文件的API，在原生JDK中，位于com.sun.org.apache.bcel。

至于BCEL在原生JDK中的原因：
> 引自P牛文章《 BCEL ClassLoader去哪了》
> 
> “据我（不严谨）的考证，JDK会将BCEL放到自己的代码中，主要原因是为了支撑Java XML相关的功能。准确的来说，Java XML功能包含了JAXP规范，而Java中自带的JAXP实现使用了Apache Xerces和Apache Xalan，Apache Xalan又依赖了BCEL，所以BCEL也被放入了标准库中。“
> 
> ”因为需要“编译”XSL文件，实际上核心是动态生成Java字节码，而BCEL正是一个处理字节码的库，所以Apache Xalan是依赖BCEL的。“

BCEL中有类com.sun.org.apache.bcel.internal.util.ClassLoader，重写了ClassLoader#loadClass()方法。

重写的ClassLoader#loadClass()
```java
protected Class loadClass(String class_name, boolean resolve)  
  throws ClassNotFoundException  
{  
  Class cl = null;  
  
  /* First try: lookup hash table.  
   */  if((cl=(Class)classes.get(class_name)) == null) {  
    /* Second try: Load system class using system class loader. You better  
     * don't mess around with them.     */    for(int i=0; i < ignored_packages.length; i++) {  
      if(class_name.startsWith(ignored_packages[i])) {  
        cl = deferTo.loadClass(class_name);  
        break;  
      }    }  
    if(cl == null) {  
      JavaClass clazz = null;  
  
      /* Third try: Special request?  
       */      if(class_name.indexOf("$$BCEL$$") >= 0)  
        clazz = createClass(class_name);  
      else { // Fourth try: Load classes via repository  
        if ((clazz = repository.loadClass(class_name)) != null) {  
          clazz = modifyClass(clazz);  
        }        else  
          throw new ClassNotFoundException(class_name);  
      }  
      if(clazz != null) {  
        byte[] bytes  = clazz.getBytes();  
        cl = defineClass(class_name, bytes, 0, bytes.length);  
      } else // Fourth try: Use default class loader  
        cl = Class.forName(class_name);  
    }  
    if(resolve)  
      resolveClass(cl);  
  }  
  classes.put(class_name, cl);  
  
  return cl;  
}
```

会判断类名是否是`$$BCEL$$`开头，如果是的话，将会对这个字符串进行decode。decode的过程是类似传统字节码的HEX编码，再将反斜线替换成`$`。默认情况下外层还会加一层GZip压缩。

接下来尝试用BCEL ClassLoader加载类

```java
package com.govuln.bytes;  
  
import com.sun.org.apache.bcel.internal.Repository;  
import com.sun.org.apache.bcel.internal.classfile.JavaClass;  
import com.sun.org.apache.bcel.internal.classfile.Utility;  
import com.sun.org.apache.bcel.internal.util.ClassLoader;  
public class HelloBCEL {  
    public static void main(String []args) throws Exception {  
        // encode();  
        decode();  
    }  
    protected static void encode() throws Exception {  
        JavaClass cls = Repository.lookupClass(evil.Hello.class);  
        String code = Utility.encode(cls.getBytes(), true);  
        System.out.println(code);  
    }  
    protected static void decode() throws Exception {  
         new ClassLoader().loadClass("$$BCEL$$$l$8b$I$A$A$A$A$A$A$AmP$cbN$CA$Q$ac$91$c7$$$cb$w$I$e2$fby0$B$P$ee$c5$h$c4$8b$89$f1$b0Q$T$M$9e$87e$82C$86$j$b3$M$q$7e$96$k4$f1$e0$H$f8Q$c6$9e$91$f8H$ecCW$ba$aa$ba$d23$ef$l$afo$AN$b0$X$a0$88$e5$Sj$a8$fbX$J$d0$c0$aa$875$P$eb$M$c5$8eL$a59e$c85$5b$3d$86$fc$99$k$I$86J$ySq9$j$f7Ev$c3$fb$8a$98Z$ac$T$aez$3c$93v$9e$93ys$t$t$Ma$yfRE$XB$v$ddf$f0$3b$89$9a$87$G$5d$3d$cd$Sq$$$ad$3bp$86$e3$R$9f$f1$Q$k$7c$P$h$n6$b1$c5Pv$ca$fe$ad$ce$d4$c0$c3v$88$j$ec$92$ff$t$95$a1j$d7$o$c5$d3at$d5$l$89$c4$fc$a1$ba$P$T$p$c6$f4$I$3d$r$a1$R$3bE$ea$e8$3a$93$a9$e9$9aL$f01$jV$ff$87f$f0$ee$ed$a4R$dak$c6$bf$o$N$d1$c3v$ab$87$D$U$e8$fbl$z$80$d9$c3$a9$97h$8a$I$Za$e1$e8$F$ec$d1$c9$B$f5$a2$ps$uS$P$bf$M$84$8b$84$3e$96$be$97$P$c9m$ab$f4$84$85Z$ee$Zy$h$c0$5c$40$e0$a4$CYmT$c5$FW$3f$B$dc$ab$c0$7f$cc$B$A$A").newInstance();  
    }}
```
![image.png](https://ek1ng-typora.oss-cn-hangzhou.aliyuncs.com/img/20230918205753.png)

被加载类代码：

```java
package com.govuln;

import com.sun.org.apache.bcel.internal.classfile.JavaClass;
import com.sun.org.apache.bcel.internal.classfile.Utility;
import com.sun.org.apache.bcel.internal.Repository;

public class HelloBCEL {
	public static void main(String []args) throws Exception {
	JavaClass cls = Repository.lookupClass(evil.Hello.class);
	String code = Utility.encode(cls.getBytes(), true);
	System.out.println(code);
	}
}
```

而在Java 8u251的更新中，BCEL ClassLoader被移除了。