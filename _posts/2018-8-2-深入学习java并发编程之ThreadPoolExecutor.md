---
layout:     post
title:      深入学习java并发编程之ThreadPoolExecutor
date:       2018-8-2
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Java基础
    - 并发
---
>本文主要介绍了自己对于juc包中的ThreadPoolExecutor类中相关API的学习及实现原理的理解。    

_ _ _
### **前言**
使用线程池来管理线程有3个好处：
1. 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。

下面就来看一个线程池的Demo实现：  
```java
public interface ThreadPool<Job extends Runnable>{
   //执行一个任务(Job),这个Job必须实现Runnable
   void execute(Job job);
  //关闭线程池
  void shutdown();
  //增加工作者线程，即用来执行任务的线程
  void addWorkers(int num);
  //减少工作者线程
  void removeWorker(int num);
  //获取正在等待执行的任务数量
  void getJobSize();
}
```
客户端可以通过execute(Job)方法将Job提交入线程池来执行，客户端完全不用等待Job的执行完成。除了execute(Job)方法以外，线程池接口提供了增加/减少工作者线程以及关闭线程池的方法。这里工作者线程代表着一个重复执行Job的线程，而每个客户端提交的Job都会进入到一个工作队列中等待工作者线程的处理。  

接下来是上述线程池接口的默认实现，示例代码如下：
```java
public class DefaultThreadPool<Job extends Runnable> implements ThreadPool<Job>{

    // 线程池维护工作者线程的最大数量
    private static final int MAX_WORKER_NUMBERS = 10;
    // 线程池维护工作者线程的默认值
    private static final int DEFAULT_WORKER_NUMBERS = 5;
    // 线程池维护工作者线程的最小数量
    private static final int MIN_WORKER_NUMBERS = 1;
    // 维护一个工作列表,里面加入客户端发起的工作
    private final LinkedList<Job> jobs = new LinkedList<Job>();
    // 工作者线程的列表
    private final List<Worker> workers = Collections.synchronizedList(new ArrayList<Worker>());
    // 工作者线程的数量
    private int workerNum;
    // 每个工作者线程编号生成
    private AtomicLong threadNum = new AtomicLong();

    //生成线程池
    public DefaultThreadPool() {
        this.workerNum = DEFAULT_WORKER_NUMBERS;
        initializeWorkers(this.workerNum);
    }

    public DefaultThreadPool(int num) {
        if (num > MAX_WORKER_NUMBERS) {
            this.workerNum =DEFAULT_WORKER_NUMBERS;
        } else {
            this.workerNum = num;
        }
        initializeWorkers(this.workerNum);
    }
    //初始化每个工作者线程
    private void initializeWorkers(int num) {
        for (int i = 0; i < num; i++) {
            Worker worker = new Worker();
            //添加到工作者线程的列表
            workers.add(worker);
            //启动工作者线程
            Thread thread = new Thread(worker);
            thread.start();
        }
    }

    public void execute(Job job) {
        if (job != null) {
        //根据线程的"等待/通知机制"这里必须对jobs加锁
            synchronized (jobs) {
                jobs.addLast(job);
                jobs.notify();
            }
        }
    }
    //关闭线程池即关闭每个工作者线程
     public void shutdown() {
        for (Worker w : workers) {
            w.shutdown();
        }
    }
    //增加工作者线程
    public void addWorkers(int num) {        
        // 为什么这里不对工作者线程列表workers加锁呢？？？
        synchronized (jobs) {
            // 限制新增的worker数量不能超过最大值
            if (num + this.workerNum > MAX_WORKER_NUMBERS) {
                num = MAX_WORKER_NUMBERS - this.workerNum;
            }
            initializeWorkers(num);
            this.workerNum += num;
        }
    }
    //减少工作者线程
    public void removeWorker(int num) {
        // 为什么这里不对工作者线程列表workers加锁呢？？？
        synchronized (jobs) {
            if(num>=this.workerNum){
                  throw new IllegalArgumentException("超过了已有的线程数量");
            }
            for (int i = 0; i < num; i++) {
                Worker worker = workers.get(i);
                if (worker != null) {
                //关闭该线程并从列表中移除
                    worker.shutdown();
                    workers.remove(i);
                }
            }
            this.workerNum -= num;
        }
    }

    public int getJobSize() {
        // TODO Auto-generated method stub
        return workers.size();
    }
    //定义工作者线程
    class Worker implements Runnable {
       // 表示是否运行该worker
       private volatile boolean running = true;

       public void run() {
            while (running) {
                Job job = null;
                //线程的等待/通知机制
                synchronized (jobs) {
                    if (jobs.isEmpty()) {
                        try {
                            jobs.wait();//线程等待唤醒
                        } catch (InterruptedException e) {
                            //感知到外部对该线程的中断操作，返回
                            Thread.currentThread().interrupt();
                            return;
                        }
                    }
                    // 取出一个job
                    job = jobs.removeFirst();
                }
                //执行job
                if (job != null) {
                    job.run();
                }
            }
        }

        // 终止该线程
        public void shutdown() {
            running = false;
        }
   } // end class Worker
} // end class DefaultThreadPool
```
从上面线程池的Demo实现中可以看到，当客户端调用execute(Job)方法时，会不断的向任务列表jobs中添加job，而每个工作者线程会不断的从jobs上取出一个Job进行执行，当jobs为空时，工作者线程进入等待状态。  

