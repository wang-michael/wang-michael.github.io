---
layout:     post
title:      深入学习java并发编程之Object.wait及Thread.join源码分析
date:       2018-3-28
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 并发
    - JDK源码阅读
---
>关于wait、notify的源码分析来源于博文[Java的wait()、notify()学习三部曲之一：JVM源码分析](https://blog.csdn.net/boling_cavalry/article/details/77793224)，这篇文章内容对我而言很深入很详细，很多地方我目前还不太理解，留待以后更加仔细的解读；转载的目的是在其中增加一些目前自己的理解，仅用于备忘。Thread.join源码分析来源于博文[Java线程源码解析之join](https://www.jianshu.com/p/21f93aa383af)。

_ _ _
### **Object.wait()、notify()源码分析理解**
#### **wait()和notify()的通常用法**
Java多线程开发中，我们常用到wait()和notify()方法来实现线程间的协作，简单的说步骤如下：   
1. A线程取得锁，执行wait()，释放锁; 
2. B线程取得锁，完成业务后执行notify()，再释放锁; 
3. B线程释放锁之后，A线程需要重新取得锁，之后继续执行wait()之后的代码；

#### **关于synchronize修饰的代码块**
通常，对于synchronize(lock){…}这样的代码块，编译后会生成monitorenter和monitorexit指令，线程执行到monitorenter指令时会尝试取得lock对应的monitor的所有权（CAS设置对象头），取得后即获取到锁，执行monitorexit指令时会释放monitor的所有权即释放锁；

#### **一个完整的demo**
为了深入学习wait()和notify()，先用完整的demo程序来模拟场景吧，以下是源码：
```java
public class NotifyDemo {

    private static void sleep(long sleepVal){
        try{
            Thread.sleep(sleepVal);
        }catch(Exception e){
            e.printStackTrace();
        }
    }

    private static void log(String desc){
        System.out.println(Thread.currentThread().getName() + " : " + desc);
    }

    Object lock = new Object();

    public void startThreadA(){
        new Thread(() -> {
            synchronized (lock){
                log("get lock");
                startThreadB();
                log("start wait");
                try {
                    lock.wait();
                }catch(InterruptedException e){
                    e.printStackTrace();
                }

                log("get lock after wait");
                log("release lock");
            }
        }, "thread-A").start();
    }

    public void startThreadB(){
        new Thread(()->{
            synchronized (lock){
                log("get lock");
                startThreadC();
                sleep(100);
                log("start notify");
                lock.notify();
                log("release lock");

            }
        },"thread-B").start();
    }

    public void startThreadC(){
        new Thread(() -> {
            synchronized (lock){
                log("get lock");
                log("release lock");
            }
        }, "thread-C").start();
    }

    public static void main(String[] args){
        new NotifyDemo().startThreadA();
    }
}
```
以上就是本次实战用到的demo，代码功能简述如下：

1. 启动线程A，取得锁之后先启动线程B再执行wait()方法，释放锁并等待；
2. 线程B启动之后会等待锁，A线程执行wait()之后，线程B取得锁，然后启动线程C，再执行notify唤醒线程A，最后退出synchronize代码块，释放锁;
3. 线程C启动之后就一直在等待锁，这时候线程B还没有退出synchronize代码块，锁还在线程B手里；
4. 线程A在线程B执行notify()之后就一直在等待锁，这时候线程B还没有退出synchronize代码块，锁还在线程B手里；
5. 线程B退出synchronize代码块，释放锁之后，线程A和线程C竞争锁；

把上面的代码在Openjdk8下面执行，反复执行多次，都得到以下结果：  
```java
thread-A : get lock
thread-A : start wait
thread-B : get lock
thread-B : start notify
thread-B : release lock
thread-A : get lock after wait
thread-A : release lock
thread-C : get lock
thread-C : release lock
```
针对以上结果，问题来了：  
第一个问题：   
将以上代码反复执行多次，结果都是B释放锁之后A会先得到锁，这又是为什么呢？c一定不能先拿到锁吗？  

第二个问题：  
线程C自开始就执行了monitorenter指令，它能得到锁是容易理解的，但是线程A呢？在wait()之后并没有没有monitorenter指令，那么它又是如何取得锁的呢？  

wait()、notify()这些方法都是native方法，所以只有从JVM源码寻找答案了，本次阅读的是openjdk8的源码；  

#### **带上问题去看JVM源码**  
按照demo代码执行顺序，我整理了如下问题，带着这些问题去看JVM源码可以聚焦主线，不要被一些支线的次要的代码卡住(例如一些异常处理，监控和上报等)：   
1. 为什么在调用Object.wait()之前线程需要持有Object上的锁？  
2. 线程A在wait()的时候做了什么？ 
3. 线程C启动后，由于此时线程B持有锁，那么线程C此时在干啥？ 
4. 线程B在notify()的时候做了什么？ 
5. 线程B释放锁的时候做了什么？

#### **源码中最重要的注释信息**  
在源码中有段注释堪称是整篇文章最重要的说明，请大家始终记住这段信息，处处都用得上：  

**ObjectWaiter对象(将等待锁的线程封装为ObjectWaiter对象)存在于WaitSet、EntryList、cxq等集合中，或者正在这些集合中移动。**  

原文如下：  
```c++
// hotspot\src\share\vm\runtime\objectMonitor.cpp
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
    ...
    // Node may be on the WaitSet, the EntryList (or cxq), or in transition
     // from the WaitSet to the EntryList.
     // See if we need to remove Node from the WaitSet.
     // We use double-checked locking to avoid grabbing _WaitSetLock
     // if the thread is not on the wait queue.
     //
    ...
}
```
**请务必记住这三个集合：WaitSet、EntryList、cxq。**  

好了，接下来看源码分析问题吧：  

#### **线程A在wait()的时候做了什么**  
打开hotspot/src/share/vm/runtime/objectMonitor.cpp,看ObjectMonitor::wait方法：
<img src="/img/2018-3-28/img1.png" width="660" height="660" alt="源码" />

如上图所示，有两处代码值得我们注意： 
1. 绿框中将当前线程包装成ObjectWaiter对象，并且状态为TS_WAIT，这里对应的是jstack看到的线程状态WAITING； 
2. 红框中调用了AddWaiter方法，跟进去看下：
<img src="/img/2018-3-28/img2.png" width="660" height="660" alt="源码" />

这个ObjectWaiter对象被放入了_WaitSet中，_WaitSet是个环形双向链表(circular doubly linked list)

回到ObjectMonitor::wait方法接着往下看，会发现关键代码如下图，当前线程通过park()方法开始挂起(suspend)：
<img src="/img/2018-3-28/img3.png" width="660" height="660" alt="源码" />

至此，我们把wait()方法要做的事情就理清了： 
1. 包装成ObjectWaiter对象，状态为TS_WAIT； 
2. ObjectWaiter对象被放入_WaitSet中； 
3. 当前线程挂起；

#### **线程B持有锁的时候线程C在干啥**  
此时的线程C无法进入synchronized{}代码块，用jstack看应该是BLOCKED状态，如下图：
<img src="/img/2018-3-28/img4.png" width="660" height="660" alt="源码" />

我们看看monitorenter指令对应的源码吧，位置：openjdk/hotspot/src/share/vm/interpreter/interpreterRuntime.cpp
```c++
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
  if (PrintBiasedLockingStatistics) {
    Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
  }
  Handle h_obj(thread, elem->obj());
  assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
         "must be NULL or an object");
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
  assert(Universe::heap()->is_in_reserved_or_null(elem->obj()),
         "must be NULL or an object");
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
IRT_END
```
上面的代码有个if (UseBiasedLocking)判断，是判断是否使用偏向锁的，本例中的锁显然已经不属于当前线程C了，所以我们还是直接看slow_enter(h_obj, elem->lock(), CHECK)方法吧；  

打开openjdk/hotspot/src/share/vm/runtime/synchronizer.cpp：
```c++
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  //是否处于无锁状态
  if (mark->is_neutral()) {
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    lock->set_displaced_header(mark);
    //无锁状态就去竞争锁
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }
    // Fall through to inflate() ...
  } else
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    //如果处于有锁状态，就检查是不是当前线程持有锁，如果是当前线程持有的，就return，然后就能执行同步代码块中的代码了
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }

#if 0
  // The following optimization isn't particularly useful.
  if (mark->has_monitor() && mark->monitor()->is_entered(THREAD)) {
    lock->set_displaced_header (NULL) ;
    return ;
  }
#endif

  // The object header will never be displaced to this lock,
  // so it does not matter what the value is, except that it
  // must be non-zero to avoid looking like a re-entrant lock,
  // and must not look locked either.
  lock->set_displaced_header(markOopDesc::unused_mark());
  //锁膨胀
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
```
线程C在上面代码中的执行顺序如下： 
1. 判断是否是无锁状态，如果是就通过Atomic::cmpxchg_ptr去竞争锁； 
2. 不是无锁状态，就检查当前锁是否是线程C持有； 
3. 不是线程C持有，调用inflate方法开始锁膨胀；

ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);

