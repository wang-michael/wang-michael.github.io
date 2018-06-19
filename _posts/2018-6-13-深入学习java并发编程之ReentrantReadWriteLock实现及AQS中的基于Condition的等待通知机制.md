---
layout:     post
title:      深入学习java并发编程之ReentrantReadWriteLock实现及AQS中的基于Condition的等待通知机制
date:       2018-6-13
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:    
    - 并发
    - JDK源码阅读
---
>本文接上篇对于AQS类分析的博客，继续分析ReentrantReadWriteLock的实现及AQS中的基于Condition的等待通知机制。  

_ _ _
### **ReentrantReadWriteLock实现分析**
ReentrantReadWriteLock顾名思义，兼具有ReentrantLock的特性(可重入、公平性选择)和ReadWriteLock的特性(读写分离)，在此基础之上还添加了锁降级、获取内部工作状态等特性及方法，接下来看下其API文档中的介绍：  

ReadWriteLock API文档：  
```java
读写锁允许访问共享数据的并发性水平高于互斥锁允许的水平。 它利用了这样一个事实，即一次只有一个线程（一个编写器线程）可以修改共享数据，在很多情况下，任何数量的线程都可以同时读取数据（因此读取器线程）。 理论上，使用读写锁定所允许的并发性增加将导致性能提高超过使用互斥锁。 实际上，这种并发性的增加只能在多处理器上完全实现，并且只有在共享数据的访问模式合适时才会如此。

读写锁是否会提高使用互斥锁的性能取决于数据读取的频率与被修改的频率，读取和写入操作的持续时间以及数据的争用——也就是，尝试同时读取或写入数据的线程数。例如，一个最初用数据填充并且此后不经常修改，而频繁搜索（例如某种目录）的集合是使用读写锁定的理想候选。然而，如果更新变得频繁，那么数据的大部分时间都被锁定，并且几乎没有并发的增加。此外，如果读取操作太短，则读写锁定实现（本质上比互斥锁定更复杂）的开销占据了执行成本的大部分，特别是因为许多读写锁定实现仍然通过小部分代码互斥执行。最终，只有分析和测量才能确定使用读写锁是否适合您的应用程序。  

实现时需要考虑的几个问题：  
1. 在写锁释放锁定时，如果读锁和写锁都在等待，是把锁给读锁还是写锁呢？偏向写锁的情况很常见，因为写操作通常很短且很少发生；偏向读操作则并不常见，因为读操作如果频繁且耗时，会导致写操作的长时间延迟。当然公平的实现也是有可能的。
2. 当读锁处于进行状态时，有写锁和读锁请求正在等待，先给谁呢？偏向读操作可能会导致写操作的长时间延迟，偏向写操作会降低并发量。  
3. 确定这些锁是否是可重入的：一个带有写入锁的线程是否可以重新获取此写锁？ 可以在保持写锁的同时获取读锁吗？ 读锁本身是否可重入？--->都可以的。  
4. 写锁降级为读锁的时候是否不允许其他写锁参与？---->是的。 一个读锁能否升级为写锁而不是偏向于其他的读锁或写锁？---->不能
```
这里我们需要重点注意的是ReentrantReadWriteLock在实现ReadWriteLock时为上面几个问题采用了何种解决方案。 