添加一个Job后，对工作队列jobs调用了其notify方法，而不是notifyAll方法，因为能够确定有工作者线程被唤醒，这时使用notify方法将会比使用notifyAll方法获得更小的开销(避免将等待队列中的线程全部移动到阻塞队列中)。     

可以看到，**线程池的本质就是使用了一个线程安全的工作队列连接工作者线程和客户端线程，客户端线程将任务放入工作队列后便返回，而工作者线程则不断的从工作队列中取出工作并执行。当工作队列为空时，所有的工作线程均等待在工作队列上，当有客户端提交了一个任务之后会通知任意一个工作者线程，随着大量的任务被提交，更多的工作者线程会被唤醒。**  

实际上ThreadPoolExecutor实现核心原理也是上面所述，只是相比于上述Demo实现的更为完备，接下来就来看看ThreadPoolExecutor的内部实现机制。  

_ _ _
### **ThreadPoolExecutor类源码分析**
其UML图如下：  
<img src="/img/2018-8-2/ThreadPoolExecutorUML.png" width="500" height="500" alt="ThreadPoolExecutorUML UML图" />
<center>图1：ThreadPoolExecutorUML UML图</center>   

#### **构造方法与成员变量**
成员变量如下：  
```java
/**
 * 线程池中的主要控制状态，是一个atomic Integer，包装了两个抽象域：
 *  workerCount ： 标记有效线程的数量(29位)
 *  runState ： 标识运行状态，running、shutting down等等(3位)
 *  
 * 为了将它们包装到一个int当中，我们限制workCount变量使用ctl中的29位，代表最多有2^29-1(大约500 million个线程)，如果这在将来
 * 成为了一个问题，也就是说可以支持比500 million更多的线程，可以将ctl由AtomicInteger变为AtomicLong，现在使用AtomicInteger
 * 可以更高效一些。
 * 
 * workerCount代表已经被准许start但是没有被允许stop的线程的数量，该值可能与实际线程的实际数量暂时不同，例如，
 * 当ThreadFactory在被要求创建线程却创建失败时，以及退出线程在终止之前仍在执行某些操作时。 用户可见的池大小将
 * 报告为workers set的当前大小。
 * 
 * runState提供主要的生命周期控制，可取以下值：
 *  RUNNING： 可接收新的任务，同时也在处理队列中的任务
 *  SHUTDOWN： 不再接收新的任务，但是会继续处理队列中的任务
 *  STOP: 不接受新的任务，不处理已经在队列中的任务，中断正在执行中的任务
 *  TIDYING：所有任务都已经被终止，workerCount是0，将状态转换为TIDYING状态的线程将会执行terminated()方法
 *  TERMINATED: terminated() 方法已经执行完毕
 *  
 * 这些值之间的数字顺序很重要，以允许有序比较。 runState随着时间的推移单调增加，但不需要命中每个状态。 过渡是：
 *  RUNNING -> SHUTDOWN：在调用shutdown（）时，可能隐含在finalize（）中
 *  (RUNNING or SHUTDOWN) -> STOP：在调用shutdownNow（）时
 *  SHUTDOWN -> TIDYING：当pool和queue都为空时
 *  STOP -> TIDYING：当pool为空时
 *  TIDYING -> TERMINATED：当回调方法terminated()已经执行完毕
 *  
 *  具体值：RUNNING:-536870912，SHUTDOWN:0，STOP:536870912，TIDYING:1073741824，TERMINATED:1610612736
 *   
 * 当state变为TERMINATED的时候，等待在awaitTermination()上的线程会返回
 * 
 * 检测从SHUTDOWN到TIDYING的转换不如你想要的那么简单，因为在SHUTDOWN状态期间队列可能在非空后变为空，反之亦然，
 * 但我们只能在看到它为空后看到workerCount时终止 是0（有时需要重新检查 - 见下文介绍）
 */
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3; // 实际就是29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1; // 工作线程的最大数量

// runState 存储在高位
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

private static int ctlOf(int rs, int wc) { return rs | wc; }
// 根据ctl获取runState和workerCount
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }

/**
 * 保存提交的任务的队列。我们不要求workQueue.poll（）返回null必然意味着workQueue.isEmpty（），因此只依赖于isEmpty来查看队列
 * 是否为空（我们必须这样做，例如在决定是否从SHUTDOWN转换为TIDYING时）。这适用于特殊目的队列，例如DelayQueues，允许poll（）返
 * 回null，即使延迟到期后它可能稍后返回非null
 */
private final BlockingQueue<Runnable> workQueue;
/**
 * 对相关workers set和相关bookkeeping加锁。
 * 
 * 虽然我们可以使用某种并发集合，但通常最好使用锁定。 其中的原因是这会使interruptIdleWorkers序列化，从而避免不必要的中断风暴，
 * 特别是在shutdown期间。否则退出线程会同时中断那些尚未中断的线程。 它还简化了largestPoolSize等相关的一些统计的bookkeeping。
 * 
 * 我们还在shutdown和shutdownNow上保持mainLock，为了在同时单独检查中断和实际中断的权限时确保工作集稳定。
 */
private final ReentrantLock mainLock = new ReentrantLock();
/**
 * 包含池中所有的工作线程的set。只有在获取到mainLock之后才能访问此set
 */
private final HashSet<Worker> workers = new HashSet<Worker>();
/**
 * Wait condition to support awaitTermination
 */
private final Condition termination = mainLock.newCondition();
/**
 * 跟踪最大的池的大小，获取mainLock之后才可访问
 */
private int largestPoolSize;
/**
 * 完成任务的计数器。 仅在终止工作线程时更新。 获取mainLock之后才可访问。
 */
private long completedTaskCount;

/*
 * 所有用户控制参数都声明为volatile的，因此正在进行的操作基于最新的值，但不需要锁定，因为在同步改变其他的相关的行为时，没有内部
 * 不变量依赖于它们。 
 */

/**
 * 创建新线程的工厂，所有线程都是使用这个工程来创建的(通过使用addWroker方法)。所有的调用者都要做好addWorker失败的准备，这可能反
 * 映了限制线程数量的系统或用户策略。即使它不被视为错误，但创建线程失败可能导致新任务被拒绝或现有的任务仍然滞留在队列中。
 * 
 * 我们进一步保留池不变量，即使遇到可能在尝试创建线程时抛出OutOfMemoryError的错误。由于需要在Thread.start中分配本机堆栈，
 * 因此这些错误相当普遍，用户希望执行清理池关闭以进行清理。可能有足够的内存可供清理代码完成而不会遇到另一个OutOfMemoryError。
 */
private volatile ThreadFactory threadFactory;
/**
 * 在执行时工作队列饱和或者shutdown时调用。
 */
private volatile RejectedExecutionHandler handler;
/**
 * 核心线程之外(当allowCoreThreadTimeOut被设置时，空闲线程也有可能)的空闲线程使用的超时时间，超过此时间之后则关闭线程。否则这些线程会一直等待新提交的工作的到来。  
 */
private volatile long keepAliveTime;
/**
 * 默认为false，代表当核心线程空闲时会等待；当为true的时候，代表核心线程也使用allowCoreThreadTimeOut作为
 * 超时时间
 */
private volatile boolean allowCoreThreadTimeOut;
/**
 * 核心线程池中工作线程的最小数量
 */
private volatile int corePoolSize;
/**
 * 最大的工作线程的数量
 */
private volatile int maximumPoolSize;
/**
 * 默认的拒绝处理策略
 */
private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
/**
 * 调用shutdown和shutdownNow方法时调用者所需的权限。我们还要求（请参阅checkShutdownAccess）调用者有实际中断工作集中的线程
 * （由Thread.interrupt控制，它依赖于ThreadGroup.checkAccess，而ThreadGroup.checkAccess又依赖于
 * SecurityManager.checkAccess）。 的权限，只有在这些权限检查通过时才会尝试shutdown。
 * 
 * 所有实际的Thread.interrupt的调用(参阅interruptIdleWorkers和interruptWorkers)都忽略了SecurityExceptions，这意味着对于
 * interrupt的尝试的失败不会有任何提示。在关闭的情况下，它们不应该失败，除非SecurityManager具有不一致的策略，有时允许访问线程，有
 * 时不允许。在这种情况下，实际中断线程失败可能会禁用或延迟完全终止。interruptIdleWorkers的其他用途是建议性的，实际中断失败只会延迟对配置更改的响应，因此不会异常处理。
 */
private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");
/* The context to be used when executing the finalizer, or null. */
private final AccessControlContext acc;
```
构造方法：  
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), handler);
}
/**
 * 实际的构造方法
 * threadFactory:当Executor创建一个新线程时使用的factory 
 * 
 * handler: 表示当拒绝处理任务时的策略，有以下四种取值:
 *  ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
 *  ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
 *  ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
 *  ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

#### **execute()方法分析**
直接看代码：  
```java
/**
 * 执行提交的给定任务的方法。提交的任务既有可能由新创建的线程执行，也可能由池中已经存在的线程来执行。
 * 
 * 如果任务由于某种原因不能被提交执行(可能由于executor已经shutdown或者其工作线程容量已经达到最大值)，这个任务就会交由创建线程池
 * 时提供的RejectedExecutionHandler来处理。
 */
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    
    /**
     * 处理过程分为以下三步：
     *   
     * 1.如果线程池中运行的工作线程数小于corePoolSize，就尝试新开一个线程运行提交的任务。对addWorker的调用以原子方式检查
     * runState和workerCount，因此通过返回false来防止在不应该添加线程时发生的错误警报。
     * 
     * 2.如果一个任务已经成功入队，我们仍然需要再次检测是否需要添加一个线程(因为自从上次检测之后存在的那些线程可能已经死掉了)或者
     * 进入到此方法时，线程池已经shutdown了。所以我们重新检查state，如果有必要的话回滚入队过程，或者新创建一个线程。
     * 
     * 3.如果任务不能入队，那么我们就尝试添加一个新的线程。如果失败了，我们就知道了线程池已经关闭或者已经饱和，并拒绝这个任务。  
     */
    int c = ctl.get(); 
    // 步骤1
    if (workerCountOf(c) < corePoolSize) { // 线程池中工作线程的数量小于corePoolSize
        if (addWorker(command, true)) // 尝试新建一个工作线程来处理提交的任务
            return;
        c = ctl.get();
    }
    // 步骤2
    //如果线程池处于RUNNING状态则尝试把请求加入等待队列
    if (isRunning(c) && workQueue.offer(command)) {
        //再次检查，获取线程池控制状态
        int recheck = ctl.get(); // 如果执行这句代码获取到了recheck代表线程池在运行；执行下句代码的时候线程池不再运行了，怎么办？
        //线程池不处于RUNNING状态，将命令从等待队列中移除
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 步骤3
    //如果请求加入等待队列失败，则尝试创建新线程
    else if (!addWorker(command, false))
        reject(command);
}
```
再来总结一下ThreadPoolExecutor.execute(Runnable command)的执行过程：  
1. 如果当前运行的线程少于corePoolSize,则创建新线程来执行任务(注意，执行这一步骤需要获取全局锁)
2. 如果运行的线程等于或多于corePoolSize,则将任务加入BlockingQueue
3. 如果无法将任务加入BlockingQueue(队列已满)，则创建新的线程来处理任务(注意，执行这一步骤需要获取全局锁)
4. 如果创建新线程将使当前运行的线程超出maximumPoolSize,任务将被拒绝，并调用拒绝策略来处理

ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行execute()方法时，尽可能的避免获取全局锁。在ThreadPoolExecutor完成预热之后(当前运行的线程数大于等于corePoolSize)，几乎所有的execute()方法调用都是执行步骤2，而步骤2不需要获取全局锁。
  
#### **addWorker()方法分析**
```java
/**
 * 根据当前线程池的状态和给定的边界检查是否可以添加一个新的工作线程。如果可以的话，就创建一个新的工作线程，并把这个提交的任务作为其第一个任务来执
 * 行。方法在pool stopped或者shutdown时候返回false。当threadfactory创建新的线程失败的时候也会返回false。线程创建失败或者是由于threadfactory
 * 返回了null，或者是由于异常(比如Thread.start()方法中抛出的OutOfMemoryError)，我们就roll back。
 * 
 * 参数firstTask代表新线程创建后应该首先执行的任务，使用初始第一个任务（在方法execute（）中）创建工作程序，以便在少于corePoolSize线程时（在这
 * 种情况下我们始终启动一个）或队列已满（在这种情况下我们必须绕过队列）时绕过排队。最初空闲线程通常通过prestartCoreThread创建或替换其他垂死的工
 * 作者。
 * 
 * 参数core，true代表使用corePoolSize作为边界，否则使用maximumPoolSize。此处使用布尔指示符而不是值，以确保在检查其他池状态后读取新的值。 
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        /**
         * 直接返回的条件，‘与’关系：
         *  1.状态大于等于SHUTDOWN，初始的ctl为RUNNING，小于SHUTDOWN
         *  2.!(状态为SHUTDOWN && 第一个任务为null && worker队列不为空)
         */
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // CAS增加worker的数量并跳出外层循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // 新创建工作线程加入到工作队列，并且另其处理提供的firstTask
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 获取全局锁，何时需要获取全局锁呢？？？
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 添加失败
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
// 添加失败后回滚worker的创建动作，即将worker从workers集合中删除，并原子性的减少workerCount。
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