来看看锁膨胀的源码：
<img src="/img/2018-3-28/img5.png" width="660" height="660" alt="源码" />

如上图，锁膨胀的代码太长，我们这里只看关键代码吧： 
红框中，如果当前状态已经是重量级锁，就通过mark->monitor()方法取得ObjectMonitor指针再返回； 
绿框中，如果还不是重量级锁，就检查是否处于膨胀中状态（其他线程正在膨胀中），如果是膨胀中，就调用ReadStableMark方法进行等待，ReadStableMark方法执行完毕后再通过continue继续检查，ReadStableMark方法中还会调用os::NakedYield()释放CPU资源；

如果红框和绿框的条件都没有命中，目前已经是轻量级锁了(不是重量级锁并且不处于锁膨胀状态)，可以开始膨胀了，如下图：
<img src="/img/2018-3-28/img6.png" width="660" height="660" alt="源码" />

简单来说，锁膨胀就是通过CAS将监视器对象OjectMonitor的状态设置为INFLATING，如果CAS失败，就在此循环，再走前一副图中的的红框和绿框中的判断，如果CAS设置成功，会继续设置ObjectMonitor中的header、owner等字段，然后inflate方法返回监视器对象OjectMonitor；  

**个人理解： 实际上当线程C执行ObjectSynchronizer::inflate()方法尝试进行锁膨胀时，这时锁的状态一定会是重量级锁，因为线程A执行wait()方法释放锁时如果当前锁不是重量级锁，会先把锁膨胀为重量级锁再释放，源码如下：**  
```c++
// -----------------------------------------------------------------------------
//  Wait/Notify/NotifyAll
// NOTE: must use heavy weight monitor to handle wait()->必须使用重量级锁处理wait()
void ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {
  if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  if (millis < 0) {
    TEVENT (wait - throw IAX) ;
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }
  // 获取ObjectMonitor指针时如果当前不是重量级锁，那么会进行膨胀
  ObjectMonitor* monitor = ObjectSynchronizer::inflate(THREAD, obj());
  DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
  monitor->wait(millis, true, THREAD);

  /* This dummy call is in place to get around dtrace bug 6254741.  Once
     that's fixed we can uncomment the following line and remove the call */
  // DTRACE_MONITOR_PROBE(waited, monitor, obj(), THREAD);
  dtrace_waited_probe(monitor, obj, THREAD);
}
```

