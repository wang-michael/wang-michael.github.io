---
layout:     post
title:      深入学习java并发编程之Wait-Notify&CountDownLatch&CyclicBarrier对比分析
date:       2018-4-29
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:    
    - 并发
    - JDK源码阅读
---
>本文记录了自己对于Wait-Notify、CountDownLatch、CyclicBarrier几种并发流程控制方式的对比分析过程，仅用于个人备忘，如有错误，敬请指出。  

_ _ _
### **从应用层面对比分析**
具体Wait-Notify、CountDownLatch、CyclicBarrier功能介绍这里不再叙述，个人观点来看这三者都是通过**等待-通知机制**来实现的并发流程控制，只不过具体的操作方式有些不同：
* Wait-Notify：**等待线程的数量是不固定的，通知线程的数量可以是一个(调用notifyAll方法)或多个(notify与notifyAll方法结合使用)**，countDownLatch中调用countDown()方法的线程可以类比于wait-notify中调用notify()方法的线程，调用await方法的线程可以类比于调用wait方法的线程；显而易见，countDownLatch在notify等待线程方式上相比于wait-notify方式更强，可以在指定数量的notify线程执行countDown()方法之后才放行等待线程。 但是wait-notify相比于countDownLatch的优势在于可以循环使用。    
* CountDownLatch：**等待线程的数量是不固定的，可指定通知线程的数量**(比如设置countDownLatch(10)可以使得未知数量的等待线程在10个通知线程调用countDown()方法之后继续运行)，countDown方法与await方法配合使用功能比较强大，但缺点在于只能使用一次。
* CyclicBarrier：**等待线程的数量是固定的，每个线程既是等待线程也是通知线程**，当最后一个等待线程(即指定数量中的最后一个线程调用await方法之后)到来时，所有的等待线程会停止阻塞，继续执行；功能没有CountDownLatch强大，但可以循环使用。  

以实现一个简易的数据库连接池为例，在数据库连接池测试类中使用上述三种方式实现并发流程控制，数据库连接池Demo如下：
```java
public class ConnectionPool {

    private LinkedList<Connection> pool = new LinkedList<>();

    public ConnectionPool(int initialSize) {
        if (initialSize > 0) {
            for (int i = 0; i < initialSize; i++) {
                pool.addLast(ConnectionDriver.createConnection());
            }
        }
    }

    public void releaseConnection(Connection connection) {
        if (connection != null) {
            synchronized (pool) {
                pool.addLast(connection);
                pool.notifyAll();
            }
        }
    }

    public Connection fetchConnection(long mills) throws InterruptedException {
        synchronized (pool) {
            if (mills <= 0) {
                while (pool.isEmpty()) {
                    pool.wait();
                }
                return pool.removeFirst();
            } else {
                long future = System.currentTimeMillis() + mills;
                long remaining = mills;
                while (pool.isEmpty() && remaining > 0) {
                    pool.wait(remaining);
                    remaining = future - System.currentTimeMillis();
                }
                Connection result = null;
                if (!pool.isEmpty()) {
                    result = pool.removeFirst();
                }
                return result;
            }
        }
    }
}

public class ConnectionDriver {

    static class ConnectionHandler implements InvocationHandler {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (method.getName().equals("commit")) {
//                System.out.println(Thread.currentThread().getName() +" commit!");
                TimeUnit.MILLISECONDS.sleep(100);
            } else if (method.getName().equals("createStatement")) {
//                System.out.println(Thread.currentThread().getName() +" createStatement!");
            }
            return null;
        }
    }

    // 基于JDK动态代理方式模拟数据库连接创建
    public static final Connection createConnection() {
        return (Connection) Proxy.newProxyInstance(ConnectionDriver.class.getClassLoader(),
                new Class[]{Connection.class}, new ConnectionHandler());
    }
}
```
先来分析一下上面ConnectionPool类中获取连接方法fetchConnection与释放连接方法releaseConnection之间为什么使用wait-notify方式而不是使用其他两种方式实现并发流程控制：执行releaseConnection方法的线程相当于通知线程，在池中没有连接时调用fetchConnection相当于等待线程，显然我们想要的并发流程控制方式是通知线程只需要一个，等待线程的数量是不固定的，并且可以重复通知(每次连接池从没有线程到有线程都需要对等待的所有线程进行通知)，满足这几个条件的上面三种方式中只有wait-notify，所有使用了wait-notify方式进行实现。  

再来分析一下接下来要实现的数据库连接池测试类的业务需求，有两处需要进行并发流程控制：
1. 主线程创建指定数量的获取数据库连接的线程之后需要让这些线程同时开始尝试获取数据库连接。
2. 主线程需要等待所有获取数据库连接的线程执行完毕之后统计连接获取成功的数量和连接获取失败的数量。