到目前为止已经看完了提交任务的过程，接下来看看工作线程在执行完firstTask之后是如何从任务队列中取新任务执行的。

#### **工作线程的工作过程**  
内部类Worker中包装了执行任务的工作线程，它是ThreadPoolExecutor的核心。每个线程池中，有为数不等的Worker对象，每个Worker对象中，包含一个需要立即执行的新任务和已经执行完成的任务数量，Worker本身，是一个Runnable对象，不是Thread对象。它内部封装一个Thread对象，用此对象执行本身的run方法，而这个Thread对象则由ThreadPoolExecutor提供的ThreadFactory对象创建新的线程。（将Worker和Thread分离的好处是，如果我们的业务代码，需要对于线程池中的线程，赋予优先级、线程名称、线程执行策略等其他控制时，可以实现自己的ThreadFactory进行扩展，无需继承或改写ThreadPoolExecutor。）
```java
// 这里把Worker类作为同步器来使用的意义在哪呢？？？
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;
    /** Thread this worker is running in.  Null if factory fails. */
    //worker持有的线程
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    //worker正在执行的任务 ，可能为null.
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;
    /**
     * 创建Worker时会同时创建一个新线程.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        /把Worker传递给新建的线程，当线程执行是会调用Worker的run方法。
        this.thread = getThreadFactory().newThread(this);
    }
    /** Delegates main run loop to outer runWorker  */
    //线程执行时会调用该方法
    public void run() {
        runWorker(this);
    }
    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```