接下来看看之前slow_enter方法中，调用inflate方法的代码如下：  
```c++
ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
```
所以inflate方法返回监视器对象OjectMonitor之后，会立刻执行OjectMonitor的enter方法，这个方法中开始竞争锁了，方法在openjdk/hotspot/src/share/vm/runtime/objectMonitor.cpp文件中：
<img src="/img/2018-3-28/img7.png" width="660" height="660" alt="源码" />

如上图，红框中表示OjectMonitor的enter方法一进来就通过CAS将OjectMonitor的_owner设置为当前线程，绿框中表示设置成功的逻辑，第一个if表示重入锁的逻辑，第二个if表示第一次设置_owner成功，都意味着竞争锁成功，而我们的线程C显然是竞争失败的，会进入下图中的无线循环，反复调用EnterI方法：  
<img src="/img/2018-3-28/img8.png" width="660" height="660" alt="源码" />

进入EnterI方法看看：
<img src="/img/2018-3-28/img9.png" width="660" height="660" alt="源码" />

如上图，首先构造一个ObjectWaiter对象node，后面的for(;;)代码块中是一段非常巧妙的代码，同一时刻可能有多个线程都竞争锁失败走进这个EnterI方法，所以在这个for循环中，用CAS将_cxq地址放入node的_next，也就是把node放到_cxq队列的首位，如果CAS失败，就表示其他线程把node放入到_cxq的首位了，所以通过for循环再放一次，只要成功，此node就一定在最新的_cxq队列的首位。  

接下来的代码又是一个无限循环，如下图：
<img src="/img/2018-3-28/img10.png" width="660" height="660" alt="源码" />

从上图可以看出，进入循环后先调用TryLock方法竞争一次锁，如果成功了就退出循环，否则就调用Self->_ParkEvent->park方法使线程挂起，这里有自旋锁的逻辑，也就是park方法带了时间参数，就会在挂起一段时间后自动唤醒，如果不是自旋的条件，就一直挂起等待被其他条件唤醒，线程被唤醒后又会执行TryLock方法竞争一次锁，竞争不到继续这个for循环；

到这里我们已经把线程C在BLOCK的时候的逻辑理清楚了，小结如下：  