在需求1中等待线程的数量是尝试获取数据库连接的所有线程，数量是固定的，通知线程即为主线程，不要求循环通知；需求2中等待线程即为主线程，数量为1个，通知线程的数量为获取数据库连接的所有线程，不要求循环通知。

通过对上面需求1、2的分析可以发现CountDownLatch方式对于这两个需求满足的最好，因此首先选用CountDownLatch方式实现对此简易数据库连接池的测试：
```java
/**
 * Created by michael on 2018/4/28.
 * @author michael
 * 要实现的功能：使数量为threadCount的线程同时开始执行获取连接，每个线程尝试获取连接count次
 *   开始时：等待线程数量为threadCount，通知线程数量为1个
 *   结束时：通知线程数量为threadCount个，等待线程数量为1个
 * 为什么使用CountDownLatch：
 *   1、使用wait-notify无法确定等待线程的数量，需要在wait-notify基础上封装计数。
 *   2、使用CyclicBarrier实现需要等待线程与通知线程数量相同，硬要使用CyclicBarrier实现此功能会造成无用的等待。
 */
public class ConnectionPoolTest {

    static ConnectionPool pool = new ConnectionPool(10);
    static CountDownLatch start = new CountDownLatch(1);
    static CountDownLatch end;


    public static void main(String[] args) throws InterruptedException {
        int threadCount = 50;
        end = new CountDownLatch(threadCount);
        int count = 20;
        AtomicInteger got = new AtomicInteger();
        AtomicInteger notGot = new AtomicInteger();
        for (int i = 0; i < threadCount; i++) {
            Thread thread = new Thread(new ConnectionRunner(count, got, notGot), "connectionThread-" + i);
            thread.start();
        }
        start.countDown();
        end.await();
        System.out.println("total invoke: " + (threadCount * count));
        System.out.println("got connection: " + got);
        System.out.println("not got connection: " + notGot);
    }

    static class ConnectionRunner implements Runnable {

        int count;
        AtomicInteger got;
        AtomicInteger notGot;

        public ConnectionRunner(int count, AtomicInteger got, AtomicInteger notGot) {
            this.count = count;
            this.got = got;
            this.notGot = notGot;
        }

        @Override
        public void run() {
            try {
                start.await();
            } catch (InterruptedException e) {
            }
            while (count > 0) {
                try {
                    Connection connection = pool.fetchConnection(1000);
                    if (connection != null) {
                        try {
                            connection.createStatement();
                            connection.commit();
                        } finally {
                            pool.releaseConnection(connection);
                            got.incrementAndGet();
                        }
                    } else {
                        notGot.incrementAndGet();
                    }
                } catch (Exception ex) {

                } finally {
                    count--;
                }
            }
            end.countDown();
        }
    }
}

输出结果示例：

total invoke: 1000
got connection: 819
not got connection: 181
```
在这里并不是必须使用CountDownLatch实现对上述两个需求的并发流程控制，也可以使用wait-notify方式和CyclicBarrier方式，只不过实现方式没有CountDownLatch方式优雅。  

使用wait-notify方式实现对上述数据库连接池的测试：
```java
public class ConnectionPoolTestWaitNotify {

    /**
     * 线程总数量
     */
    private static final int THREAD_COUNT = 50;
    /**
     * 初始化连接池中有10个连接
     */
    static ConnectionPool pool = new ConnectionPool(10);
    static int threadCount = THREAD_COUNT;
    static final Object START_LOCK = new Object();
    static final Object END_LOCK = new Object();
    static AtomicInteger got = new AtomicInteger();
    static AtomicInteger notGot = new AtomicInteger();

    public static void main(String[] args) throws InterruptedException {
        int threadCountTemp = threadCount;
        // 每个线程执行的次数
        int count = 20;
        for (int i = 0; i < threadCountTemp; i++) {
            Thread thread = new Thread(new ConnectionRunner(count), "Thread-" + i);
            thread.start();
        }
        synchronized (END_LOCK) {
            END_LOCK.wait();
        }
        System.out.println("-------------------------------------");
        System.out.println("主线程被唤醒！！！");
        System.out.println("total invoke: " + threadCount * count);
        System.out.println("got: " + got);
        System.out.println("notgot: " + notGot);
        System.out.println("-------------------------------------");
    }

    static class ConnectionRunner implements Runnable {

        int count;

        public ConnectionRunner(int count) {
            this.count = count;
        }

        @Override
        public void run() {
            synchronized (START_LOCK) {
                // 尽管threadCount不是volatile的，A线程在wait释放锁之后A线程对threadCount所做的改动B线程一定可见,happens-before
                // 需要通过threadCount记录等待线程的数量，到达数量时唤醒所有等待的线程，同时开始执行
                if (--threadCount <= 0) {
                    System.out.println(Thread.currentThread().getName() + " notify ALL!");
                    threadCount = THREAD_COUNT;
                    START_LOCK.notifyAll();
                } else {
                    try {
                        START_LOCK.wait();
                        System.out.println(Thread.currentThread().getName() + " 从wait中醒来！");
                    } catch (InterruptedException e) {
                    }
                }
            }
            while (count-- > 0) {
                try {
                    Connection connection = pool.fetchConnection(1000);
                    if (connection != null) {
                        try {
                            connection.createStatement();
                            connection.commit();
                        } catch (Exception e) {
                            e.printStackTrace();
                        } finally {
                            pool.releaseConnection(connection);
                            got.incrementAndGet();
                        }
                    } else {
                        notGot.incrementAndGet();
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            synchronized (END_LOCK) {
                // 需要通过threadCount记录通知线程的数量，到达指定数量时才进行通知
                if (--threadCount <= 0) {
                    System.out.println(Thread.currentThread().getName() + " notifyALL and finished!");
                    END_LOCK.notifyAll();
                } else {
                    System.out.println(Thread.currentThread().getName() + " finished!");
                }
            }
        }
    }
}
```
在wait-notify方式的基础上需要自己封装计数操作实现功能，没有CountDownLatch方式优雅。    

