---
layout:     post
title:      Java 并发-9  Condition
date:       2015-09-27
author:     xflyme
header-img: img/post-bg-2015-09-27.png
catalog: true
tags:
    - java
    - 并发
    - 源码分析
---

#### 前言
上一节我们学习了 AbstractQueuedSynchronizer ，这一节我们看一下 Condition，Condition 是实现多种并发工具类的基础。

#### 简介
任意一个 Java 对象，都拥有一组监视器方法（定义在 java.lang.Object 上），主要包括 wait()、wait(long timeout)、notify()以及 notifyAll（），这些方法与 Synchronized 关键字配合可以实现等待、通知机制。
Condition 接口也提供了类似 Object 的监视器方法，与 Lock 关键字配合可以实现等待、通知模式。

#### 源码解读 ArrayBlockingQueue

##### 成员变量

```java
  /** The queued items */
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```

在成员变量中我们看到有 ReentrantLock、notEmpty Condition 和 notFull Condition。

```java
  public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

上面这段代码是 ArrayBlockingQueue 的构造函数，两个 Condition 是在构造函数中以 ReentrantLock#newCondition 的方法被创建的。

ReentrantLock#newCondition
```java
 public Condition newCondition() {
        return sync.newCondition();
    }
```

继续跟踪，Sync#newCondition
```java
final ConditionObject newCondition() {
            return new ConditionObject();
        }
```

返回的是一个 ConditionObject 对象，ConditionObject 对象实现了 Condition 接口。

我们在返回阻塞队列中，看一下当有元素入队或出队时会有怎样的逻辑。

```java

    /**
     * Inserts the specified element at the tail of this queue, waiting
     * for space to become available if the queue is full.
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        Objects.requireNonNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }

```

在 put 方法中首先要获得锁，目的是确保数组修改的可见性和排他性。当数组数量等于数组的下标时，表示数组已满，调用 notFull.await()，当前线程随之释放并进入等待状态。如果数组容量不等于数组长度，表示数组未满，添加元素到数组中，同时通知等待在 notEmpty 上的线程，数组中已经有新元素可以获取。

接下来再来看一下 await()
ConditionObject#await()
```java
 public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

ConditionObject#addConditionWaiter()
```java
private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }

            Node node = new Node(Node.CONDITION);

            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

这段代码中，unlinkCancelledWaiters() 是遍历所有节点，把取消等待的节点移除，这段就不细读了。

然后构造一个新的节点，并把这个节点加入队列中。
```java
 Node(int waitStatus) {
            U.putInt(this, WAITSTATUS, waitStatus);
            U.putObject(this, THREAD, Thread.currentThread());
        } 
```
现在返回到上面的 await() 方法，走到了 fullyRelease 这一步，因为修改数组元素之前先获得了锁，现在进入等待状态要把锁释放掉。
```java
 final int fullyRelease(Node node) {
        try {
            int savedState = getState();
            if (release(savedState))
                return savedState;
            throw new IllegalMonitorStateException();
        } catch (Throwable t) {
            node.waitStatus = Node.CANCELLED;
            throw t;
        }
    }
    
    
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

如果成功释放则返回并唤醒同步队列的头节点，释放不成功，则将当前节点标记为 cancled，并抛出异常。

```java
 final boolean isOnSyncQueue(Node node) {
 //该节点的标志为 CONDITION 或该节点的 prev 为 null ，则该节点一定不在同步队列里
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
  //注意这里是 next 不是 nextWaiter
        if (node.next != null) // If has successor, it must be on queue
            return true;
 //在同步队列里从尾部开始查找该节点
        return findNodeFromTail(node);
    }

    /**
     * Returns true if node is on sync queue by searching backwards from tail.
     * Called only when needed by isOnSyncQueue.
     * @return true if present
     */
    private boolean findNodeFromTail(Node node) {
        // We check for node first, since it's likely to be at or near tail.
        // tail is known to be non-null, so we could re-order to "save"
        // one null check, but we leave it this way to help the VM.
        for (Node p = tail;;) {
            if (p == node)
                return true;
            if (p == null)
                return false;
            p = p.prev;
        }
    }
```
再继续回到 await() 如果当前节点不在同步队列里，则调用 LockSupport.Park 阻塞当前线程。

至此，线程阻塞已经跟踪完了，下面我们看一下线程的唤醒:
ConditionObject#singal
```java
     public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

```

找到头节点，然后调用 doSingal 方法：
```java
   private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }



//两步操作，首先enq将该node添加到CLH队列中，其次若CLH队列原先尾节点为CANCELLED或者对原先尾节点CAS设置成SIGNAL失败，则唤醒node节点；否则该节点在CLH队列总前驱节点已经是signal状态了，唤醒工作交给前驱节点（节省了一次park和unpark操作）
    final boolean transferForSignal(Node node) {
        //如果CAS失败，则当前节点的状态为CANCELLED
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        //enq将node添加到CLH队列队尾，返回node的prev节点p
        Node p = enq(node);
        int ws = p.waitStatus;
        //如果p是一个取消了的节点，或者对p进行CAS设置失败，则唤醒node节点，让node所在线程进入到acquireQueue方法中，重新进行相关操作
        //否则，由于该节点的前驱节点已经是signal状态了，不用在此处唤醒await中的线程，唤醒工作留给CLH队列中前驱节点
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```
