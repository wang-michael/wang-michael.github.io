---
layout:     post
title:      深入学习java并发编程之synchronized实现原理
date:       2018-3-23
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - java基础
    - 并发
---
>本文大部分内容来源于[jdk源码剖析二: 对象内存布局、synchronized终极原理](http://www.cnblogs.com/dennyzhangdd/p/6734638.html)，在文中部分地方加入了自己对于原作者文章中内容的理解，仅用于备忘。    

_ _ _
### **启蒙知识预热**
开启本文之前先介绍2个概念。
#### **CAS操作**
为了提高性能，JVM很多操作都依赖CAS实现，一种乐观锁的实现。本文锁优化中大量用到了CAS，故有必要先分析一下CAS的实现。  

CAS即Compare and Swap，例如JNI来完成CPU指令的操作：unsafe.compareAndSwapInt(this, valueOffset, expect, update);  

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。如果A=V，那么把B赋值给V，返回V；如果A！=V，直接返回V。  

打开源码：openjdk\hotspot\src\oscpu\windowsx86\vm\ atomicwindowsx86.inline.hpp，如下图：<img src="/img/2018-3-23/img1.png" width="660" height="660" alt="源码" />
  
os::is_MP()  这个是runtime/os.hpp，实际就是返回是否多处理器，源码如下：
<img src="/img/2018-3-23/img2.png" width="500" height="500" alt="源码" />

如上面源代码所示（看第一个int参数即可），LOCK_IF_MP:会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。  

#### **对象头**  
HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。 

HotSpot虚拟机的对象头(Object Header)包括两部分信息:
* 第一部分"Mark Word":用于存储对象自身的运行时数据， 如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等.
* 第二部分"Klass Pointer"：对象指向它的类的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。(数组，对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中无法确定数组的大小。 ) 

32位的HotSpot虚拟机对象头存储结构：
<img src="/img/2018-3-23/img3.jpg" width="600" height="600" alt="源码" />  

为了证实上图的正确性，这里我们看openJDK--》hotspot源码markOop.hpp，虚拟机对象头存储结构：
<img src="/img/2018-3-23/img4.jpg" width="660" height="660" alt="源码" />  

上图中相关单词解释：   
* hash： 保存对象的哈希码
* age： 保存对象的分代年龄
* biased_lock： 偏向锁标识位
* lock： 锁状态标识位
* JavaThread*： 保存持有偏向锁的线程ID
* epoch： 保存偏向时间戳

上图中有源码中对锁标志位这样枚举：  
<img src="/img/2018-3-23/img5.png" width="660" height="660" alt="源码" />
  
下面是源码注释：
<img src="/img/2018-3-23/img6.png" width="660" height="660" alt="源码" />

看上图，不管是32/64位JVM，都是1bit偏向锁+2bit锁标志位。上面红框是偏向锁（第一行是指向线程的显示偏向锁，第二行是匿名偏向锁）对应枚举biased_lock_pattern，下面红框是轻量级锁、无锁、监视器锁、GC标记，分别对应上面的前4种枚举。我们甚至能看见锁标志11时，是GC的markSweep(标记清除算法)使用的。（这里就不再拓展了）

对象头中的Mark Word，synchronized源码实现就用了Mark Word来标识对象加锁状态。  

_ _ _
### **JVM中synchronized锁实现原理（优化）**
大家都知道jdk1.6之前java中锁synchronized性能较差，当存在竞争的时候，线程会直接进入阻塞状态。本节将以图文形式来描述JVM的synchronized锁优化。  

在jdk1.6中对锁的实现引入了大量的优化来减少锁操作的开销：  

* 锁粗化（Lock Coarsening）：将多个连续的锁扩展成一个范围更大的锁，用以减少频繁互斥同步导致的性能损耗。  
* 锁消除（Lock Elimination）：JVM及时编译器在运行时，通过逃逸分析，如果判断一段代码中，堆上的所有数据不会逃逸出去从来被其他线程访问到，就可以去除这些锁。  
* 轻量级锁（Lightweight Locking）：JDK1.6引入。在没有多线程竞争的情况下避免重量级互斥锁，只需要依靠一条CAS原子指令就可以完成锁的获取及释放。  
* 偏向锁（Biased Locking）：JDK1.6引入。目的是消除数据再无竞争情况下的同步原语。使用CAS记录获取它的线程。下一次同一个线程进入则偏向该线程，无需任何同步操作。  
* 适应性自旋（Adaptive Spinning）：为了避免线程频繁挂起、恢复的状态切换消耗。产生了忙循环（循环时间固定），即自旋。JDK1.6引入了自适应自旋。自旋时间根据之前锁自旋时间和线程状态，动态变化，用以期望能减少阻塞的时间。
  
锁升级：偏向锁--》轻量级锁--》重量级锁；锁升级之后就不能再降级，但可以选择恢复到无锁状态。 
#### **偏向锁**   
HotSpot的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。  

因此偏向锁的想法是一旦线程第一次获得了监视对象，之后让监视对象“偏向”这个线程，之后的多次调用则可以避免CAS操作。  

简单的讲，就是在锁对象的对象头（开篇讲的对象头数据存储结构）中有个ThreaddId字段，这个字段如果是空的，第一次获取锁的时候，就将自身的ThreadId写入到锁的ThreadId字段内，将锁头内的是否偏向锁的状态位置1.这样下次获取锁的时候，直接检查ThreadId是否和自身线程Id一致，如果一致，则认为当前线程已经获取了锁，因此不需再次获取锁，略过了轻量级锁和重量级锁的加锁阶段。提高了效率。

**注意：**当锁有竞争关系的时候，需要解除偏向锁，进入轻量级锁。  

每一个线程在准备获取共享资源时：  

**第一步，检查MarkWord里面是不是放的自己的ThreadId ,如果是，表示当前线程是处于 “偏向锁”.跳过轻量级锁直接执行同步体。**  

偏向锁初始化流程如下图：  
<img src="/img/2018-3-23/img7.jpg" width="660" height="660" alt="源码" />

#### **轻量级锁和重量级锁**   
<img src="/img/2018-3-23/img8.jpg" width="660" height="660" alt="源码" />

如上图所示：  

**第二步，如果MarkWord不是自己的ThreadId,锁升级，这时候，用CAS来执行切换，新的线程根据MarkWord里面现有的ThreadId，通知之前线程暂停，之前线程将Markword的内容置为空。**  

**第三步，两个线程都把对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作，把共享对象的MarKword的内容修改为自己新建的记录空间的地址的方式竞争MarkWord.**  

**第四步，第三步中成功执行CAS的获得资源，失败的则进入自旋.**  

**第五步，自旋的线程在自旋过程中，成功获得资源(即之前获得资源的线程执行完成并释放了共享资源)，则整个状态依然处于轻量级锁的状态，如果自旋失败 第六步，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己.**  

**注意点：**JVM加锁流程

偏向锁--》轻量级锁--》重量级锁，从左往右可以升级，从右往左不能降级。

* 偏向锁升级成轻量级锁一种可能是当前锁对象并没有偏向任何一个线程，此时多个线程同时进行CAS操作试图让锁偏向自己，这时CAS操作失败的线程就会意识到存在竞争，尝试将锁升级为轻量级锁。还有一种可能是在第二个线程访问同步块或方法，发现当前锁是偏向锁且指向线程一，因此线程二会暂停线程一，通知其撤销偏向锁；之后线程一撤销偏向锁的过程中可能将对象头中markword标志位置为0，表示在此对象上不适合使用偏向锁，之后线程二加锁时就会使用轻量级锁，也就是完成了从偏向锁到轻量级锁的升级过程。  
* 轻量级锁升级成重量级锁可能是这种情况：线程一持有对象的轻量级锁并且在执行同步块内的代码，线程二到来发现锁已被线程一获取，于是进行自旋，当达到自旋的最大时间时，线程二将锁修改为重量级锁，线程二陷入阻塞状态，并且在线程一执行同步块代码期间到来的所有线程也都会陷入阻塞状态；之后线程一同步块代码执行结束，使用CAS替换markword准备释放锁，但是此时发现markword中存储的值与自己之前存入的值不一样了(因为线程二进行了修改)，所以线程一在释放锁之后要进行的另一个操作就是唤醒阻塞在这个锁上的等待队列中的所有线程，进行新一轮的夺锁之争。  

下图展示了这几种锁的优缺点对比：    
<img src="/img/2018-3-23/img9.jpg" width="660" height="660" alt="源码" />

_ _ _
### **synchronized  C++源码**
前两节讲了synchronized锁实现原理，这一节我们从C++源码来剖析synchronized。 重点来了，之前在第一节中的图1，看过了对象头Mark Word。现在我们从C++源码来剖析具体的数据结构和获取释放锁的过程。  

#### **C++中的监视器锁数据结构**  
oopDesc--继承-->markOopDesc--方法monitor()-->ObjectMonitor-->enter、exit 获取、释放锁。  
  
(1)oopDesc类  

openjdk\hotspot\src\share\vm\oops\oop.hpp下oopDesc类是JVM对象的顶级基类,故每个object都包含markOop。如下图所示：  
```c++
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;//markOop:Mark Word标记字段
  union _metadata {
    Klass*      _klass;//对象类型元数据的指针
    narrowKlass _compressed_klass;
  } _metadata;

  // Fast access to barrier set.  Must be initialized.
  static BarrierSet* _bs;

 public:
  markOop  mark() const         { return _mark; }
  markOop* mark_addr() const    { return (markOop*) &_mark; }

  void set_mark(volatile markOop m)      { _mark = m;   }

  void    release_set_mark(markOop m);
  markOop cas_set_mark(markOop new_mark, markOop old_mark);

  // Used only to re-initialize the mark word (e.g., of promoted
  // objects during a GC) -- requires a valid klass pointer
  void init_mark();

  Klass* klass() const;
  Klass* klass_or_null() const volatile;
  Klass** klass_addr();
  narrowKlass* compressed_klass_addr();
....省略...
}
```
(2)markOopDesc类  
 
openjdk\hotspot\src\share\vm\oops\markOop.hpp下markOopDesc继承自oopDesc，并拓展了自己的方法monitor(),如下图:  

```c++
ObjectMonitor* monitor() const {
   assert(has_monitor(), "check");
   // Use xor instead of &~ to provide one extra tag-bit check.
   return (ObjectMonitor*) (value() ^ monitor_value);
}
```  
该方法返回一个ObjectMonitor*对象指针。其中value()这样定义：  
```c++
uintptr_t value() const { return (uintptr_t) this; }
```
可知：将this转换成一个指针宽度的整数（uintptr_t），然后进行"异或"位操作。  

monitor_value是常量:  
```c++
enum { 
    locked_value             = 0,//00偏向锁 
    unlocked_value           = 1,//01无锁
    monitor_value            = 2,//10监视器锁，又叫重量级锁
    marked_value             = 3,//11GC标记
    biased_lock_pattern      = 5 //101偏向锁
};
```
指针低2位00，异或10，结果还是10.（拿一个模板为00的数，异或一个二位数=数本身。因为异或：“相同为0，不同为1”.操作）

(3)ObjectMonitor类  

在HotSpot虚拟机中，最终采用ObjectMonitor类实现monitor。openjdk\hotspot\src\share\vm\runtime\objectMonitor.hpp源码如下：  
```c++
ObjectMonitor() {
_header       = NULL;//markOop对象头
_count        = 0;
_waiters      = 0,//等待线程数
_recursions   = 0;//重入次数
_object       = NULL;//监视器锁寄生的对象。锁不是平白出现的，而是寄托存储于对象中。
_owner        = NULL;//指向获得ObjectMonitor对象的线程或基础锁
_WaitSet      = NULL;//处于wait状态的线程，会被加入到wait set；
_WaitSetLock  = 0 ;
_Responsible  = NULL ;
_succ         = NULL ;
_cxq          = NULL ;
FreeNext      = NULL ;
_EntryList    = NULL ;//处于等待锁block状态的线程，会被加入到entry set；
_SpinFreq     = 0 ;
_SpinClock    = 0 ;
OwnerIsThread = 0 ;// _owner is (Thread *) vs SP/BasicLock
_previous_owner_tid = 0;// 监视器前一个拥有者线程的ID
}
```
每个线程都有两个ObjectMonitor对象列表，分别为free和used列表，如果当前free列表为空，线程将向全局global list请求分配ObjectMonitor。  

ObjectMonitor对象中有两个队列：_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表；  
<img src="/img/2018-3-23/img10.png" width="660" height="660" alt="源码" />

#### **获取锁流程** 
**synchronized关键字修饰的代码段，在JVM被编译为monitorenter、monitorexit指令来获取和释放互斥锁。所以要想了解synchronized的实现原理，我们就要知道这两个指令是如何被JVM解释执行的。**    

解释器执行monitorenter时会进入到InterpreterRuntime.cpp的InterpreterRuntime::monitorenter函数，具体实现如下：
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
  if (UseBiasedLocking) {//标识虚拟机是否开启偏向锁功能,默认开启
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    // 调用获取偏向锁的方法
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
先看一下入参：
* JavaThread thread指向java中的当前线程；
* BasicObjectLock基础对象锁：包含一个BasicLock和一个指向Object对象的指针oop。

openjdk\hotspot\src\share\vm\runtime\basicLock.hpp中BasicObjectLock类源码如下：
```c++
class BasicObjectLock VALUE_OBJ_CLASS_SPEC {
  friend class VMStructs;
 private:
  BasicLock _lock;                                    // the lock, must be double word aligned
  oop       _obj;                                     // object holds the lock;

 public:
  // Manipulation
  oop      obj() const                                { return _obj;  }
  void set_obj(oop obj)                               { _obj = obj; }
  BasicLock* lock()                                   { return &_lock; }

  // Note: Use frame::interpreter_frame_monitor_size() for the size of BasicObjectLocks
  //       in interpreter activation frames since it includes machine-specific padding.
  static int size()                                   { return sizeof(BasicObjectLock)/wordSize; }

  // GC support
  void oops_do(OopClosure* f) { f->do_oop(&_obj); }

  static int obj_offset_in_bytes()                    { return offset_of(BasicObjectLock, _obj);  }
  static int lock_offset_in_bytes()                   { return offset_of(BasicObjectLock, _lock); }
};
```

BasicLock类型_lock对象主要用来保存：指向Object对象的对象头数据；basicLock.hpp中BasicLock源码如下：  
```c++
class BasicLock VALUE_OBJ_CLASS_SPEC {
  friend class VMStructs;
 private:
  volatile markOop _displaced_header;//markOop是不是很熟悉？1.2节中讲解对象头时就是分析的markOop源码
 public:
  markOop      displaced_header() const               { return _displaced_header; }
  void         set_displaced_header(markOop header)   { _displaced_header = header; }

  void print_on(outputStream* st) const;

  // move a basic lock (used during deoptimization
  void move_to(oop obj, BasicLock* dest);

  static int displaced_header_offset_in_bytes()       { return offset_of(BasicLock, _displaced_header); }
};
```
#### **偏向锁的获取ObjectSynchronizer::fast_enter** 
在HotSpot中，偏向锁的入口位于openjdk\hotspot\src\share\vm\runtime\synchronizer.cpp文件的ObjectSynchronizer::fast_enter函数：  
```java
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
 }
 //轻量级锁
 slow_enter (obj, lock, THREAD) ;
}
```
偏向锁的获取由BiasedLocking::revoke_and_rebias方法实现，由于实现比较长，就不贴代码了，实现逻辑如下：     

1、通过markOop mark = obj->mark()获取对象的markOop数据mark，即对象头的Mark Word；  
2、判断mark是否为可偏向状态，即mark的偏向锁标志位为 1，锁标志位为 01；  
3、判断mark中JavaThread的状态：如果为空，则进入步骤（4）；如果指向当前线程，则执行同步代码块；如果指向其它线程，进入步骤（5）；  
4、通过CAS原子指令设置mark中JavaThread为当前线程ID，如果执行CAS成功，则执行同步代码块，否则进入步骤（5）；  
5、进入此步骤有两种情况：一是如执行CAS失败，表示当前存在多个线程竞争想让锁偏向自己，二是一个线程到达时发现锁已经偏向了另一个线程；这两种情况的解决方式为当达到全局安全点（safepoint），获得偏向锁的线程被挂起，撤销偏向锁，并升级为轻量级，升级完成后被阻塞在安全点的线程继续执行同步代码块；    

#### **偏向锁的撤销** 
只有当其它线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，偏向锁的撤销由BiasedLocking::revoke_at_safepoint方法实现：     
```c++
void BiasedLocking::revoke_at_safepoint(Handle h_obj) {
  assert(SafepointSynchronize::is_at_safepoint(), "must only be called while at safepoint");//校验全局安全点
  oop obj = h_obj();
  HeuristicsResult heuristics = update_heuristics(obj, false);
  if (heuristics == HR_SINGLE_REVOKE) {
    revoke_bias(obj, false, false, NULL);
  } else if ((heuristics == HR_BULK_REBIAS) ||
             (heuristics == HR_BULK_REVOKE)) {
    bulk_revoke_or_rebias_at_safepoint(obj, (heuristics == HR_BULK_REBIAS), false, NULL);
  }
  clean_up_cached_monitor_info();
}
```
1、偏向锁的撤销动作必须等待全局安全点；  
2、暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态；  
3、撤销偏向锁，恢复到无锁（标志位为 01）或轻量级锁（标志位为 00）的状态；  

偏向锁在Java 1.6之后是默认启用的，但在应用程序启动几秒钟之后才激活，可以使用-XX:BiasedLockingStartupDelay=0参数关闭延迟，如果确定应用程序中所有锁通常情况下处于竞争状态，可以通过XX:-UseBiasedLocking=false参数关闭偏向锁。   

#### **轻量级锁的获取**  
当关闭偏向锁功能，或多个线程竞争偏向锁导致偏向锁升级为轻量级锁，会尝试获取轻量级锁，其入口位于ObjectSynchronizer::slow_enter：  
```c++
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  if (mark->is_neutral()) {//是否为无锁状态001
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    lock->set_displaced_header(mark);
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {//CAS成功，释放栈锁
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }
    // Fall through to inflate() ...
  } else
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
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
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
```
1、markOop mark = obj->mark()方法获取对象的markOop数据mark；  
2、mark->is_neutral()方法判断mark是否为无锁状态：mark的偏向锁标志位为 0，锁标志位为 01；  
3、如果mark处于无锁状态，则进入步骤（4），否则执行步骤（6）；  
4、把mark保存到BasicLock对象的_displaced_header字段；  
5、通过CAS尝试将Mark Word更新为指向BasicLock对象的指针，如果更新成功，表示竞争到锁，则执行同步代码，否则执行步骤（6）；  
6、如果当前mark处于加锁状态，且mark中的ptr指针指向当前线程的栈帧，则执行同步代码，否则说明有多个线程竞争轻量级锁，轻量级锁需要膨胀升级为重量级锁；    

假设线程A和B同时执行到临界区if (mark->is_neutral())：  

1、线程AB都把Mark Word复制到各自的_displaced_header字段，该数据保存在线程的栈帧上，是线程私有的；  
2、Atomic::cmpxchg_ptr原子操作保证只有一个线程可以把指向栈帧的指针复制到Mark Word，假设此时线程A执行成功，并返回继续执行同步代码块；  
3、线程B执行失败，退出临界区，通过ObjectSynchronizer::inflate方法开始膨胀锁；  

#### **轻量级锁的释放**   
轻量级锁的释放通过ObjectSynchronizer::slow_exit--->调用ObjectSynchronizer::fast_exit完成。  
```c++
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
  assert(!object->mark()->has_bias_pattern(), "should not see bias pattern here");
  // if displaced header is null, the previous enter is recursive enter, no-op
  markOop dhw = lock->displaced_header();
  markOop mark ;
  if (dhw == NULL) {
     // Recursive stack-lock.
     // Diagnostics -- Could be: stack-locked, inflating, inflated.
     mark = object->mark() ;
     assert (!mark->is_neutral(), "invariant") ;
     if (mark->has_locker() && mark != markOopDesc::INFLATING()) {
        assert(THREAD->is_lock_owned((address)mark->locker()), "invariant") ;
     }
     if (mark->has_monitor()) {
        ObjectMonitor * m = mark->monitor() ;
        assert(((oop)(m->object()))->mark() == mark, "invariant") ;
        assert(m->is_entered(THREAD), "invariant") ;
     }
     return ;
  }

  mark = object->mark() ;

  // If the object is stack-locked by the current thread, try to
  // swing the displaced header from the box back to the mark.
  if (mark == (markOop) lock) {
     assert (dhw->is_neutral(), "invariant") ;
     if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {//成功的释放了锁
        TEVENT (fast_exit: release stacklock) ;
        return;
     }
  }

  ObjectSynchronizer::inflate(THREAD, object)->exit (true, THREAD) ;//锁膨胀升级
}
```
1、确保处于偏向锁状态时不会执行这段逻辑；  
2、取出在获取轻量级锁时保存在BasicLock对象的mark数据dhw；  
3、通过CAS尝试把dhw替换到当前的Mark Word，如果CAS成功，说明成功的释放了锁，否则执行步骤（4）；  
4、如果CAS失败，说明有其它线程在尝试获取该锁，这时需要将该锁升级为重量级锁，并释放；    

#### **重量级锁**
重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。  

**锁膨胀过程：**锁的膨胀过程通过ObjectSynchronizer::inflate函数实现。  
```c++
ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
  // Inflate mutates the heap ...
  // Relaxing assertion for bug 6320749.
  assert (Universe::verify_in_progress() ||
          !SafepointSynchronize::is_at_safepoint(), "invariant") ;

  for (;;) {//自旋
      const markOop mark = object->mark() ;
      assert (!mark->has_bias_pattern(), "invariant") ;

      // The mark can be in one of the following states:
      // *  Inflated     - just return
      // *  Stack-locked - coerce it to inflated
      // *  INFLATING    - busy wait for conversion to complete
      // *  Neutral      - aggressively inflate the object.
      // *  BIASED       - Illegal.  We should never see this

      // CASE: inflated已膨胀，即重量级锁
      if (mark->has_monitor()) {//判断当前是否为重量级锁
          ObjectMonitor * inf = mark->monitor() ;//获取指向ObjectMonitor的指针
          assert (inf->header()->is_neutral(), "invariant");
          assert (inf->object() == object, "invariant") ;
          assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
          return inf ;
      }

      // CASE: inflation in progress - inflating over a stack-lock.膨胀等待（其他线程正在从轻量级锁转为膨胀锁）
      // Some other thread is converting from stack-locked to inflated.
      // Only that thread can complete inflation -- other threads must wait.
      // The INFLATING value is transient.
      // Currently, we spin/yield/park and poll the markword, waiting for inflation to finish.
      // We could always eliminate polling by parking the thread on some auxiliary list.
      if (mark == markOopDesc::INFLATING()) {
         TEVENT (Inflate: spin while INFLATING) ;
         ReadStableMark(object) ;
         continue ;
      }

      // CASE: stack-locked栈锁（轻量级锁） 
      // Could be stack-locked either by this thread or by some other thread.
      //
      // Note that we allocate the objectmonitor speculatively, _before_ attempting
      // to install INFLATING into the mark word.  We originally installed INFLATING,
      // allocated the objectmonitor, and then finally STed the address of the
      // objectmonitor into the mark.  This was correct, but artificially lengthened
      // the interval in which INFLATED appeared in the mark, thus increasing
      // the odds of inflation contention.
      //
      // We now use per-thread private objectmonitor free lists.
      // These list are reprovisioned from the global free list outside the
      // critical INFLATING...ST interval.  A thread can transfer
      // multiple objectmonitors en-mass from the global free list to its local free list.
      // This reduces coherency traffic and lock contention on the global free list.
      // Using such local free lists, it doesn't matter if the omAlloc() call appears
      // before or after the CAS(INFLATING) operation.
      // See the comments in omAlloc().

      if (mark->has_locker()) {
          ObjectMonitor * m = omAlloc (Self) ;//获取一个可用的ObjectMonitor 
          // Optimistically prepare the objectmonitor - anticipate successful CAS
          // We do this before the CAS in order to minimize the length of time
          // in which INFLATING appears in the mark.
          m->Recycle();
          m->_Responsible  = NULL ;
          m->OwnerIsThread = 0 ;
          m->_recursions   = 0 ;
          m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;   // Consider: maintain by type/class

          markOop cmp = (markOop) Atomic::cmpxchg_ptr (markOopDesc::INFLATING(), object->mark_addr(), mark) ;
          if (cmp != mark) {//CAS失败//CAS失败，说明冲突了，自旋等待//CAS失败，说明冲突了，自旋等待//CAS失败，说明冲突了，自旋等待
             omRelease (Self, m, true) ;//释放监视器锁
             continue ;       // Interference -- just retry
          }

          // We've successfully installed INFLATING (0) into the mark-word.
          // This is the only case where 0 will appear in a mark-work.
          // Only the singular thread that successfully swings the mark-word
          // to 0 can perform (or more precisely, complete) inflation.
          //
          // Why do we CAS a 0 into the mark-word instead of just CASing the
          // mark-word from the stack-locked value directly to the new inflated state?
          // Consider what happens when a thread unlocks a stack-locked object.
          // It attempts to use CAS to swing the displaced header value from the
          // on-stack basiclock back into the object header.  Recall also that the
          // header value (hashcode, etc) can reside in (a) the object header, or
          // (b) a displaced header associated with the stack-lock, or (c) a displaced
          // header in an objectMonitor.  The inflate() routine must copy the header
          // value from the basiclock on the owner's stack to the objectMonitor, all
          // the while preserving the hashCode stability invariants.  If the owner
          // decides to release the lock while the value is 0, the unlock will fail
          // and control will eventually pass from slow_exit() to inflate.  The owner
          // will then spin, waiting for the 0 value to disappear.   Put another way,
          // the 0 causes the owner to stall if the owner happens to try to
          // drop the lock (restoring the header from the basiclock to the object)
          // while inflation is in-progress.  This protocol avoids races that might
          // would otherwise permit hashCode values to change or "flicker" for an object.
          // Critically, while object->mark is 0 mark->displaced_mark_helper() is stable.
          // 0 serves as a "BUSY" inflate-in-progress indicator.


          // fetch the displaced mark from the owner's stack.
          // The owner can't die or unwind past the lock while our INFLATING
          // object is in the mark.  Furthermore the owner can't complete
          // an unlock on the object, either.
          markOop dmw = mark->displaced_mark_helper() ;
          assert (dmw->is_neutral(), "invariant") ;
          //CAS成功，设置ObjectMonitor的_header、_owner和_object等
          // Setup monitor fields to proper values -- prepare the monitor
          m->set_header(dmw) ;

          // Optimization: if the mark->locker stack address is associated
          // with this thread we could simply set m->_owner = Self and
          // m->OwnerIsThread = 1. Note that a thread can inflate an object
          // that it has stack-locked -- as might happen in wait() -- directly
          // with CAS.  That is, we can avoid the xchg-NULL .... ST idiom.
          m->set_owner(mark->locker());
          m->set_object(object);
          // TODO-FIXME: assert BasicLock->dhw != 0.

          // Must preserve store ordering. The monitor state must
          // be stable at the time of publishing the monitor address.
          guarantee (object->mark() == markOopDesc::INFLATING(), "invariant") ;
          object->release_set_mark(markOopDesc::encode(m));

          // Hopefully the performance counters are allocated on distinct cache lines
          // to avoid false sharing on MP systems ...
          if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
          TEVENT(Inflate: overwrite stacklock) ;
          if (TraceMonitorInflation) {
            if (object->is_instance()) {
              ResourceMark rm;
              tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
                (void *) object, (intptr_t) object->mark(),
                object->klass()->external_name());
            }
          }
          return m ;
      }

      // CASE: neutral 无锁
      // TODO-FIXME: for entry we currently inflate and then try to CAS _owner.
      // If we know we're inflating for entry it's better to inflate by swinging a
      // pre-locked objectMonitor pointer into the object header.   A successful
      // CAS inflates the object *and* confers ownership to the inflating thread.
      // In the current implementation we use a 2-step mechanism where we CAS()
      // to inflate and then CAS() again to try to swing _owner from NULL to Self.
      // An inflateTry() method that we could call from fast_enter() and slow_enter()
      // would be useful.

      assert (mark->is_neutral(), "invariant");
      ObjectMonitor * m = omAlloc (Self) ;
      // prepare m for installation - set monitor to initial state
      m->Recycle();
      m->set_header(mark);
      m->set_owner(NULL);
      m->set_object(object);
      m->OwnerIsThread = 1 ;
      m->_recursions   = 0 ;
      m->_Responsible  = NULL ;
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;       // consider: keep metastats by type/class

      if (Atomic::cmpxchg_ptr (markOopDesc::encode(m), object->mark_addr(), mark) != mark) {
          m->set_object (NULL) ;
          m->set_owner  (NULL) ;
          m->OwnerIsThread = 0 ;
          m->Recycle() ;
          omRelease (Self, m, true) ;
          m = NULL ;
          continue ;
          // interference - the markword changed - just retry.
          // The state-transitions are one-way, so there's no chance of
          // live-lock -- "Inflated" is an absorbing state.
      }

      // Hopefully the performance counters are allocated on distinct
      // cache lines to avoid false sharing on MP systems ...
      if (ObjectMonitor::_sync_Inflations != NULL) ObjectMonitor::_sync_Inflations->inc() ;
      TEVENT(Inflate: overwrite neutral) ;
      if (TraceMonitorInflation) {
        if (object->is_instance()) {
          ResourceMark rm;
          tty->print_cr("Inflating object " INTPTR_FORMAT " , mark " INTPTR_FORMAT " , type %s",
            (void *) object, (intptr_t) object->mark(),
            object->klass()->external_name());
        }
      }
      return m ;
  }
}
```
膨胀过程的实现比较复杂，大概实现过程如下：  

1、整个膨胀过程在自旋下完成；  
2、mark->has_monitor()方法判断当前是否为重量级锁（上图18-25行），即Mark Word的锁标识位为 10，如果当前状态为重量级锁，执行步骤（3），否则执行步骤（4）；  
3、mark->monitor()方法获取指向ObjectMonitor的指针，并返回，说明膨胀过程已经完成；  
4、如果当前锁处于膨胀中（上图33-37行），说明该锁正在被其它线程执行膨胀操作，则当前线程就进行自旋等待锁膨胀完成，这里需要注意一点，虽然是自旋操作，但不会一直占用cpu资源，每隔一段时间会通过os::NakedYield方法放弃cpu资源，或通过park方法挂起；如果其他线程完成锁的膨胀操作，则退出自旋并返回；  
5、如果当前是轻量级锁状态（上图58-138行），即锁标识位为 00，膨胀过程如下：  
* 通过omAlloc方法，获取一个可用的ObjectMonitor monitor，并重置monitor数据；
* 通过CAS尝试将Mark Word设置为markOopDesc:INFLATING，标识当前锁正在膨胀中，如果CAS失败，说明同一时刻其它线程已经将Mark Word设置为markOopDesc:INFLATING，当前线程进行自旋等待膨胀完成；
* 如果CAS成功，设置monitor的各个字段：_header、_owner和_object等，并返回；
6、如果是无锁（中立，上图150-186行），重置监视器值；  

**monitor竞争:**当锁膨胀完成并返回对应的monitor时，并不表示该线程竞争到了锁，而是会真正的锁竞争发生在ObjectMonitor::enter方法中。  
```c++
void ATTR ObjectMonitor::enter(TRAPS) {
  // The following code is ordered to check the most common cases first
  // and to reduce RTS->RTO cache line upgrades on SPARC and IA32 processors.
  Thread * const Self = THREAD ;
  void * cur ;

  cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
  if (cur == NULL) {//CAS成功
     // Either ASSERT _recursions == 0 or explicitly set _recursions = 0.
     assert (_recursions == 0   , "invariant") ;
     assert (_owner      == Self, "invariant") ;
     // CONSIDER: set or assert OwnerIsThread == 1
     return ;
  }

  if (cur == Self) {//重入锁
     // TODO-FIXME: check for integer overflow!  BUGID 6557169.
     _recursions ++ ;
     return ;
  }

  if (Self->is_lock_owned ((address)cur)) {
    assert (_recursions == 0, "internal state error");
    _recursions = 1 ;
    // Commute owner from a thread-specific on-stack BasicLockObject address to
    // a full-fledged "Thread *".
    _owner = Self ;
    OwnerIsThread = 1 ;
    return ;
  }

  // We've encountered genuine contention.
  assert (Self->_Stalled == 0, "invariant") ;
  Self->_Stalled = intptr_t(this) ;

  // Try one round of spinning *before* enqueueing Self
  // and before going through the awkward and expensive state
  // transitions.  The following spin is strictly optional ...
  // Note that if we acquire the monitor from an initial spin
  // we forgo posting JVMTI events and firing DTRACE probes.
  if (Knob_SpinEarly && TrySpin (Self) > 0) {
     assert (_owner == Self      , "invariant") ;
     assert (_recursions == 0    , "invariant") ;
     assert (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;
     Self->_Stalled = 0 ;
     return ;
  }

  assert (_owner != Self          , "invariant") ;
  assert (_succ  != Self          , "invariant") ;
  assert (Self->is_Java_thread()  , "invariant") ;
  JavaThread * jt = (JavaThread *) Self ;
  assert (!SafepointSynchronize::is_at_safepoint(), "invariant") ;
  assert (jt->thread_state() != _thread_blocked   , "invariant") ;
  assert (this->object() != NULL  , "invariant") ;
  assert (_count >= 0, "invariant") ;

  // Prevent deflation at STW-time.  See deflate_idle_monitors() and is_busy().
  // Ensure the object-monitor relationship remains stable while there's contention.
  Atomic::inc_ptr(&_count);

  EventJavaMonitorEnter event;

  { // Change java thread status to indicate blocked on monitor enter.
    JavaThreadBlockedOnMonitorEnterState jtbmes(jt, this);

    DTRACE_MONITOR_PROBE(contended__enter, this, object(), jt);
    if (JvmtiExport::should_post_monitor_contended_enter()) {
      JvmtiExport::post_monitor_contended_enter(jt, this);
    }

    OSThreadContendState osts(Self->osthread());
    ThreadBlockInVM tbivm(jt);

    Self->set_current_pending_monitor(this);

    // TODO-FIXME: change the following for(;;) loop to straight-line code.
    for (;;) {
      jt->set_suspend_equivalent();
      // cleared by handle_special_suspend_equivalent_condition()
      // or java_suspend_self()

      EnterI (THREAD) ;

...省略...139 }
```
1、通过CAS尝试把monitor的_owner字段设置为当前线程；  
2、如果设置之前的_owner指向当前线程，说明当前线程再次进入monitor，即重入锁，执行_recursions ++ ，记录重入的次数；  
3、如果之前的_owner指向的地址在当前线程中，这种描述有点拗口，换一种说法：之前_owner指向的BasicLock在当前线程栈上，说明当前线程是第一次进入该monitor，设置_recursions为1，_owner为当前线程，该线程成功获得锁并返回；  
4、如果获取锁失败，则等待锁的释放；  

**monitor等待**  
monitor竞争失败的线程，通过自旋执行ObjectMonitor::EnterI方法等待锁的释放，EnterI方法的部分逻辑实现如下：  
```c++
ObjectWaiter node(Self) ;
Self->_ParkEvent->reset() ;
node._prev   = (ObjectWaiter *) 0xBAD ;
node.TState  = ObjectWaiter::TS_CXQ ;

// Push "Self" onto the front of the _cxq.
// Once on cxq/EntryList, Self stays on-queue until it acquires the lock.
// Note that spinning tends to reduce the rate at which threads
// enqueue and dequeue on EntryList|cxq.
ObjectWaiter * nxt ;
for (;;) {
    node._next = nxt = _cxq ;
    if (Atomic::cmpxchg_ptr (&node, &_cxq, nxt) == nxt) break ;

    // Interference - the CAS failed because _cxq changed.  Just retry.
    // As an optional optimization we retry the lock.
    if (TryLock (Self) > 0) {
        assert (_succ != Self         , "invariant") ;
        assert (_owner == Self        , "invariant") ;
        assert (_Responsible != Self  , "invariant") ;
        return ;
    }
}
```
1、当前线程被封装成ObjectWaiter对象node，状态设置成ObjectWaiter::TS_CXQ；  
2、在for循环中，通过CAS把node节点push到_cxq列表中，同一时刻可能有多个线程把自己的node节点push到_cxq列表中；  
3、node节点push到_cxq列表之后，通过自旋尝试获取锁，如果还是没有获取到锁，则通过park将当前线程挂起，等待被唤醒，实现如下：  
```c++
for (;;) {

    if (TryLock (Self) > 0) break ;
    assert (_owner != Self, "invariant") ;

    if ((SyncFlags & 2) && _Responsible == NULL) {
       Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
    }

    // park self
    if (_Responsible == Self || (SyncFlags & 1)) {
        TEVENT (Inflated enter - park TIMED) ;
        Self->_ParkEvent->park ((jlong) RecheckInterval) ;
        // Increase the RecheckInterval, but clamp the value.
        RecheckInterval *= 8 ;
        if (RecheckInterval > 1000) RecheckInterval = 1000 ;
    } else {
        TEVENT (Inflated enter - park UNTIMED) ;
        Self->_ParkEvent->park() ;//当前线程挂起
    }

    if (TryLock(Self) > 0) break ;

    // The lock is still contested.
    // Keep a tally of the # of futile wakeups.
    // Note that the counter is not protected by a lock or updated by atomics.
    // That is by design - we trade "lossy" counters which are exposed to
    // races during updates for a lower probe effect.
    TEVENT (Inflated enter - Futile wakeup) ;
    if (ObjectMonitor::_sync_FutileWakeups != NULL) {
       ObjectMonitor::_sync_FutileWakeups->inc() ;
    }
    ++ nWakeups ;

    // Assuming this is not a spurious wakeup we'll normally find _succ == Self.
    // We can defer clearing _succ until after the spin completes
    // TrySpin() must tolerate being called with _succ == Self.
    // Try yet another round of adaptive spinning.
    if ((Knob_SpinAfterFutile & 1) && TrySpin (Self) > 0) break ;

    // We can find that we were unpark()ed and redesignated _succ while
    // we were spinning.  That's harmless.  If we iterate and call park(),
    // park() will consume the event and return immediately and we'll
    // just spin again.  This pattern can repeat, leaving _succ to simply
    // spin on a CPU.  Enable Knob_ResetEvent to clear pending unparks().
    // Alternately, we can sample fired() here, and if set, forgo spinning
    // in the next iteration.

    if ((Knob_ResetEvent & 1) && Self->_ParkEvent->fired()) {
       Self->_ParkEvent->reset() ;
       OrderAccess::fence() ;
    }
    if (_succ == Self) _succ = NULL ;

    // Invariant: after clearing _succ a thread *must* retry _owner before parking.
    OrderAccess::fence() ;
}
```
4、当该线程被唤醒时，会从挂起的点继续执行，通过ObjectMonitor::TryLock尝试获取锁，TryLock方法实现如下：  
```c++
int ObjectMonitor::TryLock (Thread * Self) {
   for (;;) {
      void * own = _owner ;
      if (own != NULL) return 0 ;
      if (Atomic::cmpxchg_ptr (Self, &_owner, NULL) == NULL) {//CAS成功，获取锁
         // Either guarantee _recursions == 0 or set _recursions = 0.
         assert (_recursions == 0, "invariant") ;
         assert (_owner == Self, "invariant") ;
         // CONSIDER: set or assert that OwnerIsThread == 1
         return 1 ;
      }
      // The lock had been free momentarily, but we lost the race to the lock.
      // Interference -- the CAS failed.
      // We can either return -1 or retry.
      // Retry doesn't make as much sense because the lock was just acquired.
      if (true) return -1 ;
   }
}
```
其本质就是通过CAS设置monitor的_owner字段为当前线程，如果CAS成功，则表示该线程获取了锁，跳出自旋操作，执行同步代码，否则继续被挂起；  

**monitor释放**
当某个持有锁的线程执行完同步代码块时，会进行锁的释放，给其它线程机会执行同步代码，在HotSpot中，通过退出monitor的方式实现锁的释放，并通知被阻塞的线程，具体实现位于ObjectMonitor::exit方法中。
```c++
void ATTR ObjectMonitor::exit(bool not_suspended, TRAPS) {
   Thread * Self = THREAD ;
   if (THREAD != _owner) {
     if (THREAD->is_lock_owned((address) _owner)) {
       // Transmute _owner from a BasicLock pointer to a Thread address.
       // We don't need to hold _mutex for this transition.
       // Non-null to Non-null is safe as long as all readers can
       // tolerate either flavor.
       assert (_recursions == 0, "invariant") ;
       _owner = THREAD ;
       _recursions = 0 ;
       OwnerIsThread = 1 ;
     } else {
       // NOTE: we need to handle unbalanced monitor enter/exit
       // in native code by throwing an exception.
       // TODO: Throw an IllegalMonitorStateException ?
       TEVENT (Exit - Throw IMSX) ;
       assert(false, "Non-balanced monitor enter/exit!");
       if (false) {
          THROW(vmSymbols::java_lang_IllegalMonitorStateException());
       }
       return;
     }
   }

   if (_recursions != 0) {
     _recursions--;        // this is simple recursive enter
     TEVENT (Inflated exit - recursive) ;
     return ;
   }
...省略...
```    
1、如果是重量级锁的释放，monitor中的_owner指向当前线程，即THREAD == _owner；  
2、根据不同的策略（由QMode指定），从cxq或EntryList中获取头节点，通过ObjectMonitor::ExitEpilog方法唤醒该节点封装的线程，唤醒操作最终由unpark完成，实现如下：  
```c++
void ObjectMonitor::ExitEpilog (Thread * Self, ObjectWaiter * Wakee) {
   assert (_owner == Self, "invariant") ;

   // Exit protocol:
   // 1. ST _succ = wakee
   // 2. membar #loadstore|#storestore;
   // 2. ST _owner = NULL
   // 3. unpark(wakee)

   _succ = Knob_SuccEnabled ? Wakee->_thread : NULL ;
   ParkEvent * Trigger = Wakee->_event ;

   // Hygiene -- once we've set _owner = NULL we can't safely dereference Wakee again.
   // The thread associated with Wakee may have grabbed the lock and "Wakee" may be
   // out-of-scope (non-extant).
   Wakee  = NULL ;

   // Drop the lock
   OrderAccess::release_store_ptr (&_owner, NULL) ;
   OrderAccess::fence() ;                               // ST _owner vs LD in unpark()

   if (SafepointSynchronize::do_call_back()) {
      TEVENT (unpark before SAFEPOINT) ;
   }

   DTRACE_MONITOR_PROBE(contended__exit, this, object(), Self);
   Trigger->unpark() ;

   // Maintain stats and report events to JVMTI
   if (ObjectMonitor::_sync_Parks != NULL) {
      ObjectMonitor::_sync_Parks->inc() ;
   }
}
```
3、被唤醒的线程，继续执行monitor的竞争；需要注意的是**synchronized是非公平锁**，并不是先等待的线程先获得锁。  
* 公平锁（Fair）：加锁前检查是否有排队等待的线程，优先排队等待的线程，先来先得
* 非公平锁（Nonfair）：加锁时不考虑排队等待问题，直接尝试获取锁，获取不到自动到队尾等待；换句话说，非公平锁来了就先争夺一下，失败了再去排队。排队之后依然满足FIFO规则。  

ReentrantLock与synchronized关键字一样，属于互斥锁，synchronized中的锁是非公平的（公平锁是指多个线程等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁），ReentrantLock默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。线程通过ReentrantLock的lock()方法获得锁，用unlock()方法释放锁。  

_ _ _
### **总结**  
本文重点介绍了Synchronized原理以及JVM对Synchronized的优化。简单来说解决三种场景：  

1）只有一个线程进入临界区，偏向锁

2）多个线程交替进入临界区，轻量级锁

3）多线程同时进入临界区，重量级锁
