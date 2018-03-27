---
layout:     post
title:      深入学习java并发编程之创建一个线程并start()究竟做了什么？
date:       2018-3-25
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:    
    - 并发
    - JDK源码阅读
---
>本文关于Thread.start0() native方法源码分析的过程来源于文章[Java线程源码解析之start](https://www.jianshu.com/p/81a56497e073)，在文中部分地方加入了自己对于原作者文章中内容的理解，仅用于备忘。    

_ _ _
### **前言**  
创建一个线程并启动可以分为两个步骤，一是创建一个Thread类对象，二是调用其start方法，线程启动之后就会调用它的run()方法运行。java线程在linux上的实现是映射到底层os上的，那么java线程映射的底层os线程(或者说是轻量级进程)是什么时候创建的呢？创建之后又是怎么调用我们创建线程时提供的run()方法的呢？  

_ _ _
### **是在new Thread()的时候创建的吗？**  
new Thread()除了初始化相关成员变量之外所做的主要工作就是执行下面这个init方法：
```java
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    this.name = name;

    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
    if (g == null) {
        /* Determine if it's an applet or not */

        /* If there is a security manager, ask the security manager
           what to do. */
        if (security != null) {
            g = security.getThreadGroup();
        }

        /* If the security doesn't have a strong opinion of the matter
           use the parent thread group. */
        if (g == null) {
            g = parent.getThreadGroup();
        }
    }

    /* checkAccess regardless of whether or not threadgroup is
       explicitly passed in. */
    g.checkAccess();

    /*
     * Do we have the required permissions?
     */
    if (security != null) {
        if (isCCLOverridden(getClass())) {
            security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
    }

    g.addUnstarted();

    this.group = g;
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;
    this.inheritedAccessControlContext =
            acc != null ? acc : AccessController.getContext();
    this.target = target;
    setPriority(priority);
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    /* Stash the specified stack size in case the VM cares */
    this.stackSize = stackSize;

    /* Set thread ID */
    tid = nextThreadID();
}
```
可以看到这个方法中只是对Thread对象一些属性进行了设置(priority、group、daemon、stackSize)，与普通java对象创建并没有什么区别，并没有涉及到os底层创建线程等操作。所以可以猜测底层依赖的os线程的创建和一些其它工作是在Thread.start()中完成的。    

_ _ _
### **是在Thread.start()的时候创建的吗？** 
从start()方法的java代码开始看起：  
```java
public synchronized void start() {
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    group.add(this);
    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
        }
    }
}

private native void start0();
```
可以看到主要的逻辑都是通过native方法star0实现的， 查看Thread.c文件可以知道，它实际上调用的是jvm.cpp文件的JVM_StartThread方法:
```c++
// hotspot\src\share\vm\prims\jvm.cpp
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  bool throw_illegal_thread_state = false;
  {
   // Threads_lock代表在活动线程表上的锁，MutexLocker会调用lock方法上锁
    MutexLocker mu(Threads_lock);
    //实际上是判断java.lang.Thread的eetop,正常情况下，在后续步骤中会赋值，但在此处为0，不指向任何对象;
    //其实在start方法中已经根据threadStatus进行了判断，但是由于创建线程对象和更新threadStatus并不是原子操作，因而再次check
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;//状态错误，返回
    } else {
     //在java.lang.Thread的init方法中，可设置stack的大小，此处获取设置的大小；
     //不过通常调用构造函数的时候都不会传入stack大小，size=0
      jlong size =
        java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      size_t sz = size > 0 ? (size_t) size : 0;
     //在下面的［创建JavaThread］介绍，完成的功能实际是创建子线程并等待其完成初始化
      native_thread = new JavaThread(&thread_entry, sz);
      if (native_thread->osthread() != NULL) {
        native_thread->prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

  assert(native_thread != NULL, "Starting null thread?");

  //Java线程实际上是通过系统线程实现的，如果创建系统线程失败，报错；
  //有很多原因会导致该错误：比如内存不足、max user processes设置过小 
  if (native_thread->osthread() == NULL) {
    delete native_thread;
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        "unable to create new native thread");
    }
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              "unable to create new native thread");
  }
  //在下面的［创建JavaThread］介绍,完成的功能实际是唤醒创建的子线程令其执行run()方法
  Thread::start(native_thread);

JVM_END
```
### **创建JavaThread**  
```c++
// hotspot\src\share\vm\runtime\thread.cpp
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
  Thread()
#ifndef SERIALGC
  , _satb_mark_queue(&_satb_mark_queue_set),
  _dirty_card_queue(&_dirty_card_queue_set)
#endif // !SERIALGC
{
  if (TraceThreadEvents) {
    tty->print_cr("creating thread %p", this);
  }
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  set_entry_point(entry_point);
 
  os::ThreadType thr_type = os::java_thread;
  //根据entry_point判断是CompilerThread还是JavaThread
  //由于此处传入的为&thread_entry,因此为os::java_thread
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                             os::java_thread;
 // os线程有可能创建失败，在上文已经看到对该场景的处理
  os::create_thread(this, thr_type, stack_sz);
  _safepoint_visible = false;
}
```
接下来看JavaThread中的os::create_thread方法：  
```c++
// \hotspot\src\os\linux\vm\os_linux.cpp
bool os::create_thread(Thread* thread, ThreadType thr_type, size_t stack_size) {
  assert(thread->osthread() == NULL, "caller responsible");

  // 创建OSThread
  OSThread* osthread = new OSThread(NULL, NULL);
  if (osthread == NULL) {
    return false;
  }
  //设置线程类型
  osthread->set_thread_type(thr_type);

  // 初始化状态为ALLOCATED
  osthread->set_state(ALLOCATED);
  thread->set_osthread(osthread);
  //linux下可以通过pthread_attr_t来设置线程属性
  pthread_attr_t attr;
  pthread_attr_init(&attr);//linux系统调用，更多内容请参考内核文档
  pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

  // 线程栈大小
  if (os::Linux::supports_variable_stack_size()) {//是否支持设置栈大小
    //如果用户创建线程时未指定栈大小,对于JavaThread会看是否设置了-Xss或ThreadStackSize；
    //如果未设置，则采用系统默认值。对于64位操作系统，默认为1M;
    //操作系统栈大小（ulimit -s）：这个配置只影响进程的初始线程；后续用pthread_create创建的线程都可以指定栈大小。
    //HotSpot VM为了能精确控制Java线程的栈大小，特意不使用进程的初始线程（primordial thread）作为Java线程
    if (stack_size == 0) {
      stack_size = os::Linux::default_stack_size(thr_type);
      switch (thr_type) {
      case os::java_thread:
      //读取Xss和ThreadStackSize
        assert (JavaThread::stack_size_at_create() > 0, "this should be set");
        stack_size = JavaThread::stack_size_at_create();
        break;
      case os::compiler_thread:
        if (CompilerThreadStackSize > 0) {
          stack_size = (size_t)(CompilerThreadStackSize * K);
          break;
        } // else fall through:
          // use VMThreadStackSize if CompilerThreadStackSize is not defined
      case os::vm_thread:
      case os::pgc_thread:
      case os::cgc_thread:
      case os::watcher_thread:
        if (VMThreadStackSize > 0) stack_size = (size_t)(VMThreadStackSize * K);
        break;
      }
    }
   //栈最小为48k
    stack_size = MAX2(stack_size, os::Linux::min_stack_allowed);
    pthread_attr_setstacksize(&attr, stack_size);
  } else {
    // let pthread_create() pick the default value.
  }

  pthread_attr_setguardsize(&attr, os::Linux::default_guard_size(thr_type));

  ThreadState state;
  {
   //如果linux线程而且不支持设置栈大小，则先获取创建线程锁，获取锁之后再创建线程
    bool lock = os::Linux::is_LinuxThreads() && !os::Linux::is_floating_stack();
    if (lock) {
      os::Linux::createThread_lock()->lock_without_safepoint_check();
    }

    pthread_t tid;
    //调用linux的pthread_create创建线程,传入4个参数
    //第一个参数：指向线程标示符pthread_t的指针；
    //第二个参数：设置线程的属性
    //第三个参数：线程运行函数的起始地址
    //第四个参数：运行函数的参数
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);
    pthread_attr_destroy(&attr);
    if (ret != 0) {//创建失败,做清理工作
      if (PrintMiscellaneous && (Verbose || WizardMode)) {
        perror("pthread_create()");
      }
      // Need to clean up stuff we've allocated so far
      thread->set_osthread(NULL);
      delete osthread;
      if (lock) os::Linux::createThread_lock()->unlock();
      return false;
    }

    // 将pthread id保存到osthread
    osthread->set_pthread_id(tid);

    // 等待pthread_create创建的子线程完成初始化或放弃，重点！！！
    {
      Monitor* sync_with_child = osthread->startThread_lock();
      MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
      while ((state = osthread->get_state()) == ALLOCATED) {
        sync_with_child->wait(Mutex::_no_safepoint_check_flag);
      }
    }

    if (lock) {
      os::Linux::createThread_lock()->unlock();
    }
  }

  // Aborted due to thread limit being reached
  if (state == ZOMBIE) {
      thread->set_osthread(NULL);
      delete osthread;
      return false;
  }

  // The thread is returned suspended (in state INITIALIZED),
  // and is started higher up in the call chain
  assert(state == INITIALIZED, "race condition");
  return true;
}
```
创建线程时传入了java_start,做为线程运行函数的初始地址,规定了创建好的子进程从哪开始运行:  
```c++
static void *java_start(Thread *thread) {
  static int counter = 0;
  int pid = os::current_process_id();
 //alloca是用来分配存储空间的，它和malloc的区别是它是在当前函数的栈上分配存储空间，而不是在堆中。
 //其优点是：当函数返回时，自动释放它所使用的栈。
  alloca(((pid ^ counter++) & 7) * 128);

  ThreadLocalStorage::set_thread(thread);

  OSThread* osthread = thread->osthread();
  Monitor* sync = osthread->startThread_lock();

  // non floating stack LinuxThreads needs extra check, see above
  if (!_thread_safety_check(thread)) {
    // notify parent thread
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);
    osthread->set_state(ZOMBIE);
    sync->notify_all();
    return NULL;
  }

  // thread_id is kernel thread id (similar to Solaris LWP id)
  osthread->set_thread_id(os::Linux::gettid());

 //优先尝试在请求线程当前所处的CPU的Local内存上分配空间。
 //如果local内存不足，优先淘汰local内存中无用的Page
  if (UseNUMA) {//默认为false
    int lgrp_id = os::numa_get_group_id();
    if (lgrp_id != -1) {
      thread->set_lgrp_id(lgrp_id);
    }
  }
  // 调用pthread_sigmask初始化signal mask:VM线程处理BREAK_SIGNAL信号
  os::Linux::hotspot_sigmask(thread);

  // initialize floating point control register
  os::Linux::init_thread_fpu_state();

  // handshaking with parent thread
  {
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);

    // 设置状态会INITIALIZED,并通过notify_all唤醒父线程
    osthread->set_state(INITIALIZED);
    sync->notify_all();

    // 一直等待父线程调用 os::start_thread()
    while (osthread->get_state() == INITIALIZED) {
      sync->wait(Mutex::_no_safepoint_check_flag);
    }
  }

  // call one more level start routine
  thread->run();

  return 0;
}
```
当子线程完成初始化，将状态设置为NITIALIZED并唤醒父线程之后，父线程会执行Thread::start方法:
```c++
void Thread::start(Thread* thread) {
// \hotspot\src\share\vm\runtime\thread.cpp
  trace("start", thread);
  if (!DisableStartThread) {
    if (thread->is_Java_thread()) {//设置线程状态为RUNNABLE
      java_lang_Thread::set_thread_status(((JavaThread*)thread)->threadObj(),
                                          java_lang_Thread::RUNNABLE);
    }
    os::start_thread(thread);
  }
}
```
```c++
void os::start_thread(Thread* thread) {
  MutexLockerEx ml(thread->SR_lock(), Mutex::_no_safepoint_check_flag);
  OSThread* osthread = thread->osthread();
//设置线程状态为RUNNABLE, 子线程可以开始执行thread->run()
  osthread->set_state(RUNNABLE);
  pd_start_thread(thread);
}

// \hotspot\src\os\linux\vm\os_linux.cpp
void os::pd_start_thread(Thread* thread) {
  OSThread * osthread = thread->osthread();
  assert(osthread->get_state() != INITIALIZED, "just checking");
  Monitor* sync_with_child = osthread->startThread_lock();
  MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
  sync_with_child->notify();//唤醒子线程后，子线程继续执行会调用thread->run();
}
```
子线程的thread->run主要逻辑为调用thread_main_inner,源码如下：
```c++
void JavaThread::thread_main_inner() {//删除部分非关键代码
  if (!this->has_pending_exception() &&
      !java_lang_Thread::is_stillborn(this->threadObj())) {
    {
      ResourceMark rm(this);
      this->set_native_thread_name(this->get_thread_name());
    }
    HandleMark hm(this);
    this->entry_point()(this, this);
  }
  this->exit(false);
  delete this;
}
```
那这儿的entry_point是什么呢？实际上在创建JavaThread时，会传入entrypoint:  
```c++
static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread->threadObj());
  JavaValue result(T_VOID);
  JavaCalls::call_virtual(&result,
                          obj,
                          KlassHandle(THREAD, SystemDictionary::Thread_klass()),
                          vmSymbols::run_method_name(),
                          vmSymbols::void_method_signature(),
                          THREAD);
}
```
可以看到实际上就是调用java.lang.Thread的run方法;

另外当run方法执行结束,会调用JavaThread::exit方法清理资源。    

具体父子线程协作流程图如下：  
<img src="/img/2018-3-25/img1.jpg" width="660" height="660" alt="源码" />

(完)