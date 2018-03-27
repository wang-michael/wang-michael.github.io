---
layout:     post
title:      深入学习java并发编程之Thread.interrupt()源码分析
date:       2018-3-27
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 并发
    - JDK源码阅读
---
>本文关于Thread.interrupt() native相关方法源码分析的过程参考文章[Java线程源码解析之interrupt](https://www.jianshu.com/p/1492434f2810)，在文中部分地方加入了自己对于原作者文章中内容的理解，仅用于备忘。  

_ _ _
### **前言**  
当我们在调用Thread.interrupt()方法的时候，被中断的线程通常有两种方式对中断进行响应，一种是主动响应，比如：  
```java
while(!Thread.currentThread().isInterrupted()) {
    ...
}
```
还有就是被动响应，通常线程处在以下状态之一时会对中断产生被动响应：  
* 如果被中断的线程阻塞在Object类对象的 wait(), wait(long), 或者 wait(long, int)等方法的调用上，或者是当前类的 join(), join(long), join(long, int), sleep(long), 或者 sleep(long, int)等方法，这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false。  
* 如果被中断的线程正阻塞在一个InterruptibleChannel上的IO操作，然后这个channel就会被关闭，这个线程的中断状态会被设置，然后这个线程会抛出ClosedByInterruptException。  
* 如果这个线程正阻塞在一个Selector之上，然后这个线程的interrupt状态就会被设置并且会立即从select操作返回，返回值可能非0，就像是在这个selector上调用wakeup方法一样。  
* 如果以上的条件都不成立，那么该线程的中断状态将被设置。比如说线程正阻塞于ServerSocket.accept()操作之上，此时对其使用interrupt()是没有影响的。但是在accept()方法返回之后此线程上接下来的操作可能被interrupt()影响，比如调用sleep()操作。     

在了解到上述线程被动响应中断的几种方式时，总会产生几个疑问：
* 一个线程在进行sleep()、wait()、join()的时候在底层os上对应的状态都是TASK_INTERRUPTIBLE，那么是如何响应中断的？肯定不会是轮询，是使用了类似linux上提供的信号机制吗？  
* 如果被中断的线程阻塞在一个InterruptibleChannel上的IO操作或者阻塞在一个Selector之上，
又是如何响应中断的？为什么ServerSocket.accept()不能响应中断？  

下面就先从Thread.interrrupt()的分析中解决线程如何被动响应中断的，之后介绍线程主动使用Thread.interrrupt()响应中断的使用场景及相关注意事项。  

_ _ _
### **线程是如何被动响应中断的？？？**  
从Thread.interrupt()的java源代码开始看起： 
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
private volatile Interruptible blocker; // 让线程在接口实现中自定义中断处理过程
private final Object blockerLock = new Object();

public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            // 唤醒阻塞的SocketChannel及Selector等操作会执行这里
            interrupt0();           // Just to set the interrupt flag
            // interrupt0() 只是设置了个标记，实际唤醒操作由b.interrupt(this);实现          
            b.interrupt(this);
            return;
        }
    }
    // 用来唤醒sleep、wait、join等操作会执行这里，原理是通过park、unpark操作唤醒。 
    interrupt0();
}

private native void interrupt0();
```
#### **阻塞在sleep(long)等类似方法上的线程是如何被动响应中断的？？？**  
之前在分析Thread.start的时候曾经提到，JavaThread有三个成员变量:  
```c++
//用于synchronized同步块和Object.wait() 
ParkEvent * _ParkEvent ; 
//用于Thread.sleep() 
ParkEvent * _SleepEvent ; 
//用于unsafe.park()/unpark(),供java.util.concurrent.locks.LockSupport调用， 
//因此它支持了java.util.concurrent的各种锁、条件变量等线程同步操作,是concurrent的实现基础 
Parker* _parker;
```
初步猜测interrupt实现应该与此有关系；interrupt方法的源码也在jvm.cpp文件中:
```c++
JVM_ENTRY(void, JVM_Interrupt(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_Interrupt");

  // Ensure that the C++ Thread and OSThread structures aren't freed before we operate
  oop java_thread = JNIHandles::resolve_non_null(jthread);
  MutexLockerEx ml(thread->threadObj() == java_thread ? NULL : Threads_lock);
  // We need to re-resolve the java_thread, since a GC might have happened during the
  // acquire of the lock
  JavaThread* thr = java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread));
  if (thr != NULL) {
    Thread::interrupt(thr);
  }