接下来来看runWorker方法： 
```java
final void runWorker(Worker w) {
	//获取当前线程
   Thread wt = Thread.currentThread();
   //获取w的任务
   Runnable task = w.firstTask;
   w.firstTask = null;
   //释放锁（设置state为0，允许中断）
   w.unlock(); // allow interrupts
   boolean completedAbruptly = true;
   try {
   	   //任务不为null或者阻塞队列还存在任务
       while (task != null || (task = getTask()) != null) {
       	   //获取锁, 表示当前工作线程正在工作
           w.lock();
           // If pool is stopping, ensure thread is interrupted;
           // if not, ensure thread is not interrupted.  This
           // requires a recheck in second case to deal with
           // shutdownNow race while clearing interrupt
           //线程池的运行状态至少应该高于STOP || 线程被中断 && 再次检查，线程池的运行状态至少应该高于STOP && // wt线程（当前线程）没有被中断
           if ((runStateAtLeast(ctl.get(), STOP) ||
                (Thread.interrupted() &&
                 runStateAtLeast(ctl.get(), STOP))) &&
               !wt.isInterrupted())
               //中断wt线程（当前线程）
               wt.interrupt();
           try {
           	//beforeExecute方法是ThreadPoolExecutor类的一个方法，没有具体实现，用户可以根据自己需要重载这个方法和后面的afterExecute方法来进行一些统计信息，比如某个任务的执行时间等
               beforeExecute(wt, task);
               Throwable thrown = null;
               try {
               	//执行任务
                   task.run();
               } catch (RuntimeException x) {
                   thrown = x; throw x;
               } catch (Error x) {
                   thrown = x; throw x;
               } catch (Throwable x) {
                   thrown = x; throw new Error(x);
               } finally {
                   afterExecute(task, thrown);
               }
           } finally {
               task = null;
               //增加worker完成的任务数量
               w.completedTasks++;
               w.unlock();
           }
       }
       completedAbruptly = false;
   } finally {
       processWorkerExit(w, completedAbruptly);
   }
}
// 获取任务失败，工作线程工作结束
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```
此函数中会实际执行给定任务（即调用用户重写的run方法），并且当给定任务完成后，会继续从阻塞队列中取任务，直到阻塞队列为空（即任务全部完成）。接下来来看从阻塞队列中取任务的getTask()方法：  
```java
/**
 * 工作线程可能由于以下原因导致获取到的Task为null，之后退出：
 *  1. 当前工作线程的数量已经超过了maximumPoolSize(由于setMaximumPoolSize方法调用带来的影响)
 *  2. pool状态为stopped
 *  3. pool已经关闭并且queue为空
 *  4. 条件allowCoreThreadTimeOut || workerCount > corePoolSize为真，并且经过给定超时时间此工作线程并没有取出任务来
 */
private Runnable getTask() {
       boolean timedOut = false; // Did the last poll() time out?
	//无限循环，确保操作成功
   for (;;) {
       int c = ctl.get();
       int rs = runStateOf(c);
       // Check if queue empty only if necessary.
       //当线程池处于STOP及以上状态时，线程数减一，该线程不使用。
      //当线程处于SHUTDOWN 状态时，并且workQueue请求队列为空，释放该线程。
       if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
           decrementWorkerCount();
           return null;
       }
	//获取当前线程数
       int wc = workerCountOf(c);
       // Are workers subject to culling?
       //如果调用allowCoreThreadTimeOut方法设置为true，则所有线程都有超时时间。
       //如果当前线程数大于核心线程数则该线程有超时时间。
       boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
	//worker数量大于maximumPoolSize
       if ((wc > maximumPoolSize || (timed && timedOut))
       	//workerCount大于1或者worker阻塞队列为空（在阻塞队列不为空时，需要保证至少有一个wc）
           && (wc > 1 || workQueue.isEmpty())) {
           // 比较并减少workerCount
           if (compareAndDecrementWorkerCount(c))
               return null;
           continue;
       }
	//poll方法从阻塞队列中取出对象，如果队列为空，则当前线程阻塞keepAliveTime时间再尝试取出，还是没有就返回null，记录超时状态，在重新进入for循环时才试图终结Worker。
       //take()方法没有超时时间，会一直获取。也就是说在这里不断获取任务，
       //也就是如果线程池处于RUNNING、SHUTDOWN状态时，只要等待队列不为空，那么线程就会一直执行。这也就是线程重用的原理。
       try {
           Runnable r = timed ?
               workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
               workQueue.take();
           if (r != null)
               return r;
           timedOut = true;
       } catch (InterruptedException retry) {
           timedOut = false;
       }
   }
}
```

