---
layout:     post
title:      深入学习java并发编程之AbstractQueuedSynchronizer
date:       2018-6-11
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:    
    - 并发
    - JDK源码阅读
---
>本文主要记录了自己对于AbstractQueuedSynchronizer类的学习过程，文中内容参考了java并发编程的艺术一书及网上的一些博客，在整理的过程中加入了自己的理解，主要用于备忘，如有错误，敬请指出。  

_ _ _
### **前言**
AbstractQueuedSynchronizer是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。只有掌握了同步器的工作原理才能更加深入的理解并发包中的其它并发组件。并发包的作者希望AbstractQueuedSynchronizer成为实现大部分同步需求的基础。所以深入学习理解AbstractQueuedSynchronizer类的实现还是很有必要的。  

我们使用同步器的主要方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态，子类被推荐作为自定义同步组件的静态内部类来帮助实现我们自定义同步组件的相关功能。  

同步器的设计是基于模板方法模式的，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中并调用同步器提供的模板方法，这些模板方法将会调用使用者重写的方法。  

具体调用顺序为：自定义同步组件---->调用同步器提供的模板方法(比如acquire)---->调用使用者重写的方法(比如tryAcquire)---->调用getState、setState、compareAndSetState修改同步状态。  

<img src="/img/2018-6-11/templateMethod.jpg" width="650" height="650" alt="AQS模板方法图" />  
<center>图1：AQS模板方法图</center> 

上图分为左右两个大矩形框，左边代表的独占访问的操作，右边代表着共享访问的操作，其中的小矩形框都代表的AQS里面的方法。其中，蓝色的小矩形框代表着AQS中默认已经实现了的方法——即模板方法，而红色小圆角矩形框代表着需要你自己去实现并覆盖的方法。箭头表示方法的调用次序(比如acquire调用tryAcquire)。从方法的名称上，就可以大概明白方法的作用，acquire用来表示获取资源数的操作，而release表示用来释放资源数的操作，不带Shared表示是独占的操作。如果我们没有实现红色圆角矩形框的方法却间接调用了，将会抛出UnsupportedOperationException异常。  

下面是一个使用AbstractQueuedSynchronizer实现的一个自定义的独占锁的示例:  
```java
 class Mutex implements Lock, java.io.Serializable {

   // 定义AbstractQueuedSynchronizer子类帮助我们实现自定义同步组件
   private static class Sync extends AbstractQueuedSynchronizer {
     // 判断锁是否处于占用状态
     protected boolean isHeldExclusively() {
       return getState() == 1;
     }

     // 当状态为0的时候获取锁
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }

     // 释放锁 将状态设置为0
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }

     // 返回一个Condition，每个Condition都包含等待在其上的一个condition等待队列
     Condition newCondition() { return new ConditionObject(); } 
   }

   // sync对象做了所有困难的工作，我们只需要将操作代理到sync上即可
   private final Sync sync = new Sync();

   public void lock()                { sync.acquire(1); }
   public boolean tryLock()          { return sync.tryAcquire(1); }
   public void unlock()              { sync.release(1); }
   public Condition newCondition()   { return sync.newCondition(); }
   public boolean isLocked()         { return sync.isHeldExclusively(); }
   public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
 }
```
我们先不需要弄清楚上面代码的具体含义，只需要看见我们自定义的同步组件对外提供的API(即public方法)都是通过AQS子类提供的模板方法实现的，我们只不过重写了AQS类规定的我们可以重写的几个方法定制了这些模板，实现了我们的自定义的功能。  

下面就来深入分析一下AQS提供给我们的模板方法。  

_ _ _
### **AQS模板方法分析**
上面已经说过，AQS模板方法帮我们把线程间同步状态的管理封装成了一些模板操作，并且给我们提供了5个可重写的方法供我们在模板方法的基础上完成自定义同步组件的功能。  
<img src="/img/2018-6-11/AQStemplatemethods.png" width="650" height="650" alt="AQS模板方法图" />  
<center>图2：同步器提供的模板方法</center> 

图2是同步器提供给我们的模板方法。  

<img src="/img/2018-6-11/AQSRewriteMethods.png" width="650" height="650" alt="AQS重写方法图" />  
<center>图3：同步器提供的需要重写的方法</center>   
  