JVM_END
```
JVM_Interrupt对参数进行了校验，然后直接调用Thread::interrupt:  
```c++
void Thread::interrupt(Thread* thread) {
  trace("interrupt", thread);
  debug_only(check_for_dangling_thread_pointer(thread);)
  os::interrupt(thread);
}
```
Thread::interrupt调用os::interrupt方法实现,os::interrupt方法定义在os_linux.cpp:
```c++
void os::interrupt(Thread* thread) {
  assert(Thread::current() == thread || Threads_lock->owned_by_self(),
    "possibility of dangling Thread pointer");

  //获取系统native线程对象
  OSThread* osthread = thread->osthread();

  if (!osthread->interrupted()) {
    osthread->set_interrupted(true);
   //内存屏障，使osthread的interrupted状态对其它线程立即可见
    OrderAccess::fence();
    //前文说过，_SleepEvent用于Thread.sleep,线程调用了sleep方法，则通过unpark唤醒
    ParkEvent * const slp = thread->_SleepEvent ;
    if (slp != NULL) slp->unpark() ;
  }

  //_parker用于concurrent相关的锁，此处同样通过unpark唤醒
  if (thread->is_Java_thread())
    ((JavaThread*)thread)->parker()->unpark();
  //synchronized同步块和Object.wait() 唤醒
  ParkEvent * ev = thread->_ParkEvent ;
  if (ev != NULL) ev->unpark() ;

}
```
由此可见，interrupt其实就是通过ParkEvent的unpark方法唤醒对象；以Thread.sleep()为例，我们指定让其休眠30s，其它线程在此线程上调用了interrupt()方法，这个线程在被unpark唤醒之后检测到发生了中断会立即抛出InterruptedException返回并清除中断标志，此时可能刚刚休眠了1s，Thread.sleep()对应的native c++代码如下：  
```c++
int os::sleep(Thread* thread, jlong millis, bool interruptible) {
  assert(thread == Thread::current(),  "thread consistency check");

  ParkEvent * const slp = thread->_SleepEvent ;
  slp->reset() ;
  OrderAccess::fence() ;

  // 如果是可中断的话
  if (interruptible) {
    jlong prevtime = javaTimeNanos();

    for (;;) {
      // 每次从休眠中醒来都会查看中断标志是否被设置
      if (os::is_interrupted(thread, true)) {
        return OS_INTRPT;
      }

      jlong newtime = javaTimeNanos();

      if (newtime - prevtime < 0) {
        // time moving backwards, should only happen if no monotonic clock
        // not a guarantee() because JVM should not abort on kernel/glibc bugs
        assert(!Linux::supports_monotonic_clock(), "time moving backwards");
      } else {
        millis -= (newtime - prevtime) / NANOSECS_PER_MILLISEC;
      }

      if(millis <= 0) {
        return OS_OK;
      }

      prevtime = newtime;

      {
        assert(thread->is_Java_thread(), "sanity check");
        JavaThread *jt = (JavaThread *) thread;
        ThreadBlockInVM tbivm(jt);
        OSThreadWaitState osts(jt->osthread(), false /* not Object.wait() */);

        jt->set_suspend_equivalent();
        // cleared by handle_special_suspend_equivalent_condition() or
        // java_suspend_self() via check_and_wait_while_suspended()

        // unpark操作调用后即使没有休眠到指定时间也会立即唤醒
        slp->park(millis);

        // were we externally suspended while we were waiting?
        jt->check_and_wait_while_suspended();
      }
    }
  } else {
    OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
    jlong prevtime = javaTimeNanos();

    for (;;) {
      // It'd be nice to avoid the back-to-back javaTimeNanos() calls on
      // the 1st iteration ...
      jlong newtime = javaTimeNanos();

      if (newtime - prevtime < 0) {
        // time moving backwards, should only happen if no monotonic clock
        // not a guarantee() because JVM should not abort on kernel/glibc bugs
        assert(!Linux::supports_monotonic_clock(), "time moving backwards");
      } else {
        millis -= (newtime - prevtime) / NANOSECS_PER_MILLISEC;
      }

      if(millis <= 0) break ;

      prevtime = newtime;
      slp->park(millis);
    }
    return OS_OK ;
  }
}
```
我们还需要注意的一些地方是：  
* object.wait、Thread.sleep和Thread.join会抛出InterruptedException并清除中断状态；
* Lock.lock()方法不会响应中断，Lock.lockInterruptibly()方法则会响应中断并抛出异常，区别在于park()等待被唤醒时lock会继续执行park()来等待锁，而 lockInterruptibly会抛出异常；
* synchronized被唤醒后会尝试获取锁，失败则会通过循环继续park()等待，因此实际上是不会被interrupt()中断的;
* 一般情况下，抛出异常时，会清空Thread的interrupt状态，在编程时需要注意；

#### **阻塞在InterruptibleChannel上的IO操作或Selector的select操作上的线程是如何被动响应中断的？？？**  
之前的interrupt方法有这么一段：
```java
private volatile Interruptible blocker;
private final Object blockerLock = new Object();
synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }

```
其中blocker是Thread的成员变量,Thread提供了blockedOn方法可以设置blocker:
```java
void blockedOn(Interruptible b) {
    synchronized (blockerLock) {
        blocker = b;
    }
}
```
如果一个nio通道实现了InterruptibleChannel接口，就可以响应interrupt()中断，其原理就在InterruptibleChannel接口的抽象实现类AbstractInterruptibleChannel的方法begin()中:
```java
protected final void begin() {
    if (interruptor == null) {
        interruptor = new Interruptible() {
                public void interrupt(Thread target) {
                    synchronized (closeLock) {
                        if (!open)
                            return;
                        open = false;
                        interrupted = target;
                        try {
                            AbstractInterruptibleChannel.this.implCloseChannel();
                        } catch (IOException x) { }
                    }
                }};
    }
    blockedOn(interruptor);//设置当前线程的blocker为interruptor
    Thread me = Thread.currentThread();
    if (me.isInterrupted())
        interruptor.interrupt(me);
}

protected final void end(boolean completed)
    throws AsynchronousCloseException
{
    blockedOn(null);//设置当前线程的blocker为null
    Thread interrupted = this.interrupted;
   //如果发生中断，Thread.interrupt方法会调用Interruptible的interrupt方法，
  //设置this.interrupted为当前线程
    if (interrupted != null && interrupted == Thread.currentThread()) {
        interrupted = null;
        throw new ClosedByInterruptException();
    }
    if (!completed && !open)
        throw new AsynchronousCloseException();
}
```
```java
//Class java.nio.channels.Channels.WritableByteChannelImpl
     public int write(ByteBuffer src) throws IOException {
        ......    
        try {
            begin();
            out.write(buf, 0, bytesToWrite);
        finally {
            end(bytesToWrite > 0);
        }
        ......
    }

//Class java.nio.channels.Channels.ReadableByteChannelImpl
    public int read(ByteBuffer dst) throws IOException {
        ......    
        try {
            begin();
            bytesRead = in.read(buf, 0, bytesToRead);
        finally {
            end(bytesRead > 0);
        }
        ......
    }
```
以上述代码为例，nio通道的ReadableByteChannel每次执行阻塞方法read()前，都会执行begin()，把Interruptible回调接口注册到当前线程上。当线程中断时，Thread.interrupt()触发回调接口Interruptible**关闭io通道,导致read方法返回，**最后在finally块中执行end()方法检查中断标记，抛出ClosedByInterruptException;

**这里需要强调的一点是被阻塞的操作返回是因为关闭了IO通道，而不是因为设置了中断标记：**  
```java
synchronized (blockerLock) {
    Interruptible b = blocker;
    if (b != null) {
        interrupt0();           // Just to set the interrupt flag
        // 这是导致阻塞操作返回的方法
        b.interrupt(this);
        return;
    }
}

interruptor = new Interruptible() {
    public void interrupt(Thread target) {
        synchronized (closeLock) {
            if (!open)
                return;
            open = false;
            interrupted = target;
            try {
                // 由于IO通道关闭导致阻塞在这个Channel上的操作返回
                AbstractInterruptibleChannel.this.implCloseChannel();
            } catch (IOException x) { }
        }
    }};
```
Selector的实现类似：
```java
//java.nio.channels.spi.AbstractSelector
  protected final void begin() {
        if (interruptor == null) {
            interruptor = new Interruptible() {
                    public void interrupt(Thread ignore) {
                        AbstractSelector.this.wakeup();
                    }};
        }
        AbstractInterruptibleChannel.blockedOn(interruptor);
        Thread me = Thread.currentThread();
        if (me.isInterrupted())
            interruptor.interrupt(me);
    }
protected final void end() {
    AbstractInterruptibleChannel.blockedOn(null);
}
//sun.nio.ch.class EPollSelectorImpl
protected int doSelect(long timeout) throws IOException {
        ......
        try {
            begin();
            pollWrapper.poll(timeout);
        } finally {
            end();
        }
        ......
    }
```
可以看到当发生中断时会调用wakeup方法唤醒poll方法，但并不会抛出中断异常；

#### **总结**
源码分析到这里我们上面的问题都得到了回答：
* 对于阻塞在sleep()、wait()、join()等方法上的线程，中断响应的实现是通过ParkEvent的unpark方法唤醒对象(内部使用了linux下的信号处理机制)；唤醒线程后检测到中断后对其进行处理；
* 对于阻塞在一个InterruptibleChannel上的IO操作或者阻塞在一个Selector之上的线程，唤醒方式在我们自定义实现的Interruptible接口中，比如通过关闭通道方式唤醒阻塞的InterruptibleChannel；通过Selector.wakeup()唤醒阻塞在select操作上的线程。  
* 为什么ServerSocket等类的阻塞操作不能被中断唤醒呢？这是因为这些类并没有提供自定义实现的Interruptible接口，仅依靠interrupt0()内操作并不能将其唤醒。  

_ _ _
### **interrupt()使用场景及相关注意事项**  
#### **通过设置中断标志模拟Thread.interrupt()操作**
```java
public class InterruptTest {

    private static volatile boolean interrupt = false;

    public static boolean getInterrupt() {
        return interrupt;
    }

    public static void setInterupt(boolean value) {
        interrupt = value;
    }

    public void method1() {
        try{
            int i = 1;
            while(!interrupt){
                System.out.println("thread continue excute " + i++);
            }
        }catch(Exception e){
            System.out.println(e.getMessage());
        }
    }

    public static void main(String args[]) throws InterruptedException, IOException {
        InterruptTest test = new InterruptTest();

        Thread thread = new Thread(new Runnable(){

            @Override
            public void run() {
                test.method1();
            }

        });

        thread.start();
        Thread.sleep(100);
        InterruptTest.setInterupt(true);
        thread.join();

        System.out.println("Main thread finished!");
    }
}
```
这种方式当然可以有效的通知一直有任务运行的线程去中断自己。但是，如果while循环中线程一直处于阻塞状态，比如Thread.sleep()很长时间。显然线程是无法及时获取中断标志的。考虑下面这种场景。这个时候就该Thread.interrupt出场了。
```java
public class InterruptTest {

    private static volatile boolean interrupt = false;

    public static boolean getInterrupt() {
        return interrupt;
    }
    
    public static void setInterupt(boolean value) {
        interrupt = value;
    }
    
    public void method2() {
            int i=1;
            while(!interrupt) {
                System.out.println("thread continue excute " + i++);       
                try{ 
                    Thread.sleep(30000); 
                } catch(Exception e) {
                    setInterupt(true);
                    System.out.println(e.getMessage());
                }     
            }     
    }
    
    public static void main(String args[]) throws InterruptedException, IOException {
        InterruptTest test = new InterruptTest();
        
        Thread thread = new Thread(new Runnable() {

            @Override
            public void run() {
                test.method2();
            }
            
        });
        
        thread.start();
        Thread.sleep(100);
        thread.interrupt();
        thread.join();
        
        System.out.println("Main thread finished!");
    }
}
```
对于一些不理会Thread.interrupt()操作的I/O阻塞，就需要针对不同的情况做处理了。比如，socket阻塞在accpet函数，interrupt函数就无能为力了，这时候就可以通过socket的close函数来关闭端口。

**Thread.interrupt使用的其他一些注意事项：**
* 在线程使用start()方法启动之前，使用thread.interrupt()并不会有影响；线程启动之后，如果调用thread.interrupt()方法时其中断标志已经设置为true，本次interrupt()方法调用无影响。  
* 中断已死的线程并不会产生任何影响。  
* isInterrupted()与interrupted():从内部代码实现中可以看出两者区别仅仅在于查询中断状态之后是否清除中断标志位。  

(完)