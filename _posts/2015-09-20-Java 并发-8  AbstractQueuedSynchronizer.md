---
layout:     post
title:      Java 并发-8  AbstractQueuedSynchronizer
date:       2015-09-20
author:     xflyme
header-img: img/post-bg-2015-09-20.jpg
catalog: true
tags:
    - java
    - 并发
---

### 简介
AbstractQueuedSynchronizer 是用来构建锁或其他同步组建的基础，在整个并发框架中占有非常重要的作用，下面我们来分析下 AbstractQueuedSynchronizer 的源码，看下它的工作原理。

AbstractQueuedSynchronizer 的主要使用方式是继承，子类通过继承 AbstractQueuedSynchronizer 并实现它的抽象方法来管理同步状态。

### 源码跟踪

从  ReentrantLock 的 lock() 方法开始：

```java
 public void lock() {
        sync.lock();
    }
```

它调用了 sync 的 lock() 方法。sync 在 ReentrantLock 的构造方法中创建。

```java
   /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

```
它分为公平锁和非公平锁两种，默认情况下是非公平锁，先看下非公平锁的 lock 方法：

```java

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```
假设这个时候还没有线程获得锁，此时第一个线程 thread1 进来尝试通过 cas 来更新 state 的值，因为没有其他线程占有锁，thread1 成功了，它得到了锁，并将 exclusiveOwnerThread 设置为当前线程。

此时来了第二个线程 thread2 ，假设 thread1 还没有释放锁，此时 CAS 获取锁会失败，然后走 acquire(1) 分支：

```java
  public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

这段代码对应的图如下：

![图1](/img/java-8-1.png)

![图2](/img/java-8-2.png)


这样看起来逻辑更清晰，首先来看下 tryAcquire：

AbstractQueuedSynchronizer#tryAcquire
```java
protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```
默认抛出一个异常，由子类重写次函数来完成自己的逻辑

NonfairSync#tryAcquire:
```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```
nonfairTryAcquire:
```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
一共有三个分支
如果 state 为 0 说明没有线程占有锁，此时通过 CAS 更新 state 的状态。
如果 state 不为 0 看当前线程是否是占有锁的线程，如果是 更新 state 值。
以上都不是的话 返回 false。
根据上面我们的假设，thread1 还没有释放锁，thread2 此时获取不到锁。然后进入 addWaiter

addWaiter：
```java
private Node addWaiter(Node mode) {
        Node node = new Node(mode);

        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                U.putObject(node, Node.PREV, oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
            } else {
                initializeSyncQueue();
            }
        }
    }
```
很明显在 addWaiter 的内部：
首先将尝试获取锁的线程也就是 thread2 封装为一个 Node 对象，并且我们知道在当前执行环境下线程队列是空的，所以直接进入 initializeSyncQueue 方法：

```java
 /**
     * Initializes head and tail fields on first contention.
     */
    private final void initializeSyncQueue() {
        Node h;
        if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
            tail = h;
    }
```
此方法创建了一个空的节点，并通过 CAS 将此节点赋值给 head 节点，然后 tail 节点也指向此节点，此时的节点状态如下：

![图3](/img/java-8-3.jpg)

然后返回 addWaiter 函数，此时 oldTail 不是 null了，然后做了以下两个操作：
1、 将刚才传递进来的 node 的前驱节点设置为 oldTail。
此时结构如下：
![图4](/img/java-8-4.jpg)

2、 通过 CAS 操作，将我们传递进来的 node 设置为 tail 节点，并将之前尾节点的 next 指向传递进来的 node 节点。此时结构如下：

![图5](/img/java-8-5.jpg)

这个时候代码中出现了 return 关键字，也就是通过两次循环跳出了这个自旋体系。

按照代码执行流程，接下来会执行 acquireQueued 方法。现在我们来看下 acquireQueued 方法，它内部也是一个自旋循环：

```java
final boolean acquireQueued(final Node node, int arg) {
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
```
首先会获取当前节点的前驱节点，判断它是不是头节点，现在我们的状态是：

![图6](/img/java-8-6.jpg)