图3是同步器提供给我们需要重写的方法。  

以模板方法acquire(int arg)为例，其功能是以独占式的方式获取同步状态，并在获取的过程中忽略中断。其内部实现是调用至少一次tryAcquire(int) 方法，如果调用acquire(int arg)方法的线程入队，就会重复的阻塞和被唤醒，直到成功获取同步状态。  

那么acquire(int arg)是如何实现上述功能的呢？这就要涉及到AQS内部的核心数据结构————同步队列的实现。  

#### **AQS同步队列介绍**  
AQS类中的内部类Node描述了用来构成同步队列(一个FIFO双向队列)的基础节点的属性类型与名称以及描述，如下表:  
<img src="/img/2018-6-11/AQSNode.png" width="650" height="650" alt="AQS同步队列基础节点Node" />  
<center>图4：AQS同步队列基础节点Node属性类型与名称及描述</center>  

其在AQS中源码如下:  
```java
  static final class Node {
        /** 表明Node节点正处于共享模式 */
        static final Node SHARED = new Node();
        /** 表明一个节点正处于独占模式 */
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;
        
        volatile int waitStatus;
        
        volatile Node prev;
        
        volatile Node next;
       
        volatile Thread thread;
        
        Node nextWaiter;
        
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```   
AQS类中持有同步队列的首节点(head成员变量)与尾节点(tail成员变量),同步队列的基本结构如图5:  
<img src="/img/2018-6-11/AQSQueueBaseStructure.png" width="500" height="500" alt="同步队列基本结构" />  
<center>图5：同步队列基本结构</center>  

没有成功获取同步状态的线程将会成为节点加入该队列的尾部，可能同时有多个线程同时尝试此操作，所以入队的时候需要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法:compareAndSetTail(Node expect, Node update),只有CAS操作操作成功后，当前节点才正式与之前的尾节点建立关联。入队示意图如图6所示：  
<img src="/img/2018-6-11/enqueue.png" width="500" height="500" alt="节点入队示意图" />  
<center>图6：节点入队示意图</center>  

同步队列遵循FIFO，首节点是获取同步状态成功的节点，首节点的线程在释放同步状态(调用release方法)时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点，示意图如图7所示：      
<img src="/img/2018-6-11/outQueue.png" width="500" height="500" alt="节点出队示意图" />  
<center>图7：节点出队示意图</center>

前一个首节点释放同步状态后，同步队列中只会有一个线程能够成功获取到同步状态,因此设置头结点的方法并不需要使用CAS来保证，当同步队列中的被首节点唤醒的节点获取同步状态成功时，它只需要将首节点设置为自己并断开原首节点的next引用即可。    

介绍完了AQS中使用的核心数据结构同步队列之后，就来看其在具体实现中的应用，从acquire(int arg)方法开始看起。

#### **acquire(int arg)源码分析**
```java
public final void acquire(int arg) { 
    // 新来的线程可以先尝试一下获取同步状态，失败后再将其加入同步队列，非公平
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```  
可以看到if语句中首先调用我们重写的tryAcquire(arg)方法尝试获取锁，获取失败的话调用addWaiter方法把当前线程包装成一个Node节点加入同步队列中并不断尝试获取锁直到成功为止。  

**需要注意的是这里获取同步状态的过程是非公平的，即使当前同步队列中已经存在很多尝试获取同步状态的线程，同步状态还是有可能被一个新来的线程获取到。**  

