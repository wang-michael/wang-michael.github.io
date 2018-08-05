# java.util.concurrent包学习笔记
## 一、概述 
java.util.concurrent中更高级的工具分成3类：Executor Framework、并发集合(Concurrent Collection)以及同步器(Synchronizer)。
1、学习java中同步器(Synchronizer) Semaphore Exchanger，lock接口及相关实现
2、学习java中并发集合结合传统集合类来学习
3、学习java中Executor Framework，Future-callable

## 二、同步器(Synchronizer)
###CountDownLatch
---
API文档翻译
```java

CountDownLatch @since 1.5 BY Doug Lea  

同步的目的是允许一个或多个线程等待，直到在其他线程中执行的一组操作完成。

CountDownLatch用给定的计数进行初始化。 调用await方法的线程会阻塞直到当前的CountDownLatch由于调用countDown()方法而达到零，此后所有等待的线程被释放，并且任何后续的await调用立即返回。CountDownLatch中的计数无法重置。 如果您需要重置计数的版本，请考虑使用CyclicBarrier。

CountDownLatch是一个多功能的同步工具，可用于多种目的。CountDownLatch初始化的计数为1用作一个简单的开/关锁存器或门：所有线程都调用在门处等待等待，直到它被调用countDown（）的线程打开为止。 初始化为N的CountDownLatch可用于使一个线程等待，直到N个线程完成某个动作，或者某个动作已完成N次。

调用countdown()方法的线程不会阻塞，调用await()方法的线程在计数器清零之前会阻塞等待。  

Driver-Worker模型：
  class Driver { // ...
   void main() throws InterruptedException {
     CountDownLatch startSignal = new CountDownLatch(1);
     CountDownLatch doneSignal = new CountDownLatch(N);

     for (int i = 0; i < N; ++i) // create and start threads
       new Thread(new Worker(startSignal, doneSignal)).start();

     doSomethingElse();            // don't let run yet
     startSignal.countDown();      // let all threads proceed
     doSomethingElse();
     doneSignal.await();           // wait for all to finish
   }
 }

 class Worker implements Runnable {
   private final CountDownLatch startSignal;
   private final CountDownLatch doneSignal;
   Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
     this.startSignal = startSignal;
     this.doneSignal = doneSignal;
   }
   public void run() {
     try {
       startSignal.await();
       doWork();
       doneSignal.countDown();
     } catch (InterruptedException ex) {} // return;
   }

   void doWork() { ... }
 }

另一个典型用法是将问题分为N个部分，用Runnable来描述每个部分，该部分执行该部分的任务并在锁存器上进行倒计数，并将所有Runnables排队到Executor。 当所有子部分完成时，协调线程将能够结束等待。 （当线程必须以这种方式重复倒数时，请使用CyclicBarrier。）
  class Driver2 { // ...
   void main() throws InterruptedException {
     CountDownLatch doneSignal = new CountDownLatch(N);
     Executor e = ...

     for (int i = 0; i < N; ++i) // create and start threads
       e.execute(new WorkerRunnable(doneSignal, i));

     doneSignal.await();           // wait for all to finish
   }
 }

 class WorkerRunnable implements Runnable {
   private final CountDownLatch doneSignal;
   private final int i;
   WorkerRunnable(CountDownLatch doneSignal, int i) {
     this.doneSignal = doneSignal;
     this.i = i;
   }
   public void run() {
     try {
       doWork(i);
       doneSignal.countDown();
     } catch (InterruptedException ex) {} // return;
   }

   void doWork() { ... }
 }

重点！！！
一个线程调用countDown方法 happen-before 另外一个线程调用await方法。

await()：使调用线程阻塞直到指定CountDownLatch倒计时为0,除非当前线程被中断，如果调用此方法时计数器值已经设置为0，则调用await不会再阻塞。如果经过指定的等待时间，则返回false值。 如果时间小于或等于零，该方法将不会等待。     
boolean	await(long timeout, TimeUnit unit)：导致当前线程等待直到锁存器计数到零，除非线程中断或经过了指定的等待时间。  
countDown()：减少锁存器的计数，直到计数达到零释放所有等待的线程。  
long getCount()：返回CountDownLatch中当前计数。  
String toString()：返回字符串标识当前latch及其状态。    

注意：调用await阻塞的线程可以响应中断(await底层还是调用了wait方法吗？？？)  

```