下面是通过CyclicBarrier方式实现对上述简易数据库连接池功能的测试：
```java
public class ConnectionPoolTestCyclicBarrier {

    private static ConnectionPool pool = new ConnectionPool(10);
    /**
     * 线程总数量
     */
    private static final int THREAD_COUNT = 50;
    /**
     * 每个线程执行的次数
     */
    private static final int EXECUTE_COUNT = 20;
    static AtomicInteger got = new AtomicInteger();
    static AtomicInteger notGot = new AtomicInteger();
    static CyclicBarrier startBarrier = new CyclicBarrier(THREAD_COUNT);
    static CyclicBarrier endBarrier = new CyclicBarrier(THREAD_COUNT, new CountRunner());

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            Thread thread = new Thread(new ConnectionRunner(EXECUTE_COUNT), "Thread-" + i);
            thread.start();
        }
        System.out.println("-------------------------------------");
        System.out.println("main finished!");
        System.out.println("-------------------------------------");
    }

    static class CountRunner implements Runnable {
        @Override
        public void run() {
            System.out.println("-------------------------------------");
            System.out.println("所有工作线程执行完毕！！！");
            System.out.println("total invoke: " + THREAD_COUNT * EXECUTE_COUNT);
            System.out.println("got: " + got);
            System.out.println("notgot: " + notGot);
            System.out.println("-------------------------------------");
        }
    }

    static class ConnectionRunner implements Runnable {

        int count;

        public ConnectionRunner(int count) {
            this.count = count;
        }

        @Override
        public void run() {
            try {
                startBarrier.await();
                while (count-- > 0) {
                    try {
                        Connection connection = pool.fetchConnection(1000);
                        if (connection != null) {
                            try {
                                connection.createStatement();
                                connection.commit();
                            } catch (Exception e) {
                                e.printStackTrace();
                            } finally {
                                pool.releaseConnection(connection);
                                got.incrementAndGet();
                            }
                        } else {
                            notGot.incrementAndGet();
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println(Thread.currentThread().getName() + " finished!");
                // 这里会造成工作线程执行结束后需要阻塞等到所有工作线程执行完毕，相比于CountDownLatch方式多了一些无用的阻塞！！！
                // 不如CountDownLatch优雅！！！
                endBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```
使用这种方式相比于CountDownLatch方式多了一些无用的阻塞，会使得即使先完成工作的线程也要阻塞到所有线程完成工作之后才能结束，不如CountDownLatch实现方式优雅。  

所以我们需要结合具体需求场景来使用这三种并发流程控制工具，尽量使用最合适的方式。  

