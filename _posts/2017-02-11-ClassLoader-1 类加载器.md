---
layout:     post
title:      ClassLoader-1 类加载器
date:       2017-02-11
author:     xflyme
header-img: img/post-bg-2017-02-11.jpg
catalog: true
tags:
    - java
    - ClassLoader
---


#### 什么是类加载器
Java 中的所有类，必须被装载到 jvm 中才能运行，这个装载工作是由 jvm 中的类装载器完成的，类装载器所做的工作的实质是把类文件从硬盘读到内存中。jvm 在装载类的时候都是通过 ClassLoader 的 loadClass() 方法来加载 class 的。

为了更好的理解类加载机制，我们来深入分析下 ClassLoader 和它的 loadClass() 方法。

#### 源码分析

```java
public abstract class ClassLoader {
```
ClassLoader 是一个抽象类，它的注释是：
```java
/**
 * A class loader is an object that is responsible for loading classes. The
 * class <tt>ClassLoader</tt> is an abstract class.  Given the <a
 * href="#name">binary name</a> of a class, a class loader should attempt to
 * locate or generate data that constitutes a definition for the class.  A
 * typical strategy is to transform the name into a file name and then read a
 * "class file" of that name from a file system.
 */
```
大致意思如下：
> ClassLoader 是一个抽象类，它负责加载 class。通常情况下通过给出的二进制的类名，ClassLoader 会尝试定位或者生成构成 class 的数据。一个经典的策略是把二进制名称转换成文件名然后从文件系统中读取它。

接下来我们看 loadClass 方法的实现：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先检查这个class 是否已经被装载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //尝试由父类 loadclass
                        c = parent.loadClass(name, false);
                    } else {
                       //尝试由虚拟机内置类加载器加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    //自己找这个 class 然后实现加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```
从上面的流程可以看到，ClassLoader 不会立即去加载 class ，而是先尝试由父 ClassLoader 加载目标类。这个流程跟委派模型也有关系，我们今天先不分析它。
在 parent 为 null 的情况下，它还尝试了由启动类加载器去加载。
在上面两种都没有找到目标类的情况下才由自己去 findClass()。

synchronized (getClassLoadingLock(name)) 这是一个同步代码块，它里面包含的应该是一个对象，下面看一下它到底是一个什么对象。
```java
protected Object getClassLoadingLock(String className) {
        Object lock = this;
        if (parallelLockMap != null) {
            Object newLock = new Object();
            lock = parallelLockMap.putIfAbsent(className, newLock);
            if (lock == null) {
                lock = newLock;
            }
        }
        return lock;
    }
```
这个方法的注释中是这样写的：
> 如果这个 ClassLoader 注册为可并行的，它会返回一个关联了这个 className 的特定类型。反之直接返回 ClassLoader 自己。
> 也就是说如果注册了并行加载，它锁住的是一个和 className 所关联的对象，这样既提高了并行能力，又不会有安全性问题。

其他的代码中都已经有注释，不再详细解释。