ReentrantReadWriteLock的API文档:
```java
ReentrantReadWriteLock类具有以下特性：
1. 不支持强制定制读写偏好，但支持公平性与非公平性定制。 当构造为不公平（缺省）时，读写锁的输入顺序未指定，但受重入限制的约束。 持续争用的非公平锁可以无限期地推迟一个或多个读或写线程，但通常具有比公平锁更高的吞吐量。  当构建为公平时，线程使用到达顺序策略竞争进入。 当前持有的锁被释放时，等待时间最长的单写入器线程将被分配写入锁，或者如果有一组读取器线程等待的时间比所有等待的写入器线程长，则该组将被分配读取锁。

2. 可重入

3. 锁降级

4. 锁获取期间可中断

5. Condition support：只有写锁支持Condition，读锁不支持。
   
6. 检测工具支持：比如通过getReadLockCount等方法监测其内部工作状态。   

ReentrantReadWriteLock类可用于提高某些集合的某种用途的并发性。这这种情况通常具备以下条件：集合很大、被读线程访问的次数更多、对集合进行某些操作的开销比同步的开销大。比如一个使用了TreeMap的类，该TreeMap预计会很大并且会同时被访问。  

```
下面是一个使用HashMap实现的Cache，通过ReentrantReadWriteLock保证其读写的线程安全性：  
```java
public class Cache {
	static Map&lt;String, Object&gt; map = new HashMap&lt;String, Object&gt;();
	static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
	static Lock r = rwl.readLock();
	static Lock w = rwl.writeLock();
	// 获取一个key对应的value
	public static final Object get(String key) {
		r.lock();
		try {
			return map.get(key);
		} finally {
			r.unlock();
		}
	}
	// 设置key对应的value，并返回旧有的value
	public static final Object put(String key, Object value) {
		w.lock();
		try {
			return map.put(key, value);
		} finally {
			w.unlock();
		}
	}
	// 清空所有的内容
	public static final void clear() {
		w.lock();
		try {
			map.clear();
		} finally {
			w.unlock();
		}
	}
}
```
ReentrantReadWriteLock也是基于AQS类实现读锁写锁实现读写分离的，我们知道在AQS类中定义了一个状态变量state，通过CAS操作改变这个状态的值来区分同步状态，在ReentrantLock实现中锁只有独占锁一种状态，我们使用state记录锁被一个线程重复获取的次数，在ReentrantReadWriteLock中我们既要记录读锁被获取的次数，也要记录写锁被同一个线程获取的次数，还要对读锁、写锁状态进行区分，锁有两种状态，那么应该如何使用这个state变量呢实现这些功能呢？   

要想在一个整形变量上维护多种状态，就一定需要按位切割使用这个变量，读写锁将变量切分成了两个部分，高16位表示读，低16位表示写，如下图：     
<img src="/img/2018-6-13/ReadWriteDivision.png" width="650" height="650" alt="读写锁状态划分示意图" />  
<center>图1：读写锁状态划分示意图</center>

上图表示一个线程已经获取了写锁，且重进入了两次(共进入三次)，在此过程中也获取了两次读锁。读写锁是如何迅速的确定读和写各自的状态呢？答案是通过位运算。假设当前同步状态值为S，写状态等于 S & 0x0000FFFF（将高16位全部抹去），读状态等于 S >>> 16（无符号补0右移16位）。当写状态增加1时，等于S + 1，当读状态增加1时，等于S + (1 << 16)，也就是S + 0x00010000。  

根据状态的划分能得出一个推论：S不等于0时，当写状态(S & 0x0000FFFF)等于0时，则读状态(S >>> 16)大于0，即读锁已被获取。  
 
接下来来分析下ReentrantReadWriteLock的源码。  

#### **写锁的获取与释放**
写锁是一个支持重进入的排它锁，获取写锁的代码如下:  
```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState(); // 获取当前状态
    int w = exclusiveCount(c); // 返回写锁被获取的次数
    if (c != 0) { // 证明存在锁，但不确定是读锁还是写锁
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // 注意这里的逻辑：一个获取到读锁的线程是不能再获得写锁的
        if (w == 0 || current != getExclusiveOwnerThread()) // 当前存在的是读锁或者获取到写锁的线程不是当前线程
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT) // 写锁被获取的次数已经到达最大(65535)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire 
        // 这里一定是单线程执行的，无需CAS操作
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}

// 非公平锁实现的writerShouldBlock方法
static final class NonfairSync extends Sync {

    final boolean writerShouldBlock() { // 偏向写锁，不管同步队列中有没有线程，新来的线程在没有锁的时候都可以尝试下获取写锁
        return false; // writers can always barge
    }    
}

// 公平锁实现的writerShouldBlock方法
static final class FairSync extends Sync {

    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    } 
}

// 新来的线程和被唤醒的线程尝试获取同步状态时都会调用此方法
public final boolean hasQueuedPredecessors() {  
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    /**
     * 当前节点含有前驱节点，需要同时满足以下条件：  
     *   1、同步队列不为空，即head ！= tail;
     *   2、当前队列仅含有头结点(即(s = h.next) == null,这个条件用来判断当前线程是不是同步队列之外的线程)或者同步队列的头结点的直接后继节点不是当前节点(用来判断被唤醒的线程获取同步状态时是否存在前驱节点)
     */
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
释放写锁代码分析如下：  
```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}

