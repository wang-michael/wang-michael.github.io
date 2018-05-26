# java.util.concurrent包学习笔记
## 一、概述 
java.util.concurrent中更高级的工具分成3类：Executor Framework、并发集合(Concurrent Collection)以及同步器(Synchronizer)。
1、学习java中同步器(Synchronizer) Semaphore Exchanger
2、学习java中并发集合结合传统集合类来学习
3、学习java中Executor Framework

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

###CyclicBarrier
---