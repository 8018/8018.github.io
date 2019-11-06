---
layout:     post
title:      ClassLoader-2 双亲委派模型
date:       2017-02-12
author:     xflyme
header-img: img/post-bg-2017-02-12.jpg
catalog: true
tags:
    - java
    - ClassLoader
---


上一节[ClassLoader-1 类加载器](/2017/02/11/ClassLoader-1-类加载器/)，这一节我们看一下 jvm 中都有哪些类加载器，以及双亲委派模型。

#### jvm 中的类加载器
jvm 中自带三个类加载器：
* Bootstrap ClassLoader 最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变Bootstrap ClassLoader的加载目录。比如java -Xbootclasspath/a:path被指定的文件追加到默认的bootstrap路径中。我们可以打开我的电脑，在上面的目录下查看，看看这些jar包是不是存在于这个目录。
* Extention ClassLoader 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录。
* Appclass Loader也称为SystemAppClass 加载当前应用的classpath的所有类。

上一篇文章中在分析 loadClass() 源码中有看到，在加载一个类的时候它首先交给 parent 去加载。那么这个 parent 又是谁，它是在什么时候被创建的？

```java
protected ClassLoader(ClassLoader parent) {
        this(checkCreateClassLoader(), parent);
    }

    protected ClassLoader() {
        this(checkCreateClassLoader(), getSystemClassLoader());
    }
```

以上是 ClassLoader 的构造方法，可以在创建时指定 parent，如果没有指定，默认的 parent 是 SystemClassLoader。

#### 例子

先看一个示例，看一下各加载器以及他们的 parent，代码如下：

```java
public class Test {
    public static void main(String[] args) {
        
        System.out.println(Test.class.getClassLoader());
        System.out.println(ThreadLocalTest.class.getClassLoader().getParent());
        System.out.println(ThreadLocalTest.class.getClassLoader().getParent().getParent().toString());
    }
}

```
结果如下：
```java
sun.misc.Launcher$AppClassLoader@2a139a55
sun.misc.Launcher$ExtClassLoader@7852e922
Exception in thread "main" java.lang.NullPointerException
    at me.xfly.test.Test.main(Test.java:8)
```
可以看到 Test 是被 AppClasLoader 加载的，它的 parent 是ExtClassLoader。而 ExtClassLoader 没有 parent。

ExtClassLoader 和 AppClasLoader 是在 Launcher 中被创建的，我们看一下它的源码：

```java
public class Launcher {
    private static URLStreamHandlerFactory factory = new Factory();
    private static Launcher launcher = new Launcher();
    private static String bootClassPath =
        System.getProperty("sun.boot.class.path");

    public static Launcher getLauncher() {
        return launcher;
    }

    private ClassLoader loader;

    public Launcher() {
        // Create the extension class loader
        ClassLoader extcl;
        try {
            extcl = ExtClassLoader.getExtClassLoader();
        } catch (IOException e) {
            throw new InternalError(
                "Could not create extension class loader", e);
        }

        // Now create the class loader to use to launch the application
        try {
        //将ExtClassLoader对象实例传递进去
            loader = AppClassLoader.getAppClassLoader(extcl);
        } catch (IOException e) {
            throw new InternalError(
                "Could not create application class loader", e);
        }

public ClassLoader getClassLoader() {
        return loader;
    }
}
```

可以看到 AppClassLoader 在创建的时候指定了 parent 为 ExtClassLoader，而 ExtClassLoader 没有指定 parent，所以为 null。

#### 双亲委派模型
上一节我们看过 loadClass() 的源码，在 ClassLoader 中：
* 首先通过 findLoadedClass 查找目标 class。
* 如果没有找到，通过 parent 的 loadClass 查找 parent 是否已经加载过此类。
* 经过以上两步还没有找到的话则会进行 findClass 。

这个过程可以用下图表示：

![图一](/img/classloader-1-1.png)

##### 双亲委派模型的优点
* 简单 双亲委派模型首先是简单易实现，现在也有其他的委派模型，这个以后有机会再学习。
* 安全性 设想这样一个场景，如果一个人写了一个恶意的基础类（如java.lang.String），但是在双亲委派模型中它首先会检查 parent 是否已经加载这个类。没有被加载的话也会由 parent 去核心类库里去加载。这样这个恶意的类不会被加载。

#### Thread.contextClassLoader

```java
public class Thread implements Runnable {
    /* The context ClassLoader for this thread */
    private ClassLoader contextClassLoader;
    
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        ...
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        ...
    }
}
```

java 的 Thread 中有这样一段代码，这个 ClassLoader 如果没有显示指定，他默认从父线程那里继承过来。程序启动时的 main 线程的 contextClassLoader 就是 AppClassLoader。这意味着如果没有人工去设置，那么所有的线程的 contextClassLoader 都是 AppClassLoader。
既然可以显示设定 contextClassLoader 那么它一定有它的用处，这个以后再分析。

#### 参考
[深度分析Java的ClassLoader机制（源码级别）](http://www.hollischuang.com/archives/199)
[一看你就懂，超详细 java ClassLoader 详解](https://blog.csdn.net/briblue/article/details/54973413)