protected final boolean isHeldExclusively() {
    // While we must in general read state before owner,
    // we don't need to do so to check if current thread is owner
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```
写锁释放的过程比较简单，每次释放均减少写状态，当写状态为0时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。
```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```  
  
#### **读锁的获取与释放**
获取源码分析如下：  
```java
// 这个类用来维护每个线程获取读锁的次数
static final class HoldCounter {
    int count = 0;

    // Use id, not reference, to avoid garbage retention
    // 这里使用tid的原因是避免垃圾留存(每个线程持有一个ThreadLocal<HoldCounter>变量——>持有一个HoldCounter变量)，如果
    // HolderCounter中含有Thread引用自身，就变为了线程含有自身的引用，GC可以回收自引用对象吗？？？
    // 使用内存模型提供的final语义避免long形变量出现无中生有现象
    final long tid = getThreadId(Thread.currentThread());
}

// 缓存最后一个获取读锁成功的线程到目前为止获取的读锁的数量
private transient HoldCounter cachedHoldCounter; 

static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
   
    // 设置默认的ThreadLocal初始值，每个线程都有自己的HoldCounter
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}

private transient ThreadLocalHoldCounter readHolds;// ThreadLocal用来维护每个线程获取读锁的次数

/**
 * firstReader是第一个获取读锁的线程，firstReaderHoldCount是第一个获取读锁的线程持有的读锁的数量
 * 更精确的来说，firstReader是唯一的线程——>最后一个把shared count由0变1的线程，并且到目前为止没有释放锁，如果不存在这样的线程则此字段为
 * null。
 * A线程获取读锁，将sharedCount由0变1，之后B获取读锁由1变2，之后A线程释放读锁由2变1，firstReader由A变为B还是变为null ???
 * 一般不会带来垃圾留存，这个字段会在tryReleaseShared字段中被设置为null
 * 这允许跟踪无竞争的读锁持有的读锁数量的操作非常廉价
 */
private transient Thread firstReader = null;
private transient int firstReaderHoldCount;

Sync() {
    readHolds = new ThreadLocalHoldCounter();
    setState(getState()); // ensures visibility of readHolds
}

protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current) // 当前存在写锁且占用写锁的线程非当前线程
        return -1; 
    int r = sharedCount(c); // 返回当前的共享锁的数量
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) { // 如果获取读锁成功
        if (r == 0) { // 我是第一个获取读锁的
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // 之前我是第一个获取读锁的，本次是重新获取锁
            firstReaderHoldCount++;
        } else { // 我不是第一个获取读锁的
            HoldCounter rh = cachedHoldCounter; // 尝试缓存当前线程获取的读锁的次数
            if (rh == null || rh.tid != getThreadId(current)) // 之前缓存的数量不是当前线程的，替换为当前线程
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0) // 之前缓存的数量是我的，但是为0，为什么会出现这种情况呢？？ 
                readHolds.set(rh);
            rh.count++; // 增加获取的读锁的数量  
        }
        return 1;
    }
   
    return fullTryAcquireShared(current);
}