###CyclicBarrier
---
API文档翻译
```java
CyclicBarrier @since 1.5 BY Doug Lea  

此工具设置的目的是允许一组线程互相等待以达到共同的障碍点。 CyclicBarriers在涉及固定大小的线程的程序中很有用，它必须偶尔等待对方。 该屏障称为循环屏障，因为它可以在等待线程释放之后重新使用。

以下是在并行计算设计中使用CyclicBarriers的示例：

  class Solver {
   final int N;
   final float[][] data;
   final CyclicBarrier barrier;

   class Worker implements Runnable {
     int myRow;
     Worker(int row) { myRow = row; }
     public void run() {
       while (!done()) {
         processRow(myRow);

         try {
           barrier.await();
         } catch (InterruptedException ex) {
           return;
         } catch (BrokenBarrierException ex) {
           return;
         }
       }
     }
   }

   public Solver(float[][] matrix) {
     data = matrix;
     N = matrix.length;
     Runnable barrierAction =
       new Runnable() { 
          public void run() {
             // 合并各个行中计算好的结果，并使得之后对于done()方法的调用返回true。   
             mergeRows(...); 
          }
     };
     // CyclicBarrier默认的构造方法是CyclicBarrier(int parties, Runnable barrierAction)，其参数表示屏障拦截的线程数量，每个线程调用await
     // 方法告诉CyclicBarrier我已
     // 经到达了屏障，然后当前线程被阻塞。直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。构造函数中的Runnable指定了在
     // 最后一个线程到达屏障时，先执行Runnable中指定的任务，此任务执行之后阻塞在当前CyclicBarrier上的所有线程才会返回，这样方便处理复杂的业务
     // 场景。
     barrier = new CyclicBarrier(N, barrierAction);

     List<Thread> threads = new ArrayList<Thread>(N);
     for (int i = 0; i < N; i++) {
       Thread thread = new Thread(new Worker(i));
       threads.add(thread);
       thread.start();
     }

     // wait until done
     for (Thread thread : threads)
       thread.join();
   }
 }

在这里，每个工作线程处理矩阵的一行，然后在障碍处等待，直到处理完所有行。 当处理完所有行后，执行提供的Runnable barrier操作并合并行。 如果合并确定已找到解决方案，则done（）将返回true，并且每个工作线程将终止。如果中间出了差错，可以使用reset方法重置CyclicBarrier并使得所有工作线程重新运行一次。  

如果屏障行为不依赖于执行时被暂停的方，那么当被暂停的方被释放时，被暂停的方中的任何线程都可以执行该行为。 为了实现这一点，await()的每次调用都会返回该线程到达屏障的索引。 然后，您可以选择执行屏障操作的线程，例如：

 if (barrier.await() == 0) {
   // log the completion of this iteration
 }

CyclicBarrier对于失败的同步尝试使用全部或一个都没有(all-or-none)的破坏模型：如果线程由于中断，失败或超时而过早离开障碍点，则在该障碍点等待的所有其他线程也将通过BrokenBarrierException异常退出（或InterruptedException 如果他们也几乎同时被打断）。

```
CyclicBarrier建立的happens-before关系：
<img src="/img/2018-4-19/CyclicBarrierHappensBefore.jpg" width="300" height="300" alt="CyclicBarrier Happens-Before图" />

**CyclicBarrier与CountDownLatch的异同：**
* CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景，例如如果发生计算错误，可以重置计数器，并让线程重新执行一次。
* CyclicBarrier相比于CountDownLatch提供了更多有用的方法，比如getNumberWaiting方法可以获得被CyclicBarrier阻塞的线程数量，isBroken方法用来了解阻塞的线程是否被中断。
* CyclicBarrier采用了all-or-none break模型，只要一个线程从await阻塞中不正常返回(比如被中断或者未到指定时间就返回，到指定时间返回会影响其他线程吗？？？)，其他所有被阻塞的线程都会立即返回并抛出BrokenBarrierException(在这之后调用此barrier.await()方法也会立即返回并抛出异常吗？？？)；CountDownLatch与其则不相同，非正常返回的线程并不会影响其他被阻塞的线程。  

**我认为仅从应用层面来讲CountDownLatch、CyclicBarrier、Wait-Notify有以下场景：**
* CountDownLatch：**等待线程的数量是不固定的，可指定通知线程的数量**(比如设置countDownLatch(10)可以使得未知数量的等待线程在10个通知线程调用countDown()方法之后继续运行)，countDown方法与await方法配合使用功能比较强大，但缺点在于只能使用一次；
* CyclicBarrier：**等待线程的数量是固定的，每个线程既是等待线程也是通知线程**，当最后一个等待线程到来时，所有的等待线程会停止阻塞，继续执行；功能没有CountDownLatch强大，但可以循环使用；
* Wait-Notify：**等待线程的数量是不固定的，通知线程的数量可以是一个(调用notifyAll方法)或多个(notify与notifyAll方法结合使用)**，countDownLatch中调用countDown()方法的线程可以类比于wait-notify中调用notify()方法的线程，调用await方法的线程可以类比于调用wait方法的线程；显而易见，countDownLatch在notify等待线程方式上相比于wait-notify方式更强，可以在指定数量的notify线程执行countDown()方法之后才放行等待线程。 但是wait-notify相比于countDownLatch的优势在于可以循环使用。  

### LOCK
---
API文档翻译:
```java
LOCK实现提供比使用Synchronized方法和语句可获得的锁定操作更广泛的锁定操作。 它们允许更灵活的结构化，可能具有完全不同的属性，并且可以支持多个关联的Condition对象。

Lock是一个控制多线程共享资源访问的工具，通常情况下，一个lock提供了一个对于共享资源的唯一的访问方式：一次只能有一个线程获取到锁并且所有对于共享资源的访问都需要首先获取到锁。当然也有例外，有些锁允许对于共享资源的并发访问，比如ReadWriteLock中的读锁。  

synchronized方法和段落提供了与其关联对象的隐式的锁获取和释放，并且多个锁被获取的时候，这些锁必须以相反的顺序被释放。  

synchronized提供的锁机制是易于编程实现的，帮助我们避免了许多涉及到锁的编程错误，当然还有一些场合你需要更自由的使用锁。比如Lock提供的锁机制允许你获取和释放锁使用任意的顺序，而不仅仅是与获取锁的顺序相反。  

增长的自由度伴随着额外的责任，通常情况下，需要这样使用Lock：  

 Lock l = ...;
 l.lock();
 try {
   // access the resource protected by this lock
 } finally {
   l.unlock();
 }

我们需要确保在出现异常情况时Lock也可以被正确的释放。  

Lock相比于synchronized提供了一些额外的功能，比如非阻塞式的获取锁:tryLock();可中断式的获取锁(即在锁的获取过程中可以中断当前线程):lockInterruptibly();可超时的获取锁：tryLock(long, TimeUnit)。  

Lock类还可以提供与隐式监视器锁定(比如synchronized)非常不同的行为和语义，比如保证顺序？、不可重入锁的使用以及死锁检测。

记住Lock实例只是一个普通的对象，甚至它自己也可以作为synchronized中的monitor。使用synchronized获取Lock对象的monitor锁与调用lock()方法获取lock对象的锁之间没有任何关系。为了避免歧义，永远不要在synchronized的目标对象中选择lock实例，这会引起歧义。  

成功的lock和unlock操作都需要内存同步语义，不成功的lock和unlock操作和重入的lock和unlock操作，不需要任何内存同步的语义。  
```
问题1：锁住的共享资源指的是什么资源？？？怎么样才叫被锁住了？？？一个Lock可以关联多个共享资源吗？？？    