接下来看下addwaiter(Node mode)方法：  
```java
private Node addWaiter(Node mode) {
    // 将当前线程包装成Node节点
    Node node = new Node(Thread.currentThread(), mode);
    // 假设执行addWaiter方法的线程只有自己或者很少，先尝试快速添加，万一添加成功了呢
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 当前队列为空或者上面的CAS添加操作失败，则调用enq方法继续尝试将当前节点加入同步队列
    enq(node);
    return node;
}

private Node enq(final Node node) {
    // 使用无限循环，保证将当前节点成功加入同步队列
    for (;;) {
        Node t = tail;
        if (t == null) { // 如果tail为null，则使用CAS操作创建node，注意这里初始化的Node与我们传入的node没有任何关系
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
``` 
介绍了将当前线程包装为Node节点加入同步队列的过程之后，我们来看下使用acquireQueued(final Node node, int arg)尝试获取同步状态的过程：  
```java
// 返回值代表获取同步状态的过程中线程是否有中断没有被处理
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 用来在线程被唤醒时记录其中断状态
        boolean interrupted = false;
       
        /**
         * 无限循环确保线程从阻塞中被唤醒时如果获取同步状态失败的话重新陷入阻塞。
         * 线程从阻塞中被唤醒可能有以下几种情况：  
         *    1、当前节点为头结点之后的最近的处于有效状态的节点，头结点释放同步状态之后将其唤醒。
         *    2、被中断唤醒
         *    3、前驱节点被cancelled之后唤醒当前节点，目的是在shouldParkAfterFailedAcquire中删除失效的前驱节点  
         */
        for (;;) {
            final Node p = node.predecessor();
            // 如果当前节点的前驱结点为头结点并且获取同步状态成功的话
            // 这里前驱结点为头结点获取同步状态还可能失败的原因是同步状态的获取是不公平的，可能被同步队列之外的新到来的线程获取到
            if (p == head && tryAcquire(arg)) {
                // 这里的代码一定是单线程执行的，同步队列中只会有一个线程能够成功获取到同步状态，新到来的线程会陷入阻塞状态
                // 设置头结点为自己
                setHead(node);
                // 设置之前头结点的next指针为null，帮助GC
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 判断一个线程在获取同步状态失败之后是否应该被阻塞
// 只有当该节点的前驱结点的状态为SIGNAL时，才可以对该结点所封装的线程进行park操作。否则，将不能进行park操作。
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 只有在当前节点的前驱节点的waitStatus为SIGNAL时，当前节点对应的线程才应该被阻塞
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) { // 前驱节点已经被CANCELLED，失效了
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         * 跳过所有失效的前驱节点，不要陷入阻塞，重新尝试获取锁
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /* 前驱节点的状态为-3(PROPAGATE)或0(初始状态),(为CONDITION -2时，表示此节点在condition queue中) 
         * 将前驱节点的状态变为Node.SIGNAL，不要陷入阻塞，重新尝试获取锁
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    // 阻塞当前线程，并设置blocker
    LockSupport.park(this);
    // 当前线程从阻塞中返回时记录并清除中断状态
    return Thread.interrupted();
}
```
到这里acquireQueued(final Node node, int arg)中的主体代码已经介绍过了，最后介绍一下其finally块中的cancelAcquire(node)方法，这个方法仅在获取同步状态失败时执行：(什么时候acquireQueued方法获取同步状态会失败呢？？？)
```java
// 方法的整体目标是将cancel的Node从同步队列中删除  
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    // predNext为找到的有效的前驱的直接后继，有可能为当前节点，有可能不是
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;

    // 如果被cancelled的节点为尾节点，直接将其移除就好
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        // 接下来的if-else实现的目标是：如果接班节点需要signal，则先将pred的ws设置为SIGNAL，之后设置pred的next节点为当前节点的next
        // 否则唤醒当前节点的接班人以便于传播
        int ws;
        // pred不为头结点 && ((pred的waitStatus为SIGNAL) || (pred的ws为0或-3，则将其更新为SIGNAL)) && pred.thread不为null
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            // 则设置pred的next节点为当前节点的next
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);// 这个CAS操作如果失败的话当前被cancelled的节点会在其后继节点执行shouldParkAfterFailedAcquire(Node pred, Node node)方法时被删除
        } else {
            // 如果设置失败了，需要唤醒当前状态为Node.CANCELLED节点的接班人，作用是删除当前被cancelled的节点。  
            // 注意此时cancelled节点并没有从同步队列中删除，这种情况下cancelled节点是在shouldParkAfterFailedAcquire(Node pred, Node node)方法中被删除的
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```
接下来看下用来唤醒后继节点的unparkSuccessor(Node node)实现:  
```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0) // 状态值小于0，为SIGNAL -1 或 CONDITION -2 或 PROPAGATE -3
        compareAndSetWaitStatus(node, ws, 0); // 比较并且设置结点等待状态，设置为0，即初始状态

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     * 唤醒接班节点的线程，接班节点通常情况下就是下一个后继节点，但是如果后继节点被cancelled或者为null，则从尾部开始从后向前寻找第一个
     * 满足条件的节点。
     * 为什么不是从前向后寻找呢？？？是因为其后继节点可能为空吗？什么情况下会出现后继节点为空但仍然在同步队列中的情况呢？？？
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从后向前搜索满足条件的后继节点(非直接后继)
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒找到的接班节点对应的线程
        LockSupport.unpark(s.thread);
}
```
到此为止，acquire(int arg)的整个方法执行过程已经介绍完了，接下来看两张图更形象的认识一下其执行过程：  
<img src="/img/2018-6-11/CASSynState.png" width="650" height="650" alt="acquireQueued方法节点自旋获取同步状态图" />  
<center>图9：acquireQueued方法节点自旋获取同步状态图</center>  