/**
 * Full version of acquire for reads, that handles CAS misses
 * and reentrant reads not dealt with in tryAcquireShared.
 * tryAcquireShared方法的补充，用来处理获取读锁线程应该被阻塞、读锁数量已经到了最大、CAS获取读锁失败三种情况
 */
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) { // 当前写线程为0并且读线程需要阻塞
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) { // 当前线程为第一个线程，后继节点等待的线程为写线程，也会执行到这里
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {  // 我不是最后一个线程
                        rh = readHolds.get();
                        if (rh.count == 0) // 在后继节点有写线程等待的时候不接受新到来的读线程的请求
                            readHolds.remove();
                    }
                }
                if (rh.count == 0) // 当前线程是新到来的想要获取读锁的线程，必须等待，防止写线程无限饥饿
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT) // 处理读锁数量已经到了最大情况
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) { // 处理CAS获取读锁失败的情况
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else { 
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}

// 非公平锁实现的readerShouldBlock方法
static final class NonfairSync extends Sync {

    final boolean readerShouldBlock() {
        /* As a heuristic to avoid indefinite writer starvation,
         * block if the thread that momentarily appears to be head
         * of queue, if one exists, is a waiting writer.  This is
         * only a probabilistic effect since a new reader will not
         * block if there is a waiting writer behind other enabled
         * readers that have not yet drained from the queue.
         * 调用这个方法为了避免无限期的写锁请求饥饿。当同步队列中等待的头部的直接后继节点是写锁请求，就阻塞当前读锁获取的请求。 
         */
        return apparentlyFirstQueuedIsExclusive();
    }
}

// true代表当前tryAcquireShared应该被阻塞
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    // 如果当前头结点的直接后继节点是写锁请求，就阻塞当前读锁获取的请求
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}

// 公平锁实现的readerShouldBlock方法
static final class FairSync extends Sync {

    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```
**防止写饥饿策略：**在上面源码中很重要的一点是tryAcquireShared方法和fullTryAcquireShared结合使用，作用是当有多个读线程执行，且同步队列中头结点的直接后继节点是一个等待的写线程，那么在此写线程之后的新的读线程请求都不会再被接受，新到来的读请求会阻塞，之前执行的读线程可以重新获取读锁执行直到完毕。所有读线程执行完毕之后才是写线程执行，之后才是新来的读线程执行。   

读锁的释放：  
```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) { // 释放的线程是第一个Reader
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current)) // 缓存的不是我才进行查找
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count; // 执行之后rh.count >= 0;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc)) // 真正释放读锁的操作
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```  

#### **锁降级**
```java
 class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock(); // 先获取读锁
     if (!cacheValid) { // 之前缓存失效了
       // Must release read lock before acquiring write lock
       // 获取写锁之前需要先释放读锁
       rwl.readLock().unlock();
       // 获取写锁
       rwl.writeLock().lock();
       try {
         // Recheck state because another thread might have
         // acquired write lock and changed state before we did.
         // 我们释放读锁到获取写锁之间cache是有可能失效的
         if (!cacheValid) {
           data = ...
           cacheValid = true;
         }
         // 更新缓存之后获取读锁
         // Downgrade by acquiring read lock before releasing write lock
         rwl.readLock().lock();
       } finally {
         // 释放写锁
         rwl.writeLock().unlock(); // Unlock write, still hold read
       }
     }

     try {
       use(data);
     } finally {
       // 释放读锁
       rwl.readLock().unlock();
     }
   }
 }