### AbstractQueuedSynchronizer
---
AbstractQueuedSynchronizer是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。只有掌握了同步器的工作原理才能更加深入的理解并发包中的其它并发组件。并发包的作者希望AbstractQueuedSynchronizer成为实现大部分同步需求的基础。

API文档翻译:
```java
根据先进先出的等待队列实现的框架，这个框架可以用来实现阻塞式的锁和其它相关的synchronizers(semaphores、events等等)。这个类的设计目标是作为那些依赖于单一atomic int值来代表状态的synchronizers的实现基础。子类必须在protected方法中定义状态的改变，并且定义这些状态的含义：比如代表这个对象被获取或者被释放。鉴于这个目标，这个类中的其它方法致力于实现所有的排队和阻塞机制。子类可以维护其它的状态值，但是只有使用getState(),setState()和compareAndSetState(int, int)操作的原子int值是与同步相关的。  

用来实现自定义类的同步特性的AbstractQueuedSynchronizer类的子类应该被定义为外部类的非public的内部类。AbstractQueuedSynchronizer类不实现任何同步接口，其内部定义了一些诸如acquireInterruptibly(int)一类的方法用来被具体的锁或同步器实现他们的public方法时调用。    

这个类同时支持独占式和共享式获取。当使用独占式获取时，其它线程的获取尝试不能成功。多线程的共享式获取可能会成功。等待在不同模式中的线程共享同一个FIFO队列，通常情况下子类的实现只支持这些模式中的一个，但是有的情况两种模式也可能都支持:比如ReadWriteLock。对于只支持独占式或者只支持共享式的子类不需要实现方法来支持不实现的模式。    

作为同步器的基础使用这些方法时，需要重定义以下方法，并在这些方法中使用getState(), setState(int) and/or compareAndSetState(int, int)方法来检查、更改同步的状态。  

使用者可以重定义的方法有tryAcquire(int)、tryRelease(int)、tryAcquireShared(int)、tryReleaseShared(int)、isHeldExclusively()，重定义这些方法的实现时，方法内部通常应该是线程安全的、代码短小并且不会阻塞的。继承AbstractQueuedSynchronizer类并重定义这些方法是实现自定义同步类的唯一手段，AbstractQueuedSynchronizer类中所有的其它方法都是final的，不能独立变化。

您也可以从AbstractOwnableSynchronizer中找到继承的方法，以便跟踪拥有独占同步器的线程。 我们鼓励您使用它们 - 这是监视和诊断工具，能够帮助用户确定哪些线程持有锁。  

这个类为部分同步类的实现提供了一个高效的、易于扩展的同步基础的实现。当这个类的功能不够使用时，可以使用更底层的atomic类，自定义的Queue类和LockSupport类来实现同步器。  
```
自定义同步组件---->调用同步器提供的模板方法---->调用使用者重写的方法---->在重写的方法中调用getState、setState、compareAndSetState方法修改同步状态。

使用者可以重写的方法包含AbstractQueuedSynchronizer中的：
* tryAcquire(int)
* tryRelease(int)
* tryAcquireShared(int)
* tryReleaseShared(int)
* isHeldExclusively()

可以重写AbstractOwnableSynchronizer中哪些方法呢？重写这些方法有何作用呢？？？  

使用示例：
这是一个不可重入的互斥锁类，它使用零值表示解锁状态，一表示锁定状态。尽管非重入锁并不严格要求记录当前所有者线程，但该类还是这样做了，以便更易于监视使用。这个类还支持condition并暴露了一个instrumentation methods。
```java
class Mutex implements Lock, java.io.Serializable {

   // Our internal helper class
   private static class Sync extends AbstractQueuedSynchronizer {
     // Reports whether in locked state
     protected boolean isHeldExclusively() {
       return getState() == 1;
     }

     // Acquires the lock if state is zero
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }

     // Releases the lock by setting state to zero
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }

     // Provides a Condition
     Condition newCondition() { return new ConditionObject(); }

     // Deserializes properly
     private void readObject(ObjectInputStream s)
         throws IOException, ClassNotFoundException {
       s.defaultReadObject();
       setState(0); // reset to unlocked state
     }
   }

   // The sync object does all the hard work. We just forward to it.
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
}
```