和明显当前 node 节点的前驱节点是 head 节点，然后再次尝试获得锁，此时 thread1 还没有释放锁，获取失败，然后会进入 shouldParkAfterFailedAcquire 方法：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
        return false;
    }
```
1. 如果前驱节点的waitStatus为-1，也就是SIGNAL，就返回true。

2. 如果当前节点的前驱节点的waitstatus大于0，也就是说被CANCEL掉了，这时候通过 do while 循环去除 waitstatus 大于 0 的节点。

3. 如果都不是以上的情况，就通过CAS操作将这个前驱节点设置成SIGHNAL。

很明显，我们在这里的情况是第3种情况，并且这个方法运行后返回false。

此时的结构如下，主要是head节点的waitStatus由0变成了-1。

![图7](/img/java-8-7.jpg)

第二次循环依然会尝试获得锁，此时 thread1 还没有释放，依然失败。然后再次进入 shouldParkAfterFailedAcquire 方法，检查前驱节点的额 waitstatus。此时 waitstatus 为 -1 返回 true。然后调用 parkAndCheckInterrupt 方法，直接将当前线程 thread2 禁止：


```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

假设现在 thread1 还没有释放锁，然后又来了一个线程 thread3，然后一路再走到 acqureQueued 时，第一次循环判断 node 的前驱节点是否是 head 节点，现在我们的状态是：

![图8](/img/java-8-8.jpg)

node 节点的前驱节点不是 head 节点，走第二个 if 分支，调用 shouldParkAfterFailedAcquire 方法。

1. 如果前驱节点的waitStatus为-1，也就是SIGNAL，就返回true。

2. 如果当前节点的前驱节点的waitstatus大于0，也就是说被CANCEL掉了，这个时候我们会除掉这个节点。

3. 如果都不是以上的情况，就通过CAS操作将这个前驱节点设置成SIGHNAL。

很明显，我们在这里的情况是第3种情况，并且这个方法运行后返回false。

此时的结构如下，主要是t节点的waitStatus由0变成了-1。

![图9](/img/java-8-9.jpg)

然后第二次循环，将 thread3 也挂起。

### 锁的释放过程
现在 thread1 要释放锁了，调用 unlock 方法，unlock 又调用了内部的 release 方法：

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
tryRelease:
```java
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
获取当前AQS的state，并减去1，判断当前线程是否等于AQS的exclusiveOwnerThread，如果不是，就抛异常，这就保证了加锁和释放锁必须是同一个线程。如果(state-1)的结果不为0，说明锁被重入了，需要多次unlock。如果(state-1)等于0，我们就将AQS的ExclusiveOwnerThread设置为null。如果上述操作成功了，也就是tryRelase方法返回了true，那么就会判断当前队列中的head节点，当前结构如下：

![图10](/img/java-8-10.jpg)

如果head节点不为null，并且head节点的waitStatus不为0, 我们就调用unparkSuccessor方法去唤醒head节点的后继节点。
```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
第一步：获取head节点的waitStatus，如果小于0，就通过CAS操作将head节点的waitStatus修改为0，现在是：

![图11](/img/java-8-11.jpg)

第二步：寻找head节点的下一个节点，如果这个节点的waitStatus小于0，就唤醒这个节点，否则遍历下去，找到第一个waitStatus<=0的节点，并唤醒。现在thread2线程被唤醒了，我们知道刚才thread2在acquireQueued被中断，现在继续执行，又进入了for循环，当前节点的前驱节点是head并且调用tryAquire方法获得锁并且成功。那么设置当前Node为head节点，将里面的thead和prev设置为null。

```java
 final boolean acquireQueued(final Node node, int arg) {
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
```

![图12](/img/java-8-12.jpg)

调用完毕后，acquireQueued返回false。并且现在thread2自由了。
以上是 NonfairSync 的获取和释放锁的流程。下面看一下 FairSync 的获取锁的代码：

```java
 /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

FairSync 的 tryAcquire 方法在尝试获取锁的时候先判断等待队列里是否有节点，以实现公平。





























































