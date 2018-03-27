---
layout:     post
title:      深入学习java并发编程之Thread.sleep()及Thread.yield()源码分析
date:       2018-3-27
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 并发
    - JDK源码阅读
---
>我们都知道Thread.yield()会释放CPU资源，让优先级更高(至少是相同)的线程获得执行机会；sleep当传入参数为0时，和yield相同，当传入参数大于0时，也是释放CPU资源，只不过此时可以让任何其它优先级的线程获得执行机会。为什么会这样？  

_ _ _
### **Thread.yield()源码分析**
Thread.yield底层是通过JVM_Yield方法实现的（见jvm.cpp）:
```c++
JVM_ENTRY(void, JVM_Yield(JNIEnv *env, jclass threadClass))
  JVMWrapper("JVM_Yield");
  //检查是否设置了DontYieldALot参数,默认为fasle
  //如果设置为true,直接返回
  if (os::dont_yield()) return;
 //如果ConvertYieldToSleep=true（默认为false）,调用os::sleep,否则调用os::yield
  if (ConvertYieldToSleep) {
    os::sleep(thread, MinSleepInterval, false);//sleep 1ms
  } else {
    os::yield();
  }
JVM_END
```
从上面知道，实际上调用的是os::yield:
```c++
//sched_yield是linux kernel提供的API,它会使调用线程放弃CPU使用权，加入到同等优先级队列的末尾；
//如果调用线程是优先级最高的唯一线程，yield方法返回后，调用线程会继续运行；
//因此可以知道，对于和调用线程相同或更高优先级的线程来说，yield方法会给予了它们一次运行的机会；
void os::yield() {
  sched_yield();
}
```

_ _ _
### **Thread.sleep()源码分析**
Thread.sleep最终调用JVM_Sleep方法：
```c++
JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
  JVMWrapper("JVM_Sleep");

  if (millis < 0) {//参数校验
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }

  //如果线程已经中断，抛出中断异常
  if (Thread::is_interrupted (THREAD, true) && !HAS_PENDING_EXCEPTION) {
    THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
  }
 //设置线程状态为SLEEPING
  JavaThreadSleepState jtss(thread);

  EventThreadSleep event;

  if (millis == 0) {
    //如果设置了ConvertSleepToYield(默认为true),和yield效果相同
    if (ConvertSleepToYield) {
      os::yield();
    } else {//否则调用os::sleep方法
      ThreadState old_state = thread->osthread()->get_state();
      thread->osthread()->set_state(SLEEPING);
      os::sleep(thread, MinSleepInterval, false);//sleep 1ms
      thread->osthread()->set_state(old_state);
    }
  } else {//参数大于0
   //保存初始状态，返回时恢复原状态
    ThreadState old_state = thread->osthread()->get_state();
    //osthread->thread status mapping:
    // NEW->NEW
    //RUNNABLE->RUNNABLE
    //BLOCKED_ON_MONITOR_ENTER->BLOCKED
    //IN_OBJECT_WAIT,PARKED->WAITING
    //SLEEPING,IN_OBJECT_WAIT_TIMED,PARKED_TIMED->TIMED_WAITING
    //TERMINATED->TERMINATED
    thread->osthread()->set_state(SLEEPING);
    //调用os::sleep方法，如果发生中断，抛出异常
    if (os::sleep(thread, millis, true) == OS_INTRPT) {
      if (!HAS_PENDING_EXCEPTION) {
        if (event.should_commit()) {
          event.set_time(millis);
          event.commit();
        }
        THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
      }
    }
    thread->osthread()->set_state(old_state);//恢复osThread状态
  }
  if (event.should_commit()) {
    event.set_time(millis);
    event.commit();
  }
JVM_END
```
从中我们可以看到当sleep()方法中传入参数为0时，默认调用的其实也是os::yield()方法，与yield()方法完全相同；当参数大于0时，才会调用真正的os::sleep方法进行睡眠。  

os::sleep的源码如下：
```c++
int os::sleep(Thread* thread, jlong millis, bool interruptible) {
  assert(thread == Thread::current(),  "thread consistency check");
  //线程有如下几个成员变量:
  //ParkEvent * _ParkEvent ;          // for synchronized()
  //ParkEvent * _SleepEvent ;        // for Thread.sleep
  //ParkEvent * _MutexEvent ;      // for native internal Mutex/Monitor
  //ParkEvent * _MuxEvent ;         // for low-level muxAcquire-muxRelease
  ParkEvent * const slp = thread->_SleepEvent ;
  slp->reset() ;
  OrderAccess::fence() ;

//如果millis>0,传入interruptible＝true,否则为false
  if (interruptible) {
    jlong prevtime = javaTimeNanos();

    for (;;) {
      if (os::is_interrupted(thread, true)) {//判断是否中断
        return OS_INTRPT;
      }

      jlong newtime = javaTimeNanos();//获取当前时间
      //如果linux不支持monotonic lock,有可能出现newtime<prevtime
      if (newtime - prevtime < 0) {
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
        OSThreadWaitState osts(jt->osthread(), false );

        jt->set_suspend_equivalent();
        slp->park(millis);
        jt->check_and_wait_while_suspended();
      }
    }
  } else {//如果interruptible=false
   //设置osthread的状态为CONDVAR_WAIT
    OSThreadWaitState osts(thread->osthread(), false );
    jlong prevtime = javaTimeNanos();

    for (;;) {
      jlong newtime = javaTimeNanos();

      if (newtime - prevtime < 0) {
        assert(!Linux::supports_monotonic_clock(), "time moving backwards");
      } else {
        millis -= (newtime - prevtime) / NANOSECS_PER_MILLISEC;
      }

      if(millis <= 0) break ;

      prevtime = newtime;
      slp->park(millis);//底层调用pthread_cond_timedwait实现
    }
    return OS_OK ;
  }
}
```

(完）

转载于：[Java线程源码解析之yield和sleep](https://www.jianshu.com/p/0964124ae822)
