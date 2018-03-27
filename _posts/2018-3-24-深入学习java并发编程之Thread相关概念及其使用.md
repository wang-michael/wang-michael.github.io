---
layout:     post
title:      深入学习java并发编程之Thread相关概念及其使用
date:       2018-3-24
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - java基础
    - 并发
---
>本文主要介绍了自己对于java.lang.Thread类中相关API的学习及理解。  

_ _ _
### **Java中线程的几种状态**
* NEW：一个还没开始的线程处于这种状态
* RUNNABLE：一个正在执行的线程的状态；处于这种状态的线程有可能正在被虚拟机执行，也有可能在等待操作系统的其它资源(例如处理器)，我觉得也可以理解为通常说的就绪状态
* BLOCKED：处于这个状态的线程是在等待获取监视器上的锁；通常有两种情况，一是执行synchronized块或方法之前等待获取监视器上的锁；二是在调用Object.wait()方法之后被notify，等待重新获取synchronized块或方法上监视器的锁
* WAITING：处于waiting状态上的线程通常是在等待另一个线程来执行一个特殊的操作；通常有以下三种情况：一、调用Object.wait()方法并且不指定超时参数；二、调用Thread.join()方法并且不指定超时参数；三、调用LockSupport.park方法。 举例来说，调用Object.wait()方法的线程需要等待另一个线程在这个对象上调用Object.notify()或者Object.notifyAll()方法；调用了Thread.join()方法的线程需要waiting到指定线程结束之后才继续执行。  
* TIMED_WAITING：处于waitting状态，但最多waitting到指定的超时时间；通常在调用以下方法之后进入到这种状态：Thread.sleep(),调用指定时间参数的Object.wait(),调用指定时间参数的Thread.sleep(),调用LockSupport.parkNanos及LockSupport.parkUntil方法。  
* TERMINATED： 执行结束的线程处于这种状态。   

关于Thread状态更详细的介绍，可以阅读：[Java线程状态专题](https://xiaogd.net/category/java%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81/)

**需要注意的是虽然Java每个线程映射到linux os中的一个轻量级进程，**但是Java线程的状态与底层os进程的状态并不是一一对应的，比如Java线程中的Runnable状态可能既包含底层os中的TASK_RUNNING状态又包含部分TASK_INTERRUPTIBLE状态；Java线程中的Blocked与Waiting状态可以看做是底层os TASK_INTERRUPTIBLE状态的一种细分。   

查看java线程与os下轻量级进程对应关系的小技巧：
```java
有些时候需要确实进程内部当前运行着多少线程,那么以下几个方法值得一用。

1.根据进程号进行查询:

# pstree -p 进程号

# top -Hp 进程号：查看某个进程和其下面所有子进程的动态，更有用。


jps或ps -ef|grep java可以看到有哪些java进程

top -p $pid -H  加上-H这个参数后，会列出有哪些线程。这样就可以看到哪个线程id最消耗系统资源了。看到的线程id是10进制的数字。

ubuntu命令行下输入 jstack $pid 可以打印出指定java进程的stack状况。

将前边top命令看到的java进程的子进程id转为16进制显示，就可以在jstack的结果中找到它了。

例如以下：
"pool-2-thread-1" prio=10 tid=0x000000004c9b2000 nid=0x11f4 waiting on condition [0x0000000042f36000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x0000000580089050> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:158)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:1987)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:399)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:947)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:907)
        at java.lang.Thread.run(Thread.java:662)


其中的“nid=0x11f4 ”， 11f4就是线程id在操作系统中对应的进程id的16进制表示
```

_ _ _
### **Thread类相关API的使用及个人理解**
#### **线程的创建** 
1、	一个新创建的线程初始优先级设置为与创建它的线程的优先级相同，当且仅当创建线程是守护线程的时候，被创建的线程才被默认设置为守护线程。  
 
2、	结束一个程序的执行通常有两种方式：  
    a) 调用Runtime类的exit方法并且security manager允许退出行为发生
    b) 所有非守护线程都死亡，或者是通过调用run方法，或者是通过抛出一个传播超出run方法的异常

3、	创建线程并执行的两种方式：
* 方式1：继承自Thread类
```java
class PrimeThread extends Thread {
 long minPrime;
 PrimeThread(long minPrime) {
     this.minPrime = minPrime;
 }

 public void run() {
     // compute primes larger than minPrime
      . . .
 }
}

PrimeThread p = new PrimeThread(143);
p.start();
```
* 方式2：实现Runnable接口
```java
class PrimeRun implements Runnable {
 long minPrime;
 PrimeRun(long minPrime) {
     this.minPrime = minPrime;
 }

 public void run() {
     // compute primes larger than minPrime
      . . .
 }
}

PrimeRun p = new PrimeRun(143);
new Thread(p).start();
```
实际上采用了模板设计模式，把run方法使用Runnable接口提取出来，其余部分封装为模板。   

还有一种创建线程的方式是使用Callable接口和FutureTask方式，但这种方式实质上也是对于方式二的封装，因此没有单列出来。  

