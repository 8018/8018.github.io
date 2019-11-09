---
layout:     post
title:      Java 并发-6 Synchronized
date:       2015-09-06
author:     xflyme
header-img: img/post-bg-2015-09-06.jpg
catalog: true
tags:
    - java
    - 并发
---

Synchronized 一直是 Java 并发编程中不可缺少的一个角色，本文分析下 Synchronized 的基本用法以及原理。

### 用法

Java 中的每一个对象都可以作为锁，具体表现为以下三种形式：
1. 普通同步方法，锁的是当前实例对象。
2. 静态同步方法，锁的是当前类的class 对象。
3. 同步方法块，锁的是括号里面的对象。

### Java 对象头与 Monitor
先来看一下 Java 对象头和 Monitor 这对理解 synchronized 实现原理很关键。

在 JVM 中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。如图：

![图一](/img/java-6-1.jpeg)

* 实例变量：存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分信息按4字节对齐。
* 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐，这点了解即可。

而对于顶部，则是Java头对象，它实现synchronized的锁对象的基础，这点我们重点分析它，一般而言，synchronized使用的锁对象是存储在Java对象头里的，jvm中采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来的1个字记录的是数组长度)，其主要结构是由Mark Word 和 Class Metadata Address 组成，其结构说明如下表：


| 虚拟机位数 | 头对象结构 | 说明 |
| --- | --- | --- |
| 32/64bit |Mark Word  |存储对象的hashCode、锁信息或分代年龄或GC标志等信息  |
|32/64bit  |Class Metadata Address  |类型指针指向对象的类元数据，JVM通过这个指针确定该对象是哪个类的实例。  |

其中Mark Word在默认情况下存储着对象的HashCode、分代年龄、锁标记位等以下是32位JVM的Mark Word默认存储结构

|锁状态|25bit|4bit|1bit是否是偏向锁|2bit 锁标志位|
| --- | --- | --- | --- | --- |
|无锁状态|对象HashCode|对象分代年龄|0|01|

由于对象头的信息是与对象自身定义的数据没有关系的额外存储成本，因此考虑到JVM的空间效率，Mark Word 被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间，如32位JVM下，除了上述列出的Mark Word默认存储结构外，还有如下可能变化的结构：

![图二](/img/java-6-2.png)

其中轻量级锁和偏向锁是Java 6 对 synchronized 锁进行优化后新增加的，稍后我们会简要分析。这里我们主要分析一下重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）

```c
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示

![图三](/img/java-6-3.png)

由此看来，monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因(关于这点稍后还会进行分析)，ok~，有了上述知识基础后，下面我们将进一步分析synchronized在字节码层面的具体语义实现。

### synchronized代码块底层原理

现在我们重新定义一个synchronized修饰的同步代码块，在代码块中操作共享变量i，如下

```java
public class SyncCodeBlock {

   public int i;

   public void syncTask(){
       //同步代码库
       synchronized (this){
           i++;
       }
   }
}
```
编译上述代码并使用javap反编译后得到字节码如下(这里我们省略一部分没有必要的信息)：

```c
Classfile /Users/zejian/Downloads/Java8_Action/src/main/java/com/zejian/concurrencys/SyncCodeBlock.class
  Last modified 2017-6-2; size 426 bytes
  MD5 checksum c80bc322c87b312de760942820b4fed5
  Compiled from "SyncCodeBlock.java"
public class com.zejian.concurrencys.SyncCodeBlock
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
  //........省略常量池中数据
  //构造函数
  public com.zejian.concurrencys.SyncCodeBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
  //===========主要看看syncTask方法实现================
  public void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter  //注意此处，进入同步方法
         4: aload_0
         5: dup
         6: getfield      #2             // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2            // Field i:I
        14: aload_1
        15: monitorexit   //注意此处，退出同步方法
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit //注意此处，退出同步方法
        22: aload_2
        23: athrow
        24: return
      Exception table:
      //省略其他字节码.......
}
SourceFile: "SyncCodeBlock.java"

```

我们主要关注字节码中的如下代码

```c
3: monitorenter  //进入同步方法
//..........省略其他  
15: monitorexit   //退出同步方法
16: goto          24
//省略其他.......
21: monitorexit //退出同步方法
```

### synchronized方法底层原理
方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。下面我们看看字节码层面如何实现：
```java
public class SyncMethod {

   public int i;

   public synchronized void syncTask(){
           i++;
   }
}
```

使用javap反编译后的字节码如下：

```c
Classfile /Users/zejian/Downloads/Java8_Action/src/main/java/com/zejian/concurrencys/SyncMethod.class
  Last modified 2017-6-2; size 308 bytes
  MD5 checksum f34075a8c059ea65e4cc2fa610e0cd94
  Compiled from "SyncMethod.java"
public class com.zejian.concurrencys.SyncMethod
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool;

   //省略没必要的字节码
  //==================syncTask方法======================
  public synchronized void syncTask();
    descriptor: ()V
    //方法标识ACC_PUBLIC代表public修饰，ACC_SYNCHRONIZED指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
}
SourceFile: "SyncMethod.java"