1. 偏向锁逻辑，未命中；
2. 如果是无锁状态，就通过CAS去竞争锁，此处由于锁已经被线程B持有，所以不是无锁状态；
3. 不是无锁状态，而且锁不是线程C持有(此时锁应该已是重量级锁)，执行锁膨胀，构造OjectMonitor对象；
4. 竞争锁，竞争失败就将线程加入_cxq队列的首位；(这里也可以体现synchronized是非公平锁，线程来了之后不管_cxq队列中有没有正在等待的线程，直接竞争锁，失败后再加入到_cxq队列中)；
5. 开始无限循环，竞争锁成功就退出循环，竞争失败线程挂起，等待被唤醒后继续竞争；(队列中被挂起的线程可能等待被指定条件唤醒之后重新竞争锁，也有可能在超过指定时间后醒来竞争锁，竞争失败后继续睡眠)

#### **线程B在notify()的时候做了什么**  
接下来该线程B执行notify了，代码是objectMonitor.cpp的ObjectMonitor::notify方法：
<img src="/img/2018-3-28/img11.png" width="660" height="660" alt="源码" />
如上图所示，首先是Policy的赋值，其次是调用DequeueWaiter()方法将_WaitSet队列的第一个值取出并返回，还记得_WaitSet么？所有wait的线程都被包装成ObjectWaiter对象然后放进来了；   
接下来对ObjectWaiter对象的处理方式，根据Policy的不同而不同：   
Policy == 0：放入_EntryList队列的排头位置；  
Policy == 1：放入_EntryList队列的末尾位置；  
Policy == 2：_EntryList队列为空就放入_EntryList，否则放入_cxq队列的排头位置；  
<img src="/img/2018-3-28/img12.png" width="660" height="660" alt="源码" />
如上图所示，请注意把ObjectWaiter的地址写到_cxq变量的时候要用CAS操作，因为此时可能有其他线程正在竞争锁。  

Policy == 3：放入_cxq队列中，末尾位置；更新_cxq变量的值的时候，同样要通过CAS注意并发问题；  
**问题：**当_cxq队列不为空(也就是进入下面for循环中的else中)，为什么不使用CAS操作更新呢？？？  
<img src="/img/2018-3-28/img13.png" width="660" height="660" alt="源码" />

Policy等于其他值，立即唤醒ObjectWaiter对应的线程；

小结一下，线程B执行notify时候做的事情：
1. 执行过wait的线程都在队列_WaitSet中，此处从_WaitSet中取出第一个；
2. 根据Policy的不同，将这个线程放入_EntryList或者_cxq队列中的起始或末尾位置；

#### **线程B释放锁的时候做了什么**  
接下来到了揭开问题的关键了，我们来看objectMonitor.cpp的ObjectMonitor::exit方法；
<img src="/img/2018-3-28/img14.png" width="660" height="660" alt="源码" />

如上图，方法一进来先做一些合法性判断，接下来如红框所示，是偏向锁逻辑，偏向次数减一后直接返回，显然线程B在此处不会返回，而是继续往下执行；

根据QMode的不同，有不同的处理方式： 
1. QMode = 2，并且_cxq非空：取_cxq队列排头位置的ObjectWaiter对象，调用ExitEpilog方法，该方法会唤醒ObjectWaiter对象的线程，此处会立即返回，后面的代码不会执行了； 
2. QMode = 3，并且_cxq非空：把_cxq队列首元素放入_EntryList的尾部； 
3. QMode = 4，并且_cxq非空：把_cxq队列首元素放入_EntryList的头部； 
4. QMode = 0，不做什么，继续往下看；

只有QMode=2的时候会提前返回，等于0、3、4的时候都会继续往下执行：  

如果_EntryList的首元素非空，就取出来调用ExitEpilog方法，该方法会唤醒ObjectWaiter对象的线程，然后立即返回； 
如果_EntryList的首元素为空，就取_cxq的首元素，放入_EntryList，然后再从_EntryList中取出来执行ExitEpilog方法，然后立即返回；

以上操作，均是执行过ExitEpilog方法然后立即返回，如果取出的元素为空，就执行循环继续取；

小结一下，线程B释放了锁之后，执行的操作如下： 
1. 偏向锁逻辑，此处未命中； 
2. 根据QMode的不同，将ObjectWaiter从_cxq或者_EntryList中取出后唤醒； 
3. 唤醒的元素会继续执行挂起前的代码，按照我们之前的分析，线程唤醒后，就会通过CAS去竞争锁，此时由于线程B已经释放了锁，那么此时应该能竞争成功；

