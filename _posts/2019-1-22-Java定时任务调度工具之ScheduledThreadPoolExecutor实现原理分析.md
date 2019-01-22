---
layout:     post
title:      Java定时任务调度工具之ScheduledThreadPoolExecutor实现原理分析
date:       2019-1-22
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - JDK源码阅读
    - 并发
---
>定时任务是最近在做的项目中一个比较重要的功能，所以对Java定时任务调度的实现方案做下系统性的学习，接续上篇文章中对于Timer的分析-[Java定时任务调度工具之Timer实现原理分析](https://wang-michael.github.io/2019/01/21/Java%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%E5%B7%A5%E5%85%B7%E4%B9%8BTimer%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)，本篇分析ScheduledThreadPoolExecutor。   

_ _ _
### **ScheduledThreadPoolExecutor的使用方法**
<img src="/img/2019-1-22/ScheduledThreadPoolExecutorUML.png" width="700" height="700" alt="ScheduledThreadPoolExecutorUML" />
<center>ScheduledThreadPoolExecutorUML图示</center>  

可见ScheduledThreadPoolExecutor具有的功能主要在ExecutorService和ScheduledExecutorService两个接口中规定，ExecutorService规定了其作为一个线程池应有的功能，ScheduledExecutorService规定了其作为一个定时调度任务工具应该有的功能。  

下面是JDK官方文档中给出的ScheduledExecutorService使用Demo：  
```java
public class BeeperControl {

    private final ScheduledExecutorService scheduler =
            Executors.newScheduledThreadPool(1);

    public void beepForAnHour() {
        final Runnable beeper = new Runnable() {
            public void run() { System.out.println("beep"); }
        };
        final ScheduledFuture<?> beeperHandle =
                scheduler.scheduleAtFixedRate(beeper, 5, 5, SECONDS);
        scheduler.schedule(new Runnable() {
            public void run() { beeperHandle.cancel(true); }
        }, 60 * 60, SECONDS);
    }

    public static void main(String[] args) {
        new BeeperControl().beepForAnHour();
    }
}
```
Demo的作用是每隔5s输出一次beep，并在1小时之后结束任务。    

ScheduledExecutorService接口中规定了任务的三种调度方式：schedule()、	scheduleAtFixedRate()、scheduleWithFixedDelay(),下面就通过分析源码来看看这三种方式的异同。  

### **ScheduledThreadPoolExecutor的实现原理分析**
**线程池的本质就是使用了一个线程安全的工作队列连接工作者线程和客户端线程，客户端线程将任务放入工作队列后便返回，而工作者线程则不断的从工作队列中取出工作并执行。当工作队列为空时，所有的工作线程均等待在工作队列上，当有客户端提交了一个任务之后会通知任意一个工作者线程，随着大量的任务被提交，更多的工作者线程会被唤醒。** 如下图所示：     
<img src="/img/2019-1-22/ThreadPoolExecutoryuanli.png" width="700" height="700" alt="ScheduledThreadPoolExecutorUML" />
<center>线程池原理图</center>    

ScheduledThreadPoolExecutor也是ThreadPoolExecutor，所以也是上述架构的，只不过其内部使用的阻塞队列(工作队列)是DelayQueue，我们提交的任务在其中会按照我们指定的delayTime进行排序，delayTime小的排在队列前面先被调度执行，从而实现定时任务调度的功能。  

### **源码分析**
构造函数：    
```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {

    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }

    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), handler);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
    }
}
```
可见都使用DelayQueue作为工作队列的实现。  

schedule、scheduleAtFixedRate、scheduleWithFixedDelay实现原理分析：  
```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {

    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        // 把提交的任务封装成ScheduledFutureTask
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        // 将任务添加到工作队列中等待执行
        delayedExecute(t);
        return t;
    }

    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        // 把提交的任务封装成ScheduledFutureTask
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        // 将任务添加到工作队列中等待执行
        delayedExecute(t);
        return t;
    }

    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft); 
        // outerTask保存当前任务以便在一次执行过后重新入队，达到重复执行的功能
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        // outerTask保存当前任务以便在一次执行过后重新入队，达到重复执行的功能
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }

    protected <V> RunnableScheduledFuture<V> decorateTask(
        Runnable runnable, RunnableScheduledFuture<V> task) {
        return task;
    }

    protected <V> RunnableScheduledFuture<V> decorateTask(
        Callable<V> callable, RunnableScheduledFuture<V> task) {
        return task;
    }

    // 当前时间基础上加上delay
    long triggerTime(long delay) {
        return now() +
            ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
    }

    private void delayedExecute(RunnableScheduledFuture<?> task) {
        if (isShutdown())
            reject(task);
        else {
            super.getQueue().add(task);
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }

    // 因为ScheduledFutureTask实现了Runnable接口，所以可以作为工作任务被加入到线程池的工作队列中
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {

        /** The actual task to be re-enqueued by reExecutePeriodic */
        RunnableScheduledFuture<V> outerTask = this;
        
        ScheduledFutureTask(Runnable r, V result, long ns) {
            super(r, result);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        ScheduledFutureTask(Callable<V> callable, long ns) {
            super(callable);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

        ScheduledFutureTask(Runnable r, V result, long ns, long period) {
            super(r, result);
            this.time = ns;
            this.period = period;
            this.sequenceNumber = sequencer.getAndIncrement();
        }
        
        // 以time的值与当前时间的差值作为在工作队列中排序的依据，差值越小排的越靠前
        public long getDelay(TimeUnit unit) {
            return unit.convert(time - now(), NANOSECONDS);
        }
    }
}
```
上面的代码执行之后，我们提交的任务就已经被加入到了工作队列，并且可以看到scheduleAtFixedRate传入的ScheduledFutureTask的period是正值，schedule传入的ScheduledFutureTask的period是0，scheduleWithFixedDelay传入的period是负值。正是通过这种方式对于调度的三种类型的任务进行区分。  

下面就来分析任务取出执行的过程。  
```java
public class ThreadPoolExecutor extends AbstractExecutorService {  

    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
     
        final Thread thread;
    
         /**
         * 创建Worker时会同时创建一个新线程.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            //把Worker传递给新建的线程，当线程执行是会调用Worker的run方法。
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            // 工作线程在runWorker方法中执行任务
            runWorker(this);
        }
    }
    
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 调用我们自定义的run方法，从ScheduledFutureTask.run方法调用
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
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
}

public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {

    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {
        
        /**
         * Overrides FutureTask version so as to reset/requeue if periodic.
         */
        public void run() {
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            // 非重复类型的任务，调用FutureTask.run()方法
            else if (!periodic)
                ScheduledFutureTask.super.run();
            // 重复类型的任务，调用FutureTask.runAndReset方法
            else if (ScheduledFutureTask.super.runAndReset()) {
                // 重新设置任务下次要执行的时间
                setNextRunTime();
                // 把任务重新加入到任务队列中去
                reExecutePeriodic(outerTask);
            }
        }
        
        // period不为0代表是重复执行类型的任务
        public boolean isPeriodic() {
            return period != 0;
        }

        /**
         * 重新设置任务下次要执行的时间
         */
        private void setNextRunTime() {
            long p = period;
            if (p > 0) // fixAtRate方式
                time += p;
            else // fixedWithDelay方式
                time = triggerTime(-p);
        }
    }

    void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        if (canRunInCurrentRunState(true)) {
            super.getQueue().add(task);
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }
    
}

public class FutureTask<V> implements RunnableFuture<V> {

    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    // 实际调用的是我们传入的ScheduledFutureTask.run方法或call方法
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    // 实际调用的是我们传入的ScheduledFutureTask.run方法
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        // 重置future为初始状态
        return ran && s == NEW;
    }
}
```  
经过上面的源码分析过程可以得知：  
1. 使用ScheduledThreadPoolExecutor.schedule方法调度的任务一定是仅执行一次的任务
2. 使用ScheduledThreadPoolExecutor.scheduleAtFixedRate方法与scheduleWithFixedDelay方法调度的任务是重复执行类型的任务

ScheduledThreadPoolExecutor.scheduleAtFixedRate方法与scheduleWithFixedDelay方法区别：主要体现在方法执行时间过长超过我们设置的period这种情况时。  

* scheduleWithFixedDelay：如0秒开始执行第一次任务，任务耗时5秒，任务间隔时间3秒，那么第二次任务计划执行的时间是在第8秒开始。
* scheduleAtFixedRate：如0秒开始执行第一次任务，任务耗时5秒，任务间隔时间3秒，那么第二次任务计划执行的时间是在第3秒开始，实际执行时间是在第5s开始。

还要注意的一点是，在上面ScheduledFutureTask.run()方法中我们可以看到，**工作线程先取出我们的任务来执行，执行完之后再更新任务下次的计划执行时间并加入到工作队列中，所以我们的定时任务总是被单线程执行的，不用担心并发问题。**下面这个Demo可以体现这一点：  
```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ScheduledThreadPoolExecutor scheduledExecutorService = new ScheduledThreadPoolExecutor(6);
    
    // 预热线程池，启动所有核心线程
    scheduledExecutorService.prestartCoreThread();       

    ScheduledFuture scheduledFuture = scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        int count = 0;
        @Override
        public void run() {
            Calendar calendar = Calendar.getInstance();
            SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            System.out.println("time: " + sf.format(calendar.getTime()) + " id: " + Thread.currentThread().getId());
            try {
                if (count++ <= 4)
                    Thread.sleep(3000);
                else
                    Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }, 1000, 2000, TimeUnit.MILLISECONDS);  

}

```
输出结果：  
<img src="/img/2019-1-22/result.png" width="700" height="700" alt="ScheduledThreadPoolExecutorUML" />  

可以看到上述虽然线程池中有6个线程，但是上面每个过程中都是任务都是被单线程执行的。   

关于ScheduledThreadPoolExecutor的源码分析就到这里，可以看到它确实强化了Timer的很多功能，但是ScheduledThreadPoolExecutor提供的功能还不够丰富，举例来讲，如果想实现在每周六12点执行一个定时任务，需要我们在ScheduledThreadPoolExecutor的基础上编程实现。Spring框架为我们提供了更丰富的定时任务的功能，使用起来也很方便，只需要做一些配置就好了，下篇文章就来分析Spring框架中定时任务的实现原理。  