```

从字节码中可以看出，synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是synchronized锁在同步代码块和同步方法上实现的基本原理。

### synchronized 的历史

Java 的线程是映射到操作系统的原生线程之上的，要阻塞或唤醒一个线程都需要操作系统的帮忙，这就需要从用户态切换到内核态，因此状态切换需要消耗很多的处理器时间。

互斥同步最主要的问题就是进行线程阻塞和唤醒所带来的性能问题。在 Java 1.5（包括）之前，Synchronized 性能问题很严重，下面是 JDK1.5 环境中 synchronized 与 ReentrantLock 的吞吐量对比：

![图四](/img/java-6-4.jpeg)

![图五](/img/java-6-5.jpeg)

Java 6之后 Java 官方对从JVM层面对 synchronized 较大优化，所以现在的synchronized 锁效率也优化得很不错了，Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，引入了轻量级锁和偏向锁，接下来我们了解一下偏向锁、轻量级锁以及锁的升降级。

### 锁的升级与对比
Java SE 1.6 为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”。在 Java SE 1.6 中锁一共有四种状态，级别从低到高依次是：无所状态、偏向锁状态、轻量级锁状态和重量级锁状态。这几个状态会随着竞争逐渐升级，锁可以升级但不能降级。这种锁可以升级但不能降级的的目的是为了提高获得锁和释放锁的效率。

#### 偏向锁
大多数情况下，锁不仅不存在多线程竞争，而且总是有同一线程多次获得，为了让线程获取锁的代价降低而引入了偏向锁。当一个线程访问同步代码并获得锁时，会在对象头和栈帧中的锁记录里存储偏向的线程 ID ，以后在该线程进入和退出同步代码时不需要进行 CAS 操作来加锁和释放锁，只需要简单的测试一下对象哦图的 Mark Word 里的线程 ID 指向当前线程。如果测试成功，表示当前线程已经获得了锁，如果测试失败，则需要测试一下 Mark Word 中偏向锁的标示是否被设置成1，如果没有设置则使用 CAS 竞争锁；如果设置了，则尝试使用 CAS 将对象头的偏向线程 ID 指向当前线程。

##### 偏向锁的撤销
偏向锁使用了一种等到竞争出现时才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等到全局安全点。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程处于不活跃状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈将会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的 Mark Word 要么重新偏向其他线程要么恢复到无锁状态或标记对象不适合作为偏向锁，最后唤醒暂停的线程。

#### 轻量级锁
##### 轻量级加锁
线程在执行同步块之前，JVM 会在当前线程的栈帧中创建用于存储锁记录的空间，并将对象头中的 Mark Word 复制到锁记录中，官方称为 Displaced Mark Word。然后线程尝试使用 CAS 将对象头中的 Mark Word 替换为指向锁记录中的指针。如果成功则表示当前线程获得锁，如果失败则表示其他线程竞争锁，当前线程尝试使用自旋来获取锁。

###### 轻量级解锁
轻量级解锁时，会使用原子的 CAS 操作将 Displaced Mark Word 替换回对象头中，如果成功，则表示没有竞争发生。如果失败则表示当前锁存在竞争，锁就会膨胀成重量级锁。

因为自旋会消耗 CPU，为了避免无用的自旋，一旦锁升级成重量级锁，就不会恢复到轻量级状态。当锁处于这个状态，其他线程试图获取锁时，都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进入到新一轮的竞争之中。

### 锁优化
高效并发是从 JDK 1.5 到 JDK 1.6 的一个重要改进，HotSpot 虚拟机团队在这个版本上花费了巨大的精力去实现各种锁优化技术，如适应性自旋、锁消除、锁粗化、轻量级锁和偏向锁。这些技术都是为了在线程之间更高效的共享数据，以及解决竞争问题。

#### 自旋与适应性自旋
互斥同步对性能最大的影响是阻塞的实现，挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给操作系统的并发性能带来很大的压力。在许多应用商，共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和恢复线程并不值得。如果物理机器上有一个以上的处理器能让两个或两个以上的线程同时并行执行，我们就可以让后面请求锁的那个线程「稍等一下」但不放弃处理器的执行时间，看看持有锁的线程是否会很快释放锁。为了让线程等待，我们只需要让线程执行一个循环，这就是所谓的自旋。

在 JDK 1.6 中引入了适应性自旋。自适应意味着自旋的时间不在固定了，而是有前一次在同一个锁上的自旋时间以及锁的拥有者的状态来确定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋很有可能再次成功，进而允许它等待相对更长的时间。如果某个锁，自旋很少成功获得过，那以后想要获得这个锁时很可能省略掉自旋，以避免浪费处理器资源。有了自适应自旋，随着程序运行和性能监控信息的不断完善，虚拟机对锁的状况预测就会越来越准确。

#### 锁消除
锁消除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。锁消除的主要判定依据来源于逃逸分析的数据支持，如果在一段代码中，堆上所有数据都不会逃逸出去进而被其他线程访问到，那就可以把它们当作栈上数据对待，认为他们是线程私有的，同步加锁自然就无需进行。

下面这段代码就会被优化：
```java
    public String concatString(String s1,String s2,String s3){
        return s1+s2+s3;
    }
```

这段代码看起来没有锁，String 是一个不可变类，对字符串的操作总是通过生成新的 String
对象来进行的。因此 Java 编译器会对 String 连接做自动优化，在 JDK 1.5 之前的版本中会转化为 StringBuffer 对象的 append() 操作：
```java
    public String concatString(String s1,String s2,String S3){
        StringBuffer sb = new StringBuffer();
        sb.append(s1);
        sb.append(s2);
        sb.append(s3);
        return sb.toString();
    }
```

每个 StringBuffer.append() 方法中都有一个同步块，在这段代码中锁永远不会逃逸到方法之外。因此虽然这里有锁，但可以被完全的消除。

#### 锁粗化

原则上，我们在编写代码的时候，总是推荐将同步块的作用范围限制的尽量小，只在数据共享的实际作用域中才进行同步，这样是为了使得需要进行同步的操作数量尽可能变小，如果存在数据竞争，那么等待锁的线程也能尽快拿到锁。
大部分情况下，上面的原则都是正确的，但是如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁的进行互斥同步操作也会导致不必要的性能损耗。

如果虚拟机探测到有一系列零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展整个操作序列的外部。


