AbstractQueuedSynchronizer方法及其简单使用：
1. 需要被使用者子类重定义的5个方法：  
```java
1、protected boolean tryAcquire(int arg)：尝试以独占锁的方式获取锁，这个方法应该询问当前锁对象是否允许以独占模式被获取，如果允许的话再获取。这个方法通常被尝试获取锁的线程调用，如果这个方法报告占锁失败，则可以使用acquire方法将此线程入队等待，直到被其它释放锁的线程唤醒。这个方法可以用来实现Lock.tryLock()。  

arg参数通常情况下可以是传递到acquire方法中的参数，或者是进入条件等待时保存的值，甚至可以是任意自己喜欢的值。  

Lock.tryLock()：尝试非阻塞的获取锁，调用该方法后立刻返回，如果能够获取则返回true，否则返回false。  

2、protected boolean tryRelease(int arg)：尝试设置一个在独占锁情况下代表锁释放的状态的值，通常被执行释放操作的线程调用。

arg参数通常是传递到release()方法中的参数或进入条件等待时的当前状态值，甚至是任意喜欢的值。

3、protected int tryAcquireShared(int arg)：与tryAcquire方法不同之处仅在于尝试以共享模式获取锁。  

返回值：负数代表失败，0代表当前以共享模式获取锁成功，但是后续的以共享模式获取锁的操作一定不能成功；正数代表当前以共享模式获取锁的线程获取成功并且之后的线程的共享模式获取操作也有可能成功。

4、protected boolean tryReleaseShared(int arg)：尝试将状态设置为共享锁释放的状态。其余与tryRelease方法相同。  

5、protected boolean isHeldExclusively()：如果同步操作被当前线程同步占有的话返回true。这个方法仅仅被AbstractQueuedSynchronizer.ConditionObject类中的non-waiting方法调用(waiting 方法调用的是release(int))。由于这个方法仅仅在AbstractQueuedSynchronizer.ConditionObject内部类时使用到，所以如果我们实现自定义同步组件时没有用到AbstractQueuedSynchronizer.ConditionObject对象的话，就没有必要重写这个方法了。  

```
2.

###ReadWriteLock
---
API文档翻译
```java
读写锁允许访问共享数据的并发性水平高于互斥锁允许的水平。 它利用了这样一个事实，即一次只有一个线程（一个编写器线程）可以修改共享数据，在很多情况下，任何数量的线程都可以同时读取数据（因此读取器线程）。 理论上，使用读写锁定所允许的并发性增加将导致性能提高超过使用互斥锁。 实际上，这种并发性的增加只能在多处理器上完全实现，并且只有在共享数据的访问模式合适时才会如此。

读写锁是否会提高使用互斥锁的性能取决于数据读取的频率与被修改的频率，读取和写入操作的持续时间以及数据的争用——也就是，尝试同时读取或写入数据的线程数。例如，一个最初用数据填充并且此后不经常修改，而频繁搜索（例如某种目录）的集合是使用读写锁定的理想候选。然而，如果更新变得频繁，那么数据的大部分时间都被锁定，并且几乎没有并发的增加。此外，如果读取操作太短，则读写锁定实现（本质上比互斥锁定更复杂）的开销占据了执行成本的大部分，特别是因为许多读写锁定实现仍然通过小部分代码互斥执行。最终，只有分析和测量才能确定使用读写锁是否适合您的应用程序。  

实现时需要考虑的几个问题：
1. 在写锁释放锁定时，如果读锁和写锁都在等待，是把锁给读锁还是写锁呢？偏向写锁的情况很常见，因为写操作通常很短且很少发生；偏向读操作则并不常见，因为读操作如果频繁且耗时，会导致写操作的长时间延迟。当然公平的实现也是有可能的。
2. 当读锁处于进行状态时，有写锁和读锁请求正在等待，先给谁呢？偏向读操作可能会导致写操作的长时间延迟，偏向写操作会降低并发量。  
3. 确定这些锁是否是可重入的：一个带有写入锁的线程是否可以重新获取此写锁？ 可以在保持写锁的同时获取读锁吗？ 读锁本身是否可重入？--->都可以的。  
4. 写锁降级为读锁的时候是否不允许其他写锁参与？---->是的。 一个读锁能否升级为写锁而不是偏向于其他的读锁或写锁？---->不能
```
ReentrantReadWriteLock的API文档翻译:
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

