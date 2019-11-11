---
layout:     post
title:      Java 并发-10 ThreadLocal
date:       2015-10-04
author:     xflyme
header-img: img/post-bg-2015-10-04.jpg
catalog: true
tags:
    - java
    - 并发
    - 源码分析
---

### 关键词
* 合理发布
* 线性探测
* 内存泄漏

### 合理发布
Java 线程间通信使用的是共享内存机制。在共享内存并发模型里，线程之间共享程序的公共状态——实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。在这种情况下可能会造成数据的安全性、一致性等问题。

ThreadLocal 的目标是不该共享的变量不被「发布」出去，从而保证数据的安全性。

##### How ？

```java
 ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
 /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

创建一个跟线程关联的「哈希表」，把数据存在这个表里。
那么 ThreadLocalMap 里保存的数据一定不能在其他线程里访问吗？不一定! 要合理的给 ThreadLocal 设置数据，不要把全局的实例域、静态域等添加进去。而是以以下方法初始化数据：

```java
 ThreadLocal local = new ThreadLocal(){
            @Override
            protected Object initialValue() {
                return super.initialValue();
            }
        };
```

这样就够了吗？还是不一定，当使用线程池的时候，如果没有进行及时清理同样会产生数据一致性、数据安全性问题。

### 线性探测
ThreadLocalMap 解决散列冲突的方式是线性探测：
> 当我们往散列表中插入数据时，如果某个数据经过散列函数散列之后，存储位置已经被占用了，我们就从当前位置开始，依次往后查找，看是否有空闲位置，直到找到为止。

除了线性探测还有**二次探测**、**双重散列**、**链表法** 等解决散列冲突的方法，这里就不深入介绍了。

线性探测法其实存在很大问题。当散列表中插入的数据越来越多时，散列冲突发生的可能性就会越来越大，空闲位置会越来越少，线性探测的时间就会越来越久。极端情况下，我们可能需要探测整个散列表，所以最坏情况下的时间复杂度为 O(n)。同理，在删除和查找时，也有可能会线性探测整张散列表，才能找到要查找或者删除的数据。因此线性探测适合数据量比较小的散列表。

### 内存泄漏

![图一](/img/java-10-01.jpg)

图中实线代表强引用，虚线代表弱引用。如果 threadLocal 外部强引用被置为 null，threadLocal 实例就没有一条引用链路可达，下一次 gc 的时候它会被回收。
而其中的 value 依然是 GCRoots 可达的，但是 key 已经为 null，该 Entry 以及 value 永远不会被访问到也就造成了**内存泄漏**。

ThreadLocal 不能完全规避这个问题，不过它会在 get 和 set 数据的时候尽量清理一些 stale「不新鲜」entry。
我们看下它是怎么做的。

##### set

```java
private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            //线性探测
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }
                //如果存在 k == null，替换不新鲜节点
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            //没有清理插槽，而且 sz 大雨装载因子 扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

这个方法不难看懂而且已经添加了注释，就不再详细解释了，下面我们看 replaceStaleEntry。

##### replaceStaleEntry

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            int slotToExpunge = staleSlot;
            //从「不新鲜」插槽向前，直至 null 节点
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                 
                //如果存在 key == null 节点，更新「需要被删除的插槽」的标记
                if (e.get() == null)
                    slotToExpunge = i;

            //从「不新鲜」插槽向后，直至 null 节点
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                
                if (k == key) {
                    //如果存在 k == key
                    e.value = value;
                    
                    tab[i] = tab[staleSlot];
                    //把需要存储的数据放入 staleSlot 节点
                    tab[staleSlot] = e;

                    
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    //删除不新鲜节点，同时尝试清理一些 slot
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```

这里会根据 staleSlot 向前和向后查找，它的查找范围是两个 null 之间（方法注释里也有提到）。
最终需要插入的数据还是放到了 staleSlot 这个节点。
如果除了 staleSlot 还有其他不新鲜（entry ！= null && k == null）会更新 slotToExpunge 后续走 expungeStaleEntry进行清理。
接下来看一下 expungeStaleEntry。

##### expungeStaleEntry

```java
private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // 删除 staleSlot 所在的 entry
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            Entry e;
            int i;
            // 从 staleSlot 向后，直至遇到 null
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    //如果 key 的哈希值和当前下标 i 不一致
                    if (h != i) {
                        tab[i] = null;
                        //从 h 下标往后尝试找一个新的位置放 entry
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

这个方法是用来清除 staleSlot 位置的「不新鲜」entry，但是清理完之后没有停止而是继续向后直至 table[i]==null，在向后遍历的时候也会进行一定的 rehash。

再往后就到了 cleanSomeSlots 方法：

##### cleanSomeSlots
```java
private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
```

它会以 expungeStaleEntry 返回的 i 为起始点，向后 log2(n)，看还有没有不新鲜的节点。
* 如果没有，直接返回。
* 如果有删除不新鲜节点，然后再进行 log2(n) 次尝试。

这是一次快速的尝试，它在不清理存在垃圾和遍历所有节点插入时间复杂度高「O(n)」之间做了一点平衡。能清理到部分垃圾，但是时间复杂度不是很高。

通过以上方法，还是不能完全的规避内存泄漏，这就需要我们使用完 ThreadLocal 的时候主动去调它的 remove。在 remove 方法里同样会进行一些「不新鲜」清理。