```
在上面更新缓存的过程中涉及到了锁降级的过程，即获取写锁——>更新缓存——>获取读锁——>释放写锁——>读取数据;这就是所谓的锁降级。
  
锁降级中读锁的获取是否必要呢？是必要的。主要原因是保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，在释放读锁与重新获取写锁之间，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，则当前线程获取的缓存中的数据，无法感知线程T的数据更新。  

RentrantReadWriteLock不支持锁升级（把持读锁、获取写锁，最后释放读锁的过程）。原因也是保证数据可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程不可见。  

#### **简单总结**
经过上面的分析之后再来看下ReadWriteLock API文档中提到的实现ReadWriteLock时需要注意的几个问题，在ReentrantReadWriteLock实现中都可以看到其实现的方式：    
1. 在写锁释放锁定时，如果读锁和写锁都在等待，是把锁给读锁还是写锁呢？偏向写锁的情况很常见，因为写操作通常很短且很少发生；偏向读操作则并不常见，因为读操作如果频繁且耗时，会导致写操作的长时间延迟。当然公平的实现也是有可能的——>ReentrantReadWriteLock写锁释放时对于同步队列中等待的线程是按照顺序来的，不管读锁还是写锁，谁在前面谁先拿。但是对于写锁释放时新来的线程，如果是读锁请求且同步队列中头结点的后继节点为写请求，则偏向队列中的写请求，否则新来的读请求和同步队列中的读请求都可能获得读锁。    
2. 当读锁处于进行状态时，有写锁和读锁请求正在等待，先给谁呢？偏向读操作可能会导致写操作的长时间延迟，偏向写操作会降低并发量。——>如果同步队列中头结点直接后继节点是写请求，则不再接受新的读请求，直到之前读请求执行完毕后执行写请求。    
3. 确定这些锁是否是可重入的：一个带有写入锁的线程是否可以重新获取此写锁？ 可以在保持写锁的同时获取读锁吗？ 读锁本身是否可重入？--->都可以的。   
4. 写锁降级为读锁的时候是否不允许其他写锁参与？---->是的。 一个读锁能否升级为写锁而不是偏向于其他的读锁或写锁？---->不能  

_ _ _
### **基于Condition的等待/通知机制实现**  
AQS中除了同步队列，还有一个等待队列，用于实现基于AQS实例实现的等待/通知机制。一个同步组件对应的等待队列可能有多个，处于等待队列的线程在等待的condition对象上的事件发生后可能会移动到同步队列中去。等待队列是一个单向链表，它并不是必须的，只有当使用Condition时，才会存在此单向链表。等待队列示意图如图2：  
<img src="/img/2018-6-13/conditionQueue.png" width="650" height="650" alt="等待队列示意图" />  
<center>图2：等待队列示意图</center>  

先来看一个基于Condition的例子：  
```java
 class BoundedBuffer {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = lock.newCondition(); 
   final Condition notEmpty = lock.newCondition(); 

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(Object x) throws InterruptedException {
     lock.lock();
     try {
       while (count == items.length)
         notFull.await();
       items[putptr] = x;
       if (++putptr == items.length) putptr = 0;
       ++count;
       notEmpty.signal();
     } finally {
       lock.unlock();
     }
   }

   public Object take() throws InterruptedException {
     lock.lock();
     try {
       while (count == 0)
         notEmpty.await();
       Object x = items[takeptr];
       if (++takeptr == items.length) takeptr = 0;
       --count;
       notFull.signal();
       return x;
     } finally {
       lock.unlock();
     }
   }
 }
```
上面是一个基于Condition实现的有界缓冲区，下面看看基于wait-notify实现同样的功能的例子，分析Condition带来的好处。  
```java
class BoundBuffer {

    final Object syncLock = new Object();
    final Object[] items = new Object[100];
    int putptr, takeptr, count;

    public void putWithSync(Object x) throws InterruptedException {
        synchronized (syncLock) {
            while (count == items.length)
                syncLock.wait();
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            syncLock.notify();
        }
    }

    public Object takeWithSync() throws InterruptedException {
        synchronized (syncLock) {
            while (count == 0)
                syncLock.wait();
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            syncLock.notify(); // 这里唤醒的线程有可能是执行put操作的线程，也可能是take操作的线程
            return x;
        }
    }
}
```
由上面两例对比分析可知使用Condition带来的好处是基于同一个Lock，将不同功能的线程放到了不同的wait-set，唤醒时可指定唤醒某种功能的线程。  

Condition接口与Object的wait-notify对比图如下:  
<img src="/img/2018-6-13/ConditionWaitCompare.png" width="650" height="650" alt="等待队列示意图" />  
<center>图3：wait-notify对比图</center>  

可见相比于wait-notify，增加了对于同一个锁上多个同步队列、wait过程中相应中断、指定超时时间的wait等特性。  

ReentrantLock里的Condition获取操作就是基于AQS内每个ConditionObject维护的一个ConditionQueue实现的，在java并发编程的艺术中其实现介绍的特别清晰，我觉除了书上说的东西，其他的没有什么需要自己特别注意的，这里就不赘述了。  


(完)  
《Java并发编程的艺术》     
[JDK1.8源码分析之ReentrantReadWriteLock（七）](https://www.cnblogs.com/leesf456/p/5419132.html)  
