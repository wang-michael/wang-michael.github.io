---
layout:     post
title:      深入学习java并发编程之ThreadLocal
date:       2018-4-1
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:    
    - 并发
    - JDK源码阅读
---
>本文记录了自己对于ThreadLocal类源码的分析过程，仅用于个人备忘，如有错误，敬请指出。  

_ _ _
### **前言**  
我们都知道ThreadLocal的作用是生成一个仅当前线程可使用的线程内局部变量，类似于下面这样：  
```java
public class ThreadLocalTest {

    private static final ThreadLocal<Long> TIME_LOCAL = new ThreadLocal<Long>() {
        // 设置初始值，若之前未set过，使用get方法第一次获取时获取的是这个值
        @Override
        protected Long initialValue() {
            return System.currentTimeMillis();
        }
    };

    private static void begin() {
        TIME_LOCAL.set(System.currentTimeMillis());
    }

    private static Long end() {
        return System.currentTimeMillis() - TIME_LOCAL.get();
    }

    public static void main(String[] args) throws InterruptedException {
        begin();
        Thread.sleep(1000);
        System.out.println(Thread.currentThread().getName() + " " + end());
        new Thread(new Runnable() {
            @Override
            public void run() {
                begin();
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + " " + end());
            }
        }).start();
    }
}
```
上面的程序就是一个使用ThreadLocal变量记录每个线程的运行时间的例子。

我们可以猜想一下ThreadLocal内部的实现原理，我认为可有下面两种情况：
* 通过并发控制实现ThreadLocal，即每个ThreadLocal变量对应于一个类似于ConcurrentHashMap的数据结构，key值为ThreadID，value为线程局部变量。
* 通过在Thread中设置成员变量存储ThreadLocal变量，无需并发控制。

下面就通过分析ThreadLocal源码来解决上述问题。    

_ _ _
### **ThreadLocal源码分析**
从它的set方法开始看起：  
```java
public class ThreadLocal<T> {
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        // 可以看到set方法就是向每个线程对应的ThreadLocalMap中存储相应的线程局部变量
        if (map != null)
	        map.set(this, value);
        else
	        createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    
}
public class ThreadLocal<T> {
    ...
    // 每个线程确实有用来存储ThreadLocal变量的数据结构称为ThreadLocalMap
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
}
```
从上面的代码分析中可以看出ThreadLocal内部实现是通过在Thread中设置成员变量存储ThreadLocal变量。下面以set方法为例分析下ThreadLocalMap的具体实现。    
```java
public class ThreadLocal<T> {
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                // hash直接命中，找到了自己所在的slot
                if (k == key) {
                    e.value = value;
                    return;
                }

                // 当前找到的key之前被hash到过，但是之后其对应的ThreadLocal的key由于GC
                // 被回收(key为weakReference)
                if (k == null) { // 出现过期数据  
                    // 遍历清洗过期数据并在index处插入新数据，其他数据后移  
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            // sz代表当前存储的k-v的对数，超过threshold就要进行rehash,threshold为size的2/3
            int sz = ++size;            
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        // hash方式为线性探测法
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            // 令slotToExpunge指向前一个过期数据
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            /**
             * 从当前的过期数据所在位置开始向后找，有两种情况可以跳出for循环
             * 1、找到当前对应的key
             * 2、当前key并没有被存储过或者之前的存储已经失效，tab[i] = null
             */
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (k == key) {
                    e.value = value;

                    // 交换之后tab[i]中存储的key值为null
                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
        /**
         * Expunge a stale entry by rehashing any possibly colliding entries
         * lying between staleSlot and the next null slot.  This also expunges
         * any other stale entries encountered before the trailing null.  See
         * Knuth, Section 6.4
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         * 具体清除算法实现有待之后分析
         */
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
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
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
    }
}
```
在上述代码中可以看见ThreadLocalMap内部使用的Entry数组中的Entry内部类key值为WeakReference<ThreadLocal<?>>，value值为我们设置的ThreadLocal变量的值；将ThreadLocal<?>使用WeakReference进行封装的目的是当外界代码中不存在对ThreadLocal<?>的强引用时，GC可以直接回收此ThreadLocal<?>变量。  