到了现在已经将之前的几个问题搞清了，汇总起来看看： 
1. 线程A在wait() 后被加入了_WaitSet队列中； 
2. 线程C被线程B启动后竞争锁失败，被加入到_cxq队列的首位； 
3. 线程B在notify()时，从_WaitSet中取出第一个，根据Policy的不同，将这个线程放入_EntryList或者_cxq队列中的起始或末尾位置； 
4. 根据QMode的不同，将ObjectWaiter从_cxq或者_EntryList中取出后唤醒；

所以，最初的问题已经清楚了，wait()的线程被唤醒后，会进入一个队列，然后JVM会根据Policy和QMode的不同对队列中的ObjectWaiter做不同的处理，被选中的ObjectWaiter会被唤醒，去竞争锁；  

最开始提出的几个问题(除了第一个问题)上面都已经解释清楚，下面再解释下第一个问题：
    
为什么在调用Object.wait()与notify()之前线程需要持有Object上的锁？
  
因为Object.wait()与notify()设计目的是实现线程间协调控制，这种控制机制是基于锁并且是重量级锁实现的，所以要求在方法调用之前必须获得锁。  

_ _ _
### **Thread.join()源码分析理解**
之所以把join()方法与wait()方法放在一起是因为join()方法本质上是通过wait实现的：  
```java
public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }
   //参数为0，调用Object.wait(0)等待
    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);//参数非0，调用Object.wait(time)等待
            now = System.currentTimeMillis() - base;
        }
    }
}
```
从上面的源码可以看到，逻辑实现比较简单，通过while循环查看线程状态(isAlive()),如果线程活着，调用Object.wait()方法等待；  

wait()方法的实现上面已经介绍的比较清楚，现在的问题是：  
1. 如何判断线程是否活着呢？
2. wait()方法是在什么时候被唤醒的呢？  

#### **如何判断线程是否活着？**
isAlive是个native方法，具体实现在jvm.cpp文件:
```c++
JVM_ENTRY(jboolean, JVM_IsThreadAlive(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_IsThreadAlive");

  oop thread_oop = JNIHandles::resolve_non_null(jthread);
  return java_lang_Thread::is_alive(thread_oop);
JVM_END
```
java_lang_Thread::is_alive在javaClasses.cpp文件定义:
```c++
bool java_lang_Thread::is_alive(oop java_thread) {
  JavaThread* thr = java_lang_Thread::thread(java_thread);
  return (thr != NULL);
}
```
可以看到是通过Thread的eetop找到内部的JavaThread, 如果为NULL,则表示线程已死;
好，第一个问题得到解决。  

#### **wait()方法在什么时候被唤醒的呢？**
在前面的文章深入学习[java并发编程之创建一个线程并start()究竟做了什么？](https://wang-michael.github.io/2018/03/25/%E6%B7%B1%E5%85%A5%E5%AD%A6%E4%B9%A0java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AA%E7%BA%BF%E7%A8%8B%E5%B9%B6start()%E7%A9%B6%E7%AB%9F%E5%81%9A%E4%BA%86%E4%BB%80%E4%B9%88/)的最后，说到:  
> 当run方法执行结束,会调用JavaThread::exit方法清理资源

```c++
//由于原方法较长，删除不相关部分
void JavaThread::exit(bool destroy_vm, ExitType exit_type) {
  ensure_join(this);
  assert(!this->has_pending_exception(), "ensure_join should have cleared");

 // Remove from list of active threads list, and notify VM thread if we are the last non-daemon thread
  Threads::remove(this);
}

static void ensure_join(JavaThread* thread) {
  // 获取Threads_lock
  Handle threadObj(thread, thread->threadObj());
  assert(threadObj.not_null(), "java thread object must exist");
  ObjectLocker lock(threadObj, thread);
  // 忽略pending exception (ThreadDeath)
  thread->clear_pending_exception();
  //设置java.lang.Thread的threadStatus为 TERMINATED.
  java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);
  //清除native线程，这将会导致isAlive()方法返回false 
  java_lang_Thread::set_thread(threadObj(), NULL);
 //通知所有等待thread锁的线程, join的wait方法将返回，由于isActive返回false,join方法将执行结束并返回
  lock.notify_all(thread);
}
```
从上面的源码实现知道，当run方法执行结束，会将native线程对象设为null,并且通过notifyAll方法，让等待在Thread对象锁上的wait方法返回;因此Thread.join常用于等待线程执行完毕，当然如果中间出现异常，join方法也会返回。

**为什么不建议在Thread对象上使用wait()方法？**

我认为原因之一是在Thread对象上wait()的线程可能被Thread.join()方法唤醒，引起想不到的逻辑错误。  

(完)