## 三、并发集合
###ConcurrentHashMap
API文档翻译
```java
ConcurrentHashMap是一个哈希表，支持检索的完全并发和更新的高预期并发性。 该类遵循与Hashtable相同的功能规范，并包括与Hashtable的每个方法相对应的方法版本。 但是，即使所有操作都是线程安全的，检索操作也并不意味着锁定，并且不支持以阻止所有访问的方式锁定整个表。 在依赖于线程安全但不依赖于其同步细节的程序中，此类可与Hashtable完全互操作。

检索操作（包括get）通常不会阻塞，因此可能与更新操作（包括put和remove）同时进行。检索反映了最近完成的更新操作的结果。 （更正式地说，更新操作与在其之后进行的检索操作之间存在Happens-Before关系)。  

对于诸如putAll和clear之类的聚合操作，并发检索可能反映仅插入或删除某些条目。 类似地，Iterators，Spliterators和Enumerations在迭代器/枚举的创建时或之后的某个时刻返回反映哈希表状态的元素。 它们不会抛出ConcurrentModificationException。 但是，迭代器设计为一次只能由一个线程使用。 请记住，聚合状态方法（包括size，isEmpty和containsValue）的结果通常仅在映射未在其他线程中进行并发更新时才有用。 否则，这些方法的结果反映了可能足以用于监视或估计目的的瞬态，但不适用于程序控制。 

当存在太多冲突时（即，具有不同哈希码的密钥但落入以表大小为模的相同槽中的密钥），该表被动态扩展，具有每个映射大致保持两个箱的预期平均效果（对应于0.75负载）调整大小的因子阈值）。随着映射的添加和删除，这个平均值可能会有很大的差异，但总的来说，这维持了哈希表的普遍接受的时间/空间权衡。但是，调整此大小或任何其他类型的散列表可能是一个相对较慢的操作。如果可能，最好将大小估计值作为可选的initialCapacity构造函数参数提供。另一个可选的loadFactor构造函数参数通过指定在计算给定数量的元素时要分配的空间量时使用的表密度，提供了另一种自定义初始表容量的方法。此外，为了与此类的先前版本兼容，构造函数可以选择将预期的concurrencyLevel指定为内部大小调整的附加提示。请注意，使用具有完全相同hashCode（）的许多键是减慢任何哈希表性能的可靠方法。为了改善影响，当键是可比较时，该类可以使用键之间的比较顺序来帮助打破关系。

可以创建ConcurrentHashMap的Set投影（使用newKeySet（）或newKeySet（int）），或者查看（仅在感兴趣的键时使用keySet（Object），并且映射的值（可能是暂时的）不使用或者全部采用 相同的映射值。

通过使用LongAdder值并通过computeIfAbsent初始化，ConcurrentHashMap可用作可伸缩频率映射（直方图或多集的形式）。 例如，要向ConcurrentHashMap <String，LongAdder> freqs添加计数，可以使用freqs.computeIfAbsent（k - > new LongAdder（））。increment（）;

与Hashtable类似，但与HashMap不同，此类不允许将null用作键或值。

ConcurrentHashMaps支持一组顺序和并行批量操作，与大多数Stream方法不同，它们被设计为安全且通常合理地应用，即使是由其他线程同时更新的映射; 例如，在共享注册表中计算值的快照摘要时。 有三种操作，每种操作有四种形式，接受具有键，值，条目和（键，值）参数和/或返回值的函数。 因为ConcurrentHashMap的元素没有以任何特定的方式排序，并且可以在不同的并行执行中以不同的顺序处理，所提供的函数的正确性不应该依赖于任何排序，或者可能依赖于任何其他可能瞬时变化的对象或值。 计算正在进行中; 除了forEach动作外，理想情况下应该是无副作用的。 Map.Entry对象上的批量操作不支持方法setValue。
```

常见问题：  
* ConcurrentHashMap如何实现线程安全？ 为何比HashTable高效?(锁隔离)
* 同时可以从ConcurrentHashMap读取多个线程吗？
* 同时可以在ConcurrentHashMap上一个线程读取和其他线程写入吗？
* ConcurrentHashMap如何在内部工作？
* 如何在ConcurrentHashMap中原子更新值？
* 在迭代ConcurrentHashMap时如何删除映射？
* ConcurrentHashMap的迭代器是否故障安全或故障快速？
* 如果在ConcurrentHashMap中添加一个新映射，而一个线程正在迭代，会发生什么？


