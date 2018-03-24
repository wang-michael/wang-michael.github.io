# java.lang.Thread学习笔记
## 一、概述 
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

为了标识每个线程，每个线程都有自己的名字，两个线程name可能相同，如果线程创建的时候名称未指定，就会为其自动产生一个新的名称。    

## 二、Thread类对外提供的所有API 
需要注意的构造方法：  
* 通过实现Runnable接口创建线程时的参数最多的构造方法：Thread(ThreadGroup group, Runnable target, String name, long stackSize)：

需要注意的静态方法：
* activeCount()：返回当前线程所在线程组及其下级线程组中估计的活跃线程的数量。线程组的概念？？？
* getDefaultUncaughtExceptionHandler()：返回当线程由于抛出的异常突然中止时调用的默认的handler。Thread.UncaughtExceptionHandler	概念？？？
* holdsLock(Object obj)：当且仅当当前线程持有指定对象的监视器锁返回true
* setDefaultUncaughtExceptionHandler(Thread.UncaughtExceptionHandler eh)：为当前线程设置默认的异常处理器。

* currentThread()：返回一个当前正在执行的线程的对象引用。如何实现的？？？
* interrupted()：测试当前线程是否被interrupted。如何实现的？？？
* sleep(long millis, int nanos)：使当前线程睡眠指定时间，如何实现的？？？
* yield()：给调度器一个暗示，当前正在执行的线程想要让出处理器，如何实现的？？？

需要注意的实例方法：
* start()：开始执行一个线程
* void run()：线程运行执行的实际方法，注意模板设计模式
* interrupt()：中断这个线程，何为中断？？？
* void join()：等待这个线程运行结束；内部如何实现的？？？  

* getId()：返回线程的唯一标识符，如何唯一的确定一个线程的？？？  
* getName()：返回线程的名称  
* getPriority()：返回线程的优先级  
* StackTraceElement[] getStackTrace()：返回线程调用栈的dump，属于虚拟机栈？？？包含本地方法栈方法调用吗？？？
* getState()：返回线程的状态，线程状态是何时转换的？是由JVM还是程序中显示转换的？
* ThreadGroup getThreadGroup()：返回当前线程归属的线程组，线程组的概念？现在过时了吗？
* Thread.UncaughtExceptionHandler getUncaughtExceptionHandler()：返回当前线程使用的ExceptionHandler
* boolean isAlive()：测试当前线程是否存活，何为存活？？？
* boolean isDaemon()：测试当前线程是否为守护线程，内部如何实现的？？？
* boolean isInterrupted()：测试当前线程是不是被中断了。
* void join(long millis)：等待指定时间，内部如何实现的？？？
* void setDaemon(boolean on)：设置线程为user Thread还是daemon Thread,线程执行之后还可设置为daemon吗？  
* void setPriority(int newPriority)：线程执行之后还能改变优先级吗？
* void setUncaughtExceptionHandler(Thread.UncaughtExceptionHandler eh)：线程运行之后还能设置吗？


已过时的实例方法：为何过时？？？从什么时候开始过时的？？？
* destroy()：
* void resume()：
* void stop()：
* void suspend()：

源码方法理解的顺序：
start() -> run() -> interrupt()【包含interrupted与isInterrupted()】 -> join() -> sleep() -> yield() -> wait()

## 三、其它Thread类相关概念 

### Thread可能的状态
* NEW：一个还没开始的线程处于这种状态
* RUNNABLE：一个正在执行的线程的状态；处于这种状态的线程有可能正在被虚拟机执行，也有可能在等待操作系统的其它资源(例如处理器)，我觉得也可以理解为通常说的就绪状态
* BLOCKED：处于这个状态的线程是在等待获取监视器上的锁；通常有两种情况，一是执行synchronized块或方法之前等待获取监视器上的锁；二是在调用Object.wait()方法之后被notify，等待重新获取synchronized块或方法上监视器的锁
* WAITING：处于waiting状态上的线程通常是在等待另一个线程来执行一个特殊的操作；通常有以下三种情况：一、调用Object.wait()方法并且不指定超时参数；二、调用Thread.join()方法并且不指定超时参数；三、调用LockSupport.park方法。 举例来说，调用Object.wait()方法的线程需要等待另一个线程在这个对象上调用Object.notify()或者Object.notifyAll()方法；调用了Thread.join()方法的线程需要waiting到指定线程结束之后才继续执行。  
* TIMED_WAITING：处于waitting状态，但最多waitting到指定的超时时间；通常在调用以下方法之后进入到这种状态：Thread.sleep(),调用指定时间参数的Object.wait(),调用指定时间参数的Thread.sleep(),调用LockSupport.parkNanos及LockSupport.parkUntil方法。  
* TERMINATED： 执行结束的线程处于这种状态。        

关于Thread状态系列文章，一定要阅读：[Java线程状态专题](https://xiaogd.net/category/java%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81/)                                

Java线程的状态与底层os进程的状态并不是一一对应的，比如Java线程中的Runnable状态可能既包含底层os中的TASK_RUNNING状态又包含部分TASK_INTERRUPTIBLE状态；Java线程中的Blocked与Waiting状态可以看做是底层os TASK_INTERRUPTIBLE状态的一种细分。  

**问题：**既然Java线程是映射到底层操作系统并依靠os来进行调度的，那么Java中区分线程的这些状态又有何意义呢？？？   
Java线程状态是为了在实现线程库中相关功能时而做的区分,比如为了实现synchronized关键字需要区分entryset -> 对应blocked状态 和 waitset，在锁释放时需要在entryset而不是waitset中选择下一个线程获得锁；在Thread.start()时判断一个线程的状态是否为NEW来判断线程是不是第一次启动；如果仅仅使用操作系统提供的几种进程状态，对于Java中的线程实现不够灵活；                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            