关于线程创建时在os层面上究竟做了什么，可以参考我的另一篇博客：[深入学习java并发编程之创建一个线程并start()究竟做了什么？](https://wang-michael.github.io/2018/03/25/%E6%B7%B1%E5%85%A5%E5%AD%A6%E4%B9%A0java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E7%BA%BF%E7%A8%8B%E5%B9%B6start()%E7%A9%B6%E7%AB%9F%E5%81%9A%E4%BA%86%E4%BB%80%E4%B9%88/)

#### **线程的中断**
* 如果被中断的线程阻塞在Object类对象的 wait(), wait(long), 或者 wait(long, int)等方法的调用上，或者是当前类的 join(), join(long), join(long, int), sleep(long), 或者 sleep(long, int)等方法，这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false。  
* 如果被中断的线程正阻塞在一个InterruptibleChannel上的IO操作，然后这个channel就会被关闭，这个线程的中断状态会被设置，然后这个线程会抛出ClosedByInterruptException。  
* 如果这个线程正阻塞在一个Selector之上，然后这个线程的interrupt状态就会被设置并且会立即从select操作返回，返回值可能非0，就像是在这个selector上调用wakeup方法一样。  
* 如果以上的条件都不成立，那么该线程的中断状态将被设置。比如说线程正阻塞于ServerSocket.accept()操作之上，此时对其使用interrupt()是没有影响的。但是在此线程上接下来的操作可能被interrupt()影响，比如调用sleep()操作。     
* 中断已死的线程并不会产生任何影响。  
* isInterrupted()与interrupted():从内部代码实现中可以看出两者区别仅仅在于查询中断状态之后是否清除中断标志位。  
* 在线程使用start()方法启动之前，使用thread.interrupt()并不会有影响；线程启动之后，如果调用thread.interrupt()方法时其中断标志已经设置为true，本次interrupt()方法调用无影响。  
```java
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}

public boolean isInterrupted() {
    return isInterrupted(false);
}  

private native boolean isInterrupted(boolean ClearInterrupted);
```
interrupt内部如何实现的？为何sleep的同时还能查询中断状态？？？是轮询吗？？？ 
```java
/* Set the blocker field; invoked via sun.misc.SharedSecrets from java.nio code
 */
void blockedOn(Interruptible b) {
    synchronized (blockerLock) {
        blocker = b;
    }
}

/* The object in which this thread is blocked in an interruptible I/O
 * operation, if any.  The blocker's interrupt method should be invoked
 * after setting this thread's interrupt status.
 */
private volatile Interruptible blocker;
private final Object blockerLock = new Object();

public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            // interrupt0() 只是设置了个标记，实际唤醒操作由b.interrupt(this);实现
            // 唤醒阻塞的SocketChannel及Selector等
            b.interrupt(this);
            return;
        }
    }
    interrupt0();// 用来唤醒sleep、wait、join等操作，通过park、unpark操作唤醒。  
}

private native void interrupt0();
```
socketChannel、ServerSocketChannel、Selector阻塞结束时通过interrupt唤醒的还是关闭通道时唤醒的？  

sleep、join、wait等是通过interrupt唤醒的吗？  

中断的使用场景？？？  
#### **线程相关属性的设置**  
**1、Thread.setPriority(int newPriority):**设置线程的优先级。  
```java
public class PriorityTest {

    private static volatile boolean notStarted = true;
    private static volatile boolean notEnded = true;

    public static void main(String[] args) throws InterruptedException {
        List<Job> jobList = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            int priority = i < 5 ? Thread.MIN_PRIORITY : Thread.MAX_PRIORITY;
            Job tempJob = new Job(priority);
            Thread tempThread = new Thread(tempJob);
            tempThread.setPriority(priority);
            jobList.add(tempJob);
            tempThread.start();
        }
        notStarted = false;
        TimeUnit.SECONDS.sleep(10);
        notEnded = false;
        for (Job job : jobList) {
            System.out.println("priority: " + job.getPriority() + " counter: " + job.getCounter());
        }
    }

    private static class Job implements Runnable {

        private int counter;
        private int priority;

        public Job(int priority) {
            this.priority = priority;
        }

        @Override
        public void run() {
            while (notStarted) {// 所有线程都开始执行之后才计算count
                Thread.yield();
            }
            while (notEnded) {
                counter++;
                Thread.yield();
            }
        }

        public int getCounter() {
            return counter;
        }

        public int getPriority() {
            return priority;
        }
    }
}

在我电脑上执行结果为(win10, java1.8)：
priority: 1 counter: 220934
priority: 1 counter: 220846
priority: 1 counter: 221264
priority: 1 counter: 220914
priority: 1 counter: 220733
priority: 10 counter: 4187476
priority: 10 counter: 4194761
priority: 10 counter: 4187167
priority: 10 counter: 4175727
priority: 10 counter: 4178632

可见设置的优先级是起了作用的。
```

**2、Thread.setDaemon(int newPriority):**设置线程的优先级；需要注意的是Daemon属性需要在启动线程之前设置，不能在启动线程之后设置。如果当前在JVM虚拟机中执行的线程已经没有非Daemon线程，虚拟机需要退出，**Java虚拟机中的所有Daemon线程都需要立即终止，即使其finally块中代码还没有执行。**  

需要注意的一点是Daemon线程守护的是创建它的线程，会随创建它的线程执行结束而结束。
```java
public class DaemonThread {

    private static class Thread1 implements Runnable {

        @Override
        public void run() {
            try {
                while (true) {
                    System.out.println("Thread1执行");
                    TimeUnit.SECONDS.sleep(1);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println("finally 1 执行");
            }
        }
    }

    private static class Thread2 implements Runnable {

        @Override
        public void run() {
            Thread daemonThread = new Thread(new Thread1());
            daemonThread.setDaemon(true);
            daemonThread.start();
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println("Thread2 finish");
            }
        }
    }

    public static void main(String[] args) {
        Thread thread2 = new Thread(new Thread2());
        thread2.start();
        System.out.println("main finish");
    }
}
// main线程结束之后对Thread1无影响，Thread2线程结束之后Thread1线程也会立即结束。  
```   