_ _ _
### **从源码层面对比分析**
CountDownLatch内部是基于AQS框架来实现的，其源码分析如下：  
```java
public class CountDownLatch {

    // 封装AQS相关操作
    private static final class Sync extends AbstractQueuedSynchronizer {

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }
        
        // 如果state不为0，则尝试获取锁失败，获取锁的线程陷入阻塞
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        // 调用countdown时会尝试释放锁
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public void countDown() {
        sync.releaseShared(1);
    }
}
```
下面是基于lock.condition()模拟实现的一个可重复使用的CountDownLatch:
```java
public class ConditionCountDownDemo implements Runnable {
    //用Condition实现CountDownLatch功能
    static ReentrantLock reentrantLock = new ReentrantLock();
    static Condition condition = reentrantLock.newCondition();
    static ConditionCountDown conditionCountDown = new ConditionCountDown(10);
    @Override
    public void run() {
        try{
            // 调用countDown的线程先进行某些耗时的操作
            Thread.sleep(1000);
            conditionCountDown.countDown();
            //Thread.sleep(new Random().nextInt(10)*1000);
            System.out.println("check over ,count = "+conditionCountDown.getCount());
        }catch(Exception e){
            e.printStackTrace();
        }
    }
    static class ConditionCountDown {
        //计数器
        int count = 0;
        int tempCount;
        public ConditionCountDown(int count) {
            this.count = count;
            tempCount = count;
        }
        public void countDown() throws InterruptedException {
            reentrantLock.lock();
            tempCount--;
            try{
                //线程做完，才signal被阻塞的线程
                if(tempCount == 0) {
                    condition.signalAll();
                    tempCount = count;
                }
            }finally{
                reentrantLock.unlock();
            }
        }
        public int getCount() {
            reentrantLock.lock();
            try {
                return tempCount;
            } finally {
                reentrantLock.unlock();
            }
        }
        public void await() throws InterruptedException {
            reentrantLock.lock();
            try{
                //处于等待唤醒状态
                condition.await();
            }finally{
                reentrantLock.unlock();
            }
        }
    }
    public static void main(String args[]) throws InterruptedException {
        ConditionCountDownDemo con = new ConditionCountDownDemo();
        ExecutorService exec = Executors.newFixedThreadPool(15);
        for(int i = 0;i<15;i++) {exec.submit(con);}
        conditionCountDown.await();
        System.out.println("notify： " + conditionCountDown.getCount());
        // conditionCountDown可重复使用
        conditionCountDown.await();
        exec.shutdown();
    }
}
```  
java目前的实现当中的countdownLatch采用的是以获取共享锁的方式实现，上面自写的以condition方式的实现是以获取互斥锁的方式实现，共享锁的方式没有从等待队列与同步队列当中的移动的开销。  

为什么countdownLatch底层不使用condition实现呢?  性能原因吗？？？  

CyclicBarrier内部是基于AQS框架中ReentrantLock.newComdition()来实现的，其源码分析如下：  
```java
public class CyclicBarrier {

    /** The lock for guarding barrier entry */
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
    /** The number of parties */
    // 定义CyclicBarrier时指定的初始值
    private final int parties;
    /* The command to run when tripped */
    private final Runnable barrierCommand;
    /** The current generation */
    private Generation generation = new Generation();

    // 从给定的初始值开始倒计时直至为0，为0时打开栅栏
    private int count;

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        // 指定栅栏打开时要进行的操作
        this.barrierCommand = barrierAction;
    }

    private void nextGeneration() {
        // signal completion of last generation
        // 唤醒所有等待的线程并准备进行下一次拦截
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
    }

    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            // 判断是否可以打开栅栏
            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    // 执行初始化栅栏时指定的开栅栏动作
                    if (command != null)
                        command.run();
                    ranAction = true;
                    // 唤醒所有等待的线程并准备进行下一次拦截
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await(); // condition阻塞等待
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();
                // 从await中醒来之后判断是否需要继续睡眠  
                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    } 
    
    ...
}
```  

_ _ _
### **Semaphore简单介绍**
信号量用于对有限数量的资源的同时并发访问数进行控制。若有m个资源，但有n条线程（n>m），因此同一时刻只能允许m条线程访问资源，此时可以使用Semaphore控制访问该资源的线程数量。Semaphone 可以将任何一种容器变为有界阻塞容器，如用于实现资源池。例如数据库连接池。我们可以构造一个固定长度的连接池，使用阻塞方法 acquire和release获取释放连接，而不是获取不到便失败。（当然，一开始设计时就使用BlockingQueue来保存连接池的资源是一种更简单的方法)。 Semaphone底层实现也是基于AQS框架的。  

下面是使用Semaphore实现数据库连接池的一个简单示例，相比于wait-notify方式不能使用等待超时机制:  
```java
public class ConnectionPoolWithSemaphore {

    private List<Connection> pool =  Collections.synchronizedList(new LinkedList<>());
    private Semaphore semaphore;

    public ConnectionPoolWithSemaphore(int initialSize) {
        if (initialSize > 0) {
            for (int i = 0; i < initialSize; i++) {
                pool.add(ConnectionDriver.createConnection());
            }
            semaphore = new Semaphore(initialSize);
        }
    }

    public void releaseConnection(Connection connection) {
        if (connection != null) {
            pool.add(connection);
            semaphore.release();
        }
    }

    public Connection fetchConnection() throws InterruptedException {
        semaphore.acquire();
        return pool.remove(0);
    }
}
```
关于Semaphore相关就记录到这里，源码不再进行分析。  

(完)
参考文章：书籍：《Java并发编程的艺术》

