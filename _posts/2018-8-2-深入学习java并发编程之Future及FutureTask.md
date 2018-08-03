---
layout:     post
title:      深入学习java并发编程之Future及FutureTask
date:       2018-8-2
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础
    - 并发
---
>本文主要介绍了自己对于juc包中的Future及FutureTask类中相关API的学习及实现原理的理解。    

_ _ _
### **前言**
Future接口给我们提供了一些监测开启的任务的运行状态的方法，比如get()方法阻塞直到查询的任务执行完毕，cancel()方法取消一个任务的执行等等。FutureTask类是Future接口的一个实现，它同时还实现了Runnable接口，可以作为Thread中执行的任务。使用示例如下：  
```java
public class FutureStudy {

    public static void main(String[] args) {
        Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<>(task);
        Thread thread = new Thread(futureTask);
        thread.start();

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }

        System.out.println("主线程在执行任务");

        try {
            System.out.println("task运行结果" + futureTask.get()); // 阻塞直到call方法内任务执行完毕
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        System.out.println("所有任务执行完毕");
    }

    static class Task implements Callable<Integer> {
        @Override
        public Integer call() throws Exception {
            System.out.println("子线程在进行计算");
            Thread.sleep(3000);
            int sum = 0;
            for(int i = 0; i < 100; i++)
                sum += i;
            return sum;
        }
    }
}
```  

接下来就来看下FutureTask内部各方法的实现原理。  

_ _ _
### **FutureTask类源码分析**
首先来看下FutureTask的UML图：  
<img src="/img/2018-8-2/FutureTaskUML.png" width="500" height="500" alt="FutureTask UML图" />
<center>图1：FutureTask UML图</center>    

#### **构造方法与成员变量**
成员变量如下：  
```java
/**
 * 用来标识一个任务的运行状态，初始时设置为NEW，仅在方法set，setException和cancel中转换为最终状态
 * 在完成期间，状态可以采用COMPLETING的瞬态值（当设置结果时）或INTERRUPTING（在中断runner以满足cancel（true）时）。
 * 从这些中间状态到最终状态的转换使用更便宜的ordered/lazy writes，因为值是唯一的并且不能进一步修改。
 * 
 * 有可能出现的状态转换包含：
 * NEW -> COMPLETING -> NORMAL
 * NEW -> COMPLETING -> EXCEPTIONAL
 * NEW -> CANCELLED
 * NEW -> INTERRUPTING -> INTERRUPTED
 */
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5; // 状态5与状态6何时会出现？？？
private static final int INTERRUPTED  = 6;

private Callable<V> callable; // 实际运行的任务，构造函数中传入
private Object outcome; // 存储返回的结果或者从get方法中要抛出的exception
private volatile Thread runner; // 运行callable中call方法的线程; CASed during run()
private volatile WaitNode waiters; // 等待队列，保存所有等待获取结果的线程  
```

构造方法如下：  
```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

#### **run方法源码解析**
在上面代码示例中，以new Thread(futureTask).start()的方式开启了一个新的线程,新线程开始执行之后首先执行的必然是FutureTask类中的run方法:  
```java
public void run() {
    // 上一个任务未执行完或者CAS将执行任务的线程设置为当前线程失败
    // 一个FutureTask实例对象在run方法执行一次之后，此FutureTask就不可再被执行了，因为其状态不再为NEW
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread())) 
        return;
    try {
        Callable<V> c = callable; // 创建FutureTask实例对象时传入的callable对象
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s); 
    }
}
// run方法执行过程中不论是顺利获得结果还是抛出异常，都将其存储在outcome对象中
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
protected void setException(Throwable t) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        finishCompletion();
    }
}
// 作用是自旋等待状态由INTERRUPTING变为INTERRUPTED
private void handlePossibleCancellationInterrupt(int s) {    
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt  
}
```

#### **get方法源码解析**
FutureTask类中get()方法会一直阻塞到等待的线程执行完毕，get(long timeout, TimeUnit unit)可以指定阻塞时间，两方法内部实现原理大致相同，这里仅介绍get()方法内部实现：  
```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING) // 如果线程正在执行过程中，但是还没有执行完毕
        s = awaitDone(false, 0L);
    return report(s);
}
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL) // 正常结束，返回执行结果
        return (V)x;
    if (s >= CANCELLED) // future.cancel中取消了此任务
        throw new CancellationException();
    throw new ExecutionException((Throwable)x); // run方法中抛出的异常保存在了outcome对象中，现在向上抛出
}
// 等待绑定的线程执行完毕或者由于中断或超时放弃等待
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {  // 等待的线程由于中断从park中被唤醒
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q); // 将当前线程包装后加入等待队列
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this);
    }
}
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

#### **cancel方法源码解析**
cancel方法尝试终止这个future绑定到的任务的执行，如果对应的任务已经完成、已经被cancelled、或者由于一些其它原因不能被cancelled，这次尝试就会失败。当cancel方法被调用的时候，如果这个任务还没有开始。这个任务就永远不会再执行；如果这个任务已经开始了，那么cancel方法中的mayInterruptIfRunning参数就决定了是否对正在执行任务的线程执行interrupt方法。  
```java
// mayInterruptIfRunning参数为true，即对正在执行任务的线程执行interrupt方法，否则则不执行
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED))) // 先更改任务状态再进行cancel，保证cancel的可见性
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```
这个方法调用之后，接下来对于Future.isDone()方法的调用都会返回true，如果本次cancel方法返回true之后，接下来对于Future.isCancelled()的调用会一直返回true。  

#### **finishCompletion方法源码解析**
在FutureTask到达最终状态之后(可能为NORMAL、EXCEPTIONAL、CANCELLED、INTERRUPTED)，都会执行finishCompletion()方法，下面就来看看方法中究竟做了什么工作：  
```java
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) { // 先将当前FutureTask中的waiters设为null
            for (;;) { // 逐个唤醒等待线程
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```
可见主要功能就是唤醒等待线程。  

(完)  