图10展示了acquire(int arg)方法的调用流程：  
<img src="/img/2018-6-11/acquireProcess.png" width="650" height="650" alt="acquire(int arg)方法调用流程" />  
<center>图10：acquire(int arg)方法调用流程</center>  

最后对acquire(int arg)方法的执行过程做一个简单总结：   
1. 新来的线程先尝试一下获取同步状态，获取失败则加入到同步队列中并在队列中进行自旋。   
2. 当前节点被唤醒后重新尝试自旋获取锁，获取失败后根据前驱节点的状态选择陷入阻塞或者尝试重新获取。    

acquireInterruptibly方法与tryAcquireNanos相比于acquire方法不同之处仅在于增加了中断和超时相应操作，主题代码逻辑完全相同，下面不再赘述。  

#### **release(int arg)源码分析**
源码如下：  
```java
public final boolean release(int arg) {
    // 先调用我们重定义的tryRelease方法释放同步状态
    if (tryRelease(arg)) {
        // 释放同步状态的节点一定是同步队列的头结点
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h); // 唤醒头结点的后继节点
        return true;
    }
    return false;
}
```
在release方法中需要注意的一点是在tryRelease(arg)执行完毕释放同步状态成功之后到unparkSuccessor唤醒后继节点的过程中，同步状态可能被新来的线程获取到。unparkSuccessor方法源码上面已经分析过，此处不再赘述。  

#### **acquireShared(int arg)源码分析**
从方法名称中就可以看出与acquire(int arg)方法的不同之处在于尝试以共享的方式访问资源，即获取同步状态，首先来看下共享式获取与独占式获取的对比： 
<img src="/img/2018-6-11/acquireShared.png" width="650" height="650" alt="共享式访问与独占式访问对比" />  
<center>图11：共享式访问与独占式访问对比</center>   

在图11中，左半部分，共享式访问资源时，其它共享式的访问均被允许，而独占式访问被阻塞，右半部分是独占式访问资源时，同一时刻其它访问均被阻塞。  