###ConcurrentLinkedQueue
---
API文档翻译:
```java
基于链接节点的无界线程安全队列。 此队列命令元素FIFO（先进先出）。 队列的头部是队列中最长时间的元素。 队列的尾部是队列中最短时间的元素。 在队列的尾部插入新元素，队列检索操作获取队列头部的元素。 当许多线程共享对公共集合的访问权限时，ConcurrentLinkedQueue是一个合适的选择。 与大多数其他并发集合实现一样，此类不允许使用null元素。

该实现采用有效的非阻塞算法，该算法基于Maged M.Michael和Michael L.Scott的简单，快速，实用的非阻塞和阻塞并发队列算法中描述的算法。

迭代器是弱一致的，在迭代器创建时或之后的某个时刻返回反映队列状态的元素。它们不会抛出ConcurrentModificationException，并且可能与其他操作同时进行。自创建迭代器以来队列中包含的元素将只返回一次。

请注意，与大多数集合不同，size方法不是恒定时间操作。由于这些队列的异步性质，确定元素的当前数量需要遍历元素，因此如果在遍历期间修改此集合，则可能会报告不准确的结果。此外，批量操作addAll，removeAll，retainAll，containsAll，equals和toArray不保证以原子方式执行。例如，与addAll操作同时运行的迭代器可能只查看一些添加的元素。

该类及其迭代器实现了Queue和Iterator接口的所有可选方法。

内存一致性影响：与其他并发集合一样，在将对象放入ConcurrentLinkedQueue之前，线程中的操作发生在从另一个线程中的ConcurrentLinkedQueue访问或删除该元素之后的操作之前。

此类是Java Collections Framework的成员。  
```
Michael-Scott 非阻塞队列算法中的插入:  
```java
public class LinkedQueue <E> {
    private static class Node <E> {
        final E item;
        final AtomicReference<Node<E>> next;
        Node(E item, Node<E> next) {
            this.item = item;
            this.next = new AtomicReference<Node<E>>(next);
        }
    }
    private AtomicReference<Node<E>> head
        = new AtomicReference<Node<E>>(new Node<E>(null, null));
    private AtomicReference<Node<E>> tail = head;
    public boolean put(E item) {
        Node<E> newNode = new Node<E>(item, null);
        while (true) {
            Node<E> curTail = tail.get();
            Node<E> residue = curTail.next.get();
            if (curTail == tail.get()) {
                if (residue == null) /* A */ {
                    if (curTail.next.compareAndSet(null, newNode)) /* C */ {
                        tail.compareAndSet(curTail, newNode) /* D */ ;
                        return true;
                    }
                } else {
                    tail.compareAndSet(curTail, residue) /* B */;
                }
            }
        }
    }
}
```
[非阻塞算法简介](https://www.ibm.com/developerworks/cn/java/j-jtp04186/)   

### BlockingQueue及其子类
---

### CopyOnWriteArrayList
---

add方法： 
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
get方法:
```java
public E get(int index) {
    return get(getArray(), index);
}

@SuppressWarnings("unchecked")
private E get(Object[] a, int index) {
    return (E) a[index];
}
```
setArray与getArray：
```java
final Object[] getArray() {
    return array;
}
final void setArray(Object[] a) {
    array = a;
}
```
remove：
```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```
iterator:
```java

```

### Fork-join框架原理分析
---
Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。类似于Map-Reduce思想，我们只需要在覆盖的方法中分割任务及汇总结果即可。

ForkJoin框架要求分割出的子任务之间互不相关，这恰好是分治思想的要求，所以某种程度上可以把 Fork/Join 模式看作并行版本的 Divide and Conquer 策略，仅仅关注如何划分任务和组合中间结果，将剩下的事情丢给 Fork/Join 框架。  

Demo实现如下：  
```java
public class CountTask extends RecursiveTask<Integer> {

    private static final int THRESHOLD = 2;
    private int start;
    private int end;

    @Override
    protected Integer compute() {
        int sum = 0;

        boolean canCompute = (end - start) <= THRESHOLD;
        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            int middle = (start + end) / 2;
            CountTask leftTask = new CountTask(start, middle);
            CountTask rightTask = new CountTask(middle + 1, end);
            // ForkJoinPool内部采用什么策略将任务分配到不同工作线程的工作队列中的呢？？？
            leftTask.fork(); // 向其内部工作队列中添加新的任务
            rightTask.fork();
            // 如何做到阻塞直到两个分割后的任务执行完才继续向下执行呢？？？有可能是当前工作线程在join方法中停止执行当前任务，
            // 而是执行刚刚提交的leftTask
            int leftResult = leftTask.join();
            int rightResult = rightTask.join();
            sum = leftResult + rightResult;
        }
        return sum;
    }

    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }
    
    /**
     * ForkJoinPool 使用submit 或 invoke 提交的区别：invoke是同步执行，调用之后需要等待任务完成，才能执行后面的代码；submit是
     * 异步执行，只有在Future调用get的时候会阻塞。
     */
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        CountTask task = new CountTask(1, 4);
        Future<Integer> result = forkJoinPool.submit(task);
        Future<Integer> result1 = forkJoinPool.submit(task);
        System.out.println(result.get());
        System.out.println(result1.get());
    }
}
```

上面的代码可以改进的地方是：  执行子任务调用fork方法并不是最佳的选择，最佳的选择是invokeAll方法。
```java
leftTask.fork(); rightTask.fork(); 
替换为 invokeAll(leftTask, rightTask);
```
对于Fork/Join模式，假如Pool里面线程数量是固定的，那么调用子任务的fork方法相当于A先分工给B，然后A当监工不干活，B去完成A交代的任务。所以上面的模式相当于浪费了一个线程。那么如果使用invokeAll相当于A分工给B后，A和B都去完成工作。这样可以更好的利用线程池，缩短执行的时间。  

Fork-Join内部实现原理可以简单理解为：ForkJoinPool内部有多个工作线程ForkJoinWorkerThread，每个工作线程维护一个自己的工作队列，处理其中的任务，ForkJoinPool负责将提交的任务分配给不同的工作线程；

ForkJoin也是线程池的实现之一，其父类是AbstractExecutorService，它与ThreadPoolExecutor同级。  

**工作窃取算法：即某个线程从其它队列中窃取任务来执行。**为什么要使用这个算法呢？？？假如我们需要处理一个比较大的任务，可以把这个任务分割为若干个互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列中的任务，线程与队列一一对应，比如A线程负责处理A队列里的任务。但是有的线程会先把自己队列里的任务干完，而其他线程的工作队列里还有任务等待处理，干完活的线程与其等着，不如去帮其他线程干活，于是他就去其他线程的工作队列里窃取一个任务来执行。而在这是就出现了多个线程访问同一个工作队列的情况，为了减少窃取任务的线程和被窃取任务的线程之间的竞争，通常会使用双端队列，被窃取任务的线程永远从双端队列头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。   

使用Fork-Join时，如果任务层次划分的很深，一直得不到返回，可能出现两种情况：第一、系统内线程数量越积越多，导致性能严重下降。第二、函数的调用层次变的很深，最终导致栈溢出。不同版本的JDK内部实现机制可能有差异，从而导致其表现不同。比如在JDK8中就可能抛出StackOverflowError。  

此外，ForkJoin线程池使用一个无锁的栈来管理空闲线程。如果一个工作线程暂时取不到可用的任务，则可能会被挂起，挂起的线程将会被压入由线程池维护的栈中。将来有任务可用时，再从栈中唤醒这些线程。  

**使用Fork-Join实现并行快排示例：**  
```java
package tests;
 
import static org.junit.Assert.*;
 
import java.util.Arrays;
import java.util.Random;
import java.util.concurrent.TimeUnit;
 
import jsr166y.forkjoin.ForkJoinPool;
import jsr166y.forkjoin.ForkJoinTask;
import jsr166y.forkjoin.RecursiveAction;
 
import org.junit.Before;
import org.junit.Test;
 
class SortTask extends RecursiveAction {
    final long[] array;
    final int lo;
    final int hi;
    private int THRESHOLD = 0; //For demo only
 
    public SortTask(long[] array) {
        this.array = array;
        this.lo = 0;
        this.hi = array.length - 1;
    }
 
    public SortTask(long[] array, int lo, int hi) {
        this.array = array;
        this.lo = lo;
        this.hi = hi;
    }
 
    protected void compute() {
        if (hi - lo < THRESHOLD)
            sequentiallySort(array, lo, hi);
        else {
            int pivot = partition(array, lo, hi);
            System.out.println("\npivot = " + pivot + ", low = " + lo + ", high = " + hi);
            System.out.println("array" + Arrays.toString(array));
            coInvoke(new SortTask(array, lo, pivot - 1), new SortTask(array,
                    pivot + 1, hi));
        }
    }
 
    private int partition(long[] array, int lo, int hi) {
        long x = array[hi];
        int i = lo - 1;
        for (int j = lo; j < hi; j++) {
            if (array[j] <= x) {
                i++;
                swap(array, i, j);
            }
        }
        swap(array, i + 1, hi);
        return i + 1;
    }
 
    private void swap(long[] array, int i, int j) {
        if (i != j) {
            long temp = array[i];
            array[i] = array[j];
            array[j] = temp;
        }
    }
 
    private void sequentiallySort(long[] array, int lo, int hi) {
        Arrays.sort(array, lo, hi + 1);
    }
}
 
public class TestForkJoinSimple {
    private static final int NARRAY = 16; //For demo only
    long[] array = new long[NARRAY];
    Random rand = new Random();
 
    @Before
    public void setUp() {
        for (int i = 0; i < array.length; i++) {
            array[i] = rand.nextLong()%100; //For demo only
        }
        System.out.println("Initial Array: " + Arrays.toString(array));
    }
 
    @Test
    public void testSort() throws Exception {
        ForkJoinTask sort = new SortTask(array);
        ForkJoinPool fjpool = new ForkJoinPool();
        fjpool.submit(sort);
        fjpool.shutdown();
 
        fjpool.awaitTermination(30, TimeUnit.SECONDS);
 
        assertTrue(checkSorted(array));
    }
 
    boolean checkSorted(long[] a) {
        for (int i = 0; i < a.length - 1; i++) {
            if (a[i] > (a[i + 1])) {
                return false;
            }
        }
        return true;
    }
}

```


API文档翻译：  
```java

```
从submit一个任务的过程开始分析：  
```java
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
    if (task == null)
        throw new NullPointerException();
    externalPush(task);
    return task;
}
final void externalPush(ForkJoinTask<?> task) {
    WorkQueue[] ws; WorkQueue q; int m;
    int r = ThreadLocalRandom.getProbe();
    int rs = runState;
    if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
        (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
        U.compareAndSwapInt(q, QLOCK, 0, 1)) {
        ForkJoinTask<?>[] a; int am, n, s;
        if ((a = q.array) != null &&
            (am = a.length - 1) > (n = (s = q.top) - q.base)) {
            int j = ((am & s) << ASHIFT) + ABASE;
            U.putOrderedObject(a, j, task);
            U.putOrderedInt(q, QTOP, s + 1);
            U.putIntVolatile(q, QLOCK, 0);
            if (n <= 1)
                signalWork(ws, q);
            return;
        }
        U.compareAndSwapInt(q, QLOCK, 1, 0);
    }
    externalSubmit(task);
}
private void externalSubmit(ForkJoinTask<?> task) {
    int r;                                    // initialize caller's probe
    if ((r = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        r = ThreadLocalRandom.getProbe();
    }
    for (;;) {
        WorkQueue[] ws; WorkQueue q; int rs, m, k;
        boolean move = false;
        if ((rs = runState) < 0) {
            tryTerminate(false, false);     // help terminate
            throw new RejectedExecutionException();
        }
        else if ((rs & STARTED) == 0 ||     // initialize
                 ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
            int ns = 0;
            rs = lockRunState();
            try {
                if ((rs & STARTED) == 0) {
                    U.compareAndSwapObject(this, STEALCOUNTER, null,
                                           new AtomicLong());
                    // create workQueues array with size a power of two
                    int p = config & SMASK; // ensure at least 2 slots
                    int n = (p > 1) ? p - 1 : 1;
                    n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                    n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                    workQueues = new WorkQueue[n];
                    ns = STARTED;
                }
            } finally {
                unlockRunState(rs, (rs & ~RSLOCK) | ns);
            }
        }
        else if ((q = ws[k = r & m & SQMASK]) != null) {
            if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                ForkJoinTask<?>[] a = q.array;
                int s = q.top;
                boolean submitted = false; // initial submission or resizing
                try {                      // locked version of push
                    if ((a != null && a.length > s + 1 - q.base) ||
                        (a = q.growArray()) != null) {
                        int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                        U.putOrderedObject(a, j, task);
                        U.putOrderedInt(q, QTOP, s + 1);
                        submitted = true;
                    }
                } finally {
                    U.compareAndSwapInt(q, QLOCK, 1, 0);
                }
                if (submitted) {
                    signalWork(ws, q);
                    return;
                }
            }
            move = true;                   // move on failure
        }
        else if (((rs = runState) & RSLOCK) == 0) { // create new queue
            q = new WorkQueue(this, null);
            q.hint = r;
            q.config = k | SHARED_QUEUE;
            q.scanState = INACTIVE;
            rs = lockRunState();           // publish index
            if (rs > 0 &&  (ws = workQueues) != null &&
                k < ws.length && ws[k] == null)
                ws[k] = q;                 // else terminated
            unlockRunState(rs, rs & ~RSLOCK);
        }
        else
            move = true;                   // move if busy
        if (move)
            r = ThreadLocalRandom.advanceProbe(r);
    }
}
```
参考文章：  
[并发之Fork/Join框架使用及注意点](https://www.imooc.com/article/24822) 
[JDK 7 中的 Fork/Join 模式](https://www.ibm.com/developerworks/cn/java/j-lo-forkjoin/index.html) 
[Java 理论与实践-应用 fork-join 框架](https://www.ibm.com/developerworks/cn/java/j-jtp11137.html) 

### ThreadPoolExecutor原理分析
---
API文档翻译：  
```java
一个ExecutorService的实现，它使用可能的几个池化线程执行每个提交的任务，通常使用Executors工厂方法配置。

线程池解决了两个不同的问题：它们通常在执行大量异步任务时提供改进的性能，这是由于减少了每个任务的调用开销，并且它们提供了一种绑定和管理资源的方法，包括执行集合时所消耗的线程。 任务。 每个ThreadPoolExecutor还维护一些基本统计信息，例如已完成任务的数量。

为了在各种上下文中有用，该类提供了许多可调参数和可扩展性钩子方法。 但是，程序员被要求使用更方便的Executors工厂方法Executors.newCachedThreadPool（）（无界线程池，具有自动线程回收），Executors.newFixedThreadPool（int）（固定大小线程池）和Executors.newSingleThreadExecutor（）（单个背景） thread），为最常见的使用场景预配置设置。 否则，在手动配置和调整此类时，请使用以下指南：

。。。

此类的大多数扩展都会覆盖一个或多个受保护的钩子方法。 例如，这是一个添加简单暂停/恢复功能的子类：

class PausableThreadPoolExecutor extends ThreadPoolExecutor {
   private boolean isPaused;
   private ReentrantLock pauseLock = new ReentrantLock();
   private Condition unpaused = pauseLock.newCondition();

   public PausableThreadPoolExecutor(...) { super(...); }

   protected void beforeExecute(Thread t, Runnable r) {
     super.beforeExecute(t, r);
     pauseLock.lock();
     try {
       while (isPaused) unpaused.await();
     } catch (InterruptedException ie) {
       t.interrupt();
     } finally {
       pauseLock.unlock();
     }
   }

   public void pause() {
     pauseLock.lock();
     try {
       isPaused = true;
     } finally {
       pauseLock.unlock();
     }
   }

   public void resume() {
     pauseLock.lock();
     try {
       isPaused = false;
       unpaused.signalAll();
     } finally {
       pauseLock.unlock();
     }
   }
 }
```

### ScheduledThreadPoolExecutor原理分析
---
API文档翻译：
```java
ScheduledThreadPoolExecutor可以使得提交的任务在给定的延迟时间之后执行或者周期性执行。当需要更多的工作线程的时候，这个类相比于Timer来说使用起来更加合适。  

延时任务在enable之后可以被立即execute，但是究竟过多长时间被execute是不能被保证的。TASK执行的顺序是按照enable时的时间排出的先进先出的顺序来决定的。  

提交的任务在运行之前(是指enable之前还是enable之后被执行之前)被取消时，这个任务将不会被执行。 默认情况下，此类已取消的任务不会自动从工作队列中删除，直到其延迟时间过去。 虽然这可以进一步检查和监控，但也可能导致取消任务的无限制保留。 要避免这种情况，请将setRemoveOnCancelPolicy（boolean）设置为true，这会导致在取消时立即从工作队列中删除任务。

通过scheduleAtFixedRate或scheduleWithFixedDelay计划的任务的连续执行不会重叠。 虽然不同的任务的execute可能被不同的线程完成，但是先前执行的效果happens-before在后续执行的效果。

虽然这个类继承自ThreadPoolExecutor，但是一些继承的调优方法对它没用。 特别是，因为它使用corePoolSize线程和无界队列作为固定大小的池，所以对maximumPoolSize的调整没有任何有用的效果。 此外，将corePoolSize设置为零或使用allowCoreThreadTimeOut几乎绝不是一个好主意，因为一旦它们有资格运行，这可能会使池没有线程来处理任务。
```
使用Demo：
```java
public class ThreadPoolStudy {

    public static void main(String[] args) {
        // ScheduledExecutorService可以使任务 定期或延时 执行
        ScheduledExecutorService executor =  (ScheduledThreadPoolExecutor) Executors.newScheduledThreadPool(10);
        for (int i = 0; i <= 5; i++)
        {
            Task task = new Task("Task " + i);
            System.out.println("A new task has been added : " + task.getName());
            // 让提交的任务在等待指定时间之后才有资格被选中执行
            executor.schedule(task, i * 3, TimeUnit.SECONDS);
        }
        executor.shutdown();
    }

    static class Task implements Runnable
    {
        private String name;

        public Task(String name)
        {
            this.name = name;
        }

        public String getName() {
            return name;
        }

        @Override
        public void run()
        {
            try
            {
                Long duration = (long) (Math.random() * 2);
                System.out.println("Doing a task during : " + name);
                TimeUnit.SECONDS.sleep(duration);
            }
            catch (InterruptedException e)
            {
                e.printStackTrace();
            }
        }
    }
}
```

### AIO学习待完成