ThreadLocalMap中会产生一些旧的、无用的键值对，产生方式可能是由于GC回收了key值ThreadLocal<?>,这时ThreadMap中Entry数组中此key对应的空间key值为null，但是value并不为null；这些无用键值对如果一直不将其移除可能导致ThreadLocalMap频繁进行rehash操作，但是其中存储了大量的无用数据，这些无用数据key值为null，但其对应的value值不为null，浪费内存空间。  

为了清除这些无用键值对，ThreadLocalMap中提供了expungeStaleEntry方法，清理无用键值对的方法都是直接或间接的调用这个方法来实现的。ThreadLocal类给程序员提供的API有get，set，remove；当使用get、set操作直接命中时并不会清理无用键值对，只有未命中时可能会尝试清理无用键值对；我们可以使用remove方法主动清理使用不到的ThreadLocal变量，释放内存空间；(待确认)remove方法不仅会清理我们指定的键值对，它的边际效应是清理其它无用的键值对。    

我们经常使用ThreadLocal变量的方式是private static ThreadLocal ... , 如果一直不将这个static变量置为null的话，在外界就一直存在对此ThreadLocal变量的强引用，这个ThreadLocal变量就不会被GC回收，在各个线程中就一直会占用内存空间，这时就需要我们在不需要使用某个ThreadLocal变量的时候主动使用remove方法释放其占用的内存空间，对于使用线程池的情况下尤其需要注意这一点(由于线程池的线程一般会复用，Thread不结束,其对应的ThreadLocalMap占用空间不会被回收，更需要我们主动去remove)。

_ _ _
### **API文档翻译**
这个类提供线程局部变量。 这些变量不同于它们的正常副本，因为访问一个线程的每个线程（通过它的get或set方法）都有其自己的，独立初始化的变量副本。 ThreadLocal实例通常是希望将状态与线程关联的类中的私有静态(private static)字段（例如用户ID或事务ID）。  

例如，下面的类为每个线程生成本地唯一的标识符。 线程的ID在第一次调用ThreadId.get（）时被分配，并在随后的调用中保持不变。
```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadId {
 // Atomic integer containing the next thread ID to be assigned
 private static final AtomicInteger nextId = new AtomicInteger(0);

 // Thread local variable containing each thread's ID
 private static final ThreadLocal<Integer> threadId =
     new ThreadLocal<Integer>() {
         @Override protected Integer initialValue() {
             return nextId.getAndIncrement();
     }
 };

 // Returns the current thread's unique ID, assigning it if necessary
 public static int get() {
     return threadId.get();
 }
}
```
只要线程处于活动状态并且ThreadLocal实例可以访问，每个线程就拥有对其线程局部变量副本的隐式引用; 线程消失后，线程本地实例的所有副本都将受垃圾回收处理（除非存在对这些副本的其他引用）。

_ _ _
### **InheritableThreadLocal**
```java
private static void testInheritableThreadLocal() throws InterruptedException {
    // 由此线程创建的子线程会继承当前线程中的ThreadLocal变量
    final ThreadLocal threadLocal = new InheritableThreadLocal();
    threadLocal.set("droidyue.com");
    Thread t = new Thread() {
        @Override
        public void run() {
            System.out.println("child get: " + threadLocal.get());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("child get: " + threadLocal.get());
        }
    };
    t.start();

    Thread.sleep(500);
    threadLocal.set("michael-wang");
    System.out.println("parent set : michael-wang");
    System.out.println("parent get : " + threadLocal.get());
}
```

(完)
参考文章：[并发编程 | ThreadLocal源码深入分析](http://www.sczyh30.com/posts/Java/java-concurrent-threadlocal/)