下面来分析acquireShared(int arg)方法源码：
```java
public final void acquireShared(int arg) {
    /**
     * tryAcquireShared方法返回值有三种情况:  
     * 负数代表失败，0代表当前以共享模式获取锁成功，但是后续的以共享模式获取锁的操作一定不能成功；正数代表当前以共享模式获取锁的线程获取成功并且之后的* 线程的共享模式获取操作也有可能成功。 
     * 
     * 新来的线程直接尝试以共享模式获取同步状态，有下面几种情况可能导致获取失败：  
     *   1、当前同步状态被独占式访问
     *   2、可共享式访问同步状态的线程数量是有限的，并且已经到达了最大数量
     */
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
private void doAcquireShared(int arg) {
    // 注意这里addWaiter的方式，处于共享模式的节点Node中的nextWaiter属性一定为Node.SHARED，独占式为null
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 无限循环确保一定可以获取到同步状态
        for (;;) {
            final Node p = node.predecessor();
            // 如果前驱节点为头节点
            if (p == head) {
                int r = tryAcquireShared(arg);
                // 以共享模式获取同步状态成功
                if (r >= 0) {
                    // 将当前节点设置为头节点并且向后传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted) // 处理中断
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     * 独占锁与共享锁不一样的地方的体现：
     *   某个节点被设置为head之后，如果它的后继节点是SHARED状态的，那么将继续通过doReleaseShared方法尝试往后唤醒节点，实现了共享状态的向后传播。
     *   doReleaseShared方法中可能出现前一个头结点的状态为PROPAGATE(-3)的情况   
     */
    // 为什么之前的头结点为null或者ws<0或者现在的头结点为null或者ws<0，就要将状态向后传播呢？？？
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
**待解决：**在setHeadAndPropagate方法中的if条件判断中，propagate > 0需要将同步状态向后传播容易理解，但为什么之前的头结点为null或者ws<0或者现在的头结点为null或者ws<0，也要将状态向后传播呢？？？  

acquireSharedInterruptibly方法与tryAcquireSharedNanos相比于acquireShared方法不同之处仅在于增加了中断和超时相应操作，主题代码逻辑完全相同，下面不再赘述。

#### **releaseShared(int arg)源码分析**
源码如下:  
```java
public final boolean releaseShared(int arg) {
    // 释放同步状态的操作可能同时来自多个线程，所以需要使用循环+CAS操作来确保同步状态安全释放
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
// 可能会在releaseShared方法或者acquireShared方法中的setHeadAndPropagate中被调用
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
             // 出现这种情况是由于当前节点后面的阻塞节点将其状态设置为了SIGNAL，代表告诉当前节点你释放同步状态时通知我一下
            if (ws == Node.SIGNAL) { 
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 这个else if应该是获取同步状态时由setHeadAndPropagate调用进来的，对于同一个共享节点，其状态变化猜测应该是PROPAGATE总先于SIGNAL
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // 只有在这里有可能将节点状态设置为PROPAGATE
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
主要是两层逻辑：

1. 头结点本身的waitStatus是SIGNAL且能通过CAS算法将头结点的waitStatus从SIGNAL设置为0，唤醒头结点的后继节点。  
2. 头结点本身的waitStatus是0的话，尝试将其设置为PROPAGATE状态的，意味着共享状态可以向后传播。传播方式是以一个节点唤醒其直接后继节点方式进行。  


到这里AQS类提供给我们的主要模板方法已经介绍完了，下面看一个使用AQS类实现的一个自定义同步组件TwinsLock的示例。  
    
_ _ _
### **自定义同步组件TwinsLock** 
TwinsLock目的是实现一个同一时刻至多允许两个线程同时访问的并发工具，超过两个线程的访问将被阻塞。分析其需求可知这是一个资源数为2的共享锁，使用AQS实现如下：  
```java
public class TwinsLock implements Lock {

    private final Sync sync = new Sync(2);

    private static final class Sync extends AbstractQueuedSynchronizer{
        Sync(int count){
            if(count <= 0){
                throw new IllegalArgumentException("count must larger than zero");
            }
            setState(count);
        }

        @Override
        public int tryAcquireShared(int reduceCount){
            for(;;){
                int current = getState();
                int newCount = current - reduceCount;
                if(newCount < 0 || compareAndSetState(current, newCount)){
                    return newCount;
                }
            }
        }

        @Override
        public boolean tryReleaseShared(int returnCount){
            for(;;){
                int current = getState();
                int newCount = current+returnCount;
                if(compareAndSetState(current, newCount)){
                    return true;
                }
            }
        }

    }
    
    @Override
    public void lock() {
        sync.acquireShared(1);
    }

    @Override
    public void unlock() {
        sync.tryReleaseShared(1);
    }
    
    // 其他接口方法略
}
```
TwinsLock测试类实现如下：  
```java
public class TwinsLockTest {
    public static void main(String[] args) {
        final Lock lock = new TwinsLock();
        class Worker extends Thread{
            public void run(){
                for(;;){
                    lock.lock();
                    try {
                        Thread.sleep(1000);
                        System.out.println(Thread.currentThread().getName());
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }finally{
                        lock.unlock();
                    }
                }
            }
        }

        for(int i=0;i<10;i++){
            Worker worker = new Worker();
            worker.setDaemon(true);
            worker.start();
        }

        for(int i=0;i<10;i++){
            try {
                Thread.sleep(1000);
                System.out.println("----");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

输出：
----
Thread-1
Thread-0
----
----
Thread-0
Thread-1
----
----
Thread-0
Thread-1
----
----
Thread-0
Thread-1
----
----
Thread-1
Thread-0
----
```
可见确实将最大并发数限制在了两个线程以内。如果将资源限制改为5，运行结果可能如下：  
```java
Thread-2
Thread-3
Thread-0
----
Thread-1
Thread-4
----
----
Thread-3
Thread-2
Thread-0
Thread-4
Thread-1
----
----
Thread-0
Thread-3
Thread-4
Thread-2
Thread-1
----
Thread-4
Thread-3
Thread-2
Thread-0
----
```  
由于并发的情况不同，同一时刻运行的线程数不同，但是都控制在了5以内。
  
_ _ _
### **ReentrantLock源码分析**
ReentrantLock锁是支持重进入的锁，它表示该锁能够支持一个线程对于资源的重复加锁(synchronized支持)，此外，该锁还支持获取锁时的公平性与非公平性选择(synchronized不支持)。  
    
#### **ReentrantLock如何实现对于资源重复加锁**
重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实现需要解决一下两个问题：  
1. 识别再次获取锁的线程是否为当前线程。
2. 锁被同一线程获取的次数统计，当计数为0时才代表锁被释放。 

以非公平性实现(默认)为例，获取同步状态的代码如下：  
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 尝试使用CAS操作获取锁并记录获取锁的线程为当前线程
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { // 如果之前获取锁的线程就是当前线程
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        // 记录锁被当前线程获取的次数
        setState(nextc);
        return true;
    }
    return false;
}
```
释放同步状态的代码如下：  
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread()) // 判断释放锁的线程是不是持有锁的线程
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 这里一定是单线程调用
    setState(c);
    return free;
}
```

#### **ReentrantLock如何实现锁的公平性选择**
如果一个锁是公平的，那么获取锁的顺序就应该符合请求的绝对时间顺序，也就是FIFO。ReentrantLock的默认实现是非公平的，那么哪里违背了FIFO原则呢？  

比如一个新来的线程并不在同步队列中，在同步队列头结点释放同步状态之后，同步状态可能被新到来的线程而不是同步队列中头结点之后的线程获取到，这就违反了FIFO原则。  

上面我们已经介绍过了非公平锁的获取方式，下面来分析下公平锁的获取方式：  
```java
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
可见上述方法与非公平的获取方式相比仅在于判断条件中多了hasQueuedPredecessors()方法，即加入了同步队列中是否有比当前节点更早到来的节点的判断，有的话当前线程就不能获取到锁。  

hasQueuedPredecessors()源码如下：  
```java
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
     *   2、当前队列仅含有头结点(即(s = h.next) == null)或者同步队列的头结点的直接后继节点不是当前节点
     */
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

对于一个非公平的ReentrantLock，它是非常有可能被同一个线程连续获得的，比如像下面这样
```java
private static class Job extends Thread {
    
    private Lock lock;
    public Job(Lock lock) {
        this.lock = lock;
    }
    public void run() {
        lock.lock();
        try {
            // 执行一些操作
        } finally {
            lock.unlock();
        }
        lock.lock(); // 这里是非常有可能被刚刚释放锁的线程获得的
        try {
            // 执行一些操作
        } finally {
            lock.unlock();
        }        
    }
}
```
非公平锁可能使线程饥饿，为什么它又被设置为ReentrantLock的默认实现呢？这是因为公平性锁保证FIFO原则的代价是大量的线程切换，非公平锁虽然可能造成线程饥饿，但减少了线程切换，增加了吞吐量。  

关于AQS类分析本篇博文就到这里，下一篇博文会分析下基于AQS实现的ReentrantReadWriteLock和AQS中的基于Condition的等待/通知机制实现。  

(完)

参考文章：
《Java并发编程的艺术》     
[JDK1.8源码分析之AbstractQueuedSynchronizer（二）](https://www.cnblogs.com/leesf456/p/5350186.html)  


 

  