#### **结束线程池**  
两个方法，shutDown与shutDownNow：
```java
public void shutdown() {
   final ReentrantLock mainLock = this.mainLock;
   mainLock.lock();
   try {
   	//检查shutdown权限
       checkShutdownAccess();
       //把线程池的状态置为SHUTDOWN
       advanceRunState(SHUTDOWN);
       //中断所有空闲线程
       interruptIdleWorkers();
       //调用shutdown方法
       onShutdown(); // hook for ScheduledThreadPoolExecutor
   } finally {
       mainLock.unlock();
   }
   tryTerminate();
}
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) { // w.tryLock()失败代表此worker正在工作
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
shutdown是把线程池状态转为SHUTDOWN，这时等待队列中的任务可以继续执行，但是不会接受新任务了，通过中断方式停止空闲的（根据没有获取锁来确定）线程。  

shutdownNow源码如下：  
```java
public List<Runnable> shutdownNow() {
   List<Runnable> tasks;
   final ReentrantLock mainLock = this.mainLock;
   mainLock.lock();
   try {
       checkShutdownAccess();
       advanceRunState(STOP);
       interruptWorkers();
       //清空队列
       tasks = drainQueue();
   } finally {
       mainLock.unlock();
   }
   tryTerminate();
   return tasks;
}
```
与shutdown的区别在于不仅会中断阻塞的线程，也会中断正在执行任务的线程，并且会清空待处理的任务的队列。shutdownNow()返回值是未被处理的任务列表。

最后简单记录一下线程池几个监控方法的作用：  
* getTaskCount : 线程池已经执行和未执行的任务总数
* getCompletedTaskCount : 线程池已经完成的任务数目
* getLargestPoolSize : 线程池曾创建过线程数最多的线程数
* getPoolSize : 线程池当前线程数目
* getActiveCount : 当前线程池正在执行的任务数目

线程池如果处理CPU密集型任务居多，则需要尽量压榨CPU，大小可以设为NCPU + 1，如果IO密集型任务居多，应配置尽可能多的线程，比如2*NCPU。  

还要注意的一点就是创建线程池时使用的BlockingQueue要设置上界，防止创建太多线程撑爆内存。  

_ _ _
### **总结**
ThreadPoolExecutor是线程池框架的一个核心类，通过对ThreadPoolExecutor的分析，可以知道其对资源进行了复用，并非无限制的创建线程，可以有效的减少线程创建和切换的开销，关于ThreadPoolExecutor的源码就分析到这里。

(完)  

参考文章：        
[ThreadPoolExecutor源码分析（JDK1.8）](http://penghb.com/2017/11/02/java/threadPoolExecutor/)   
