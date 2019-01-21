---
layout:     post
title:      Java定时任务调度工具之Timer实现原理分析
date:       2019-1-21
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - JDK源码阅读
---
>定时任务是最近在做的项目中一个比较重要的功能，所以对Java定时任务调度的实现方案做下系统性的学习，就从最基础的组件Timer开始。  

_ _ _
### **Timer常见功能概述**
<img src="/img/2019-1-21/TimerUML.png" width="700" height="700" alt="TimerUML" />
<center>TimerUML图示</center>  
如上图所示，使用Timer实现定时任务调度，重要的组件有两个：Timer与TimerTask。在使用时只需自定义我们需要创建的定时任务TimerTask交给Timer来调度即可。**一个TimerTask只能交由一个Timer来调度；一个Timer可以调度多个TimerTask。**  

Demo：提交时执行第一次执行MyTimerTask，之后每2s执行一次MyTimerTask。    
```java  
public class MyTimer {
    public static void main(String[] args) {
        Timer timer = new Timer();
        SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        MyTimerTask myTimerTask = new MyTimerTask("schedule 1");
        Calendar calendar = Calendar.getInstance();
        // 提交时执行第一次，之后每2s执行一次
        timer.schedule(myTimerTask, calendar.getTime(), 2000L);
    }
}

public class MyTimerTask extends TimerTask {

    private String name;

    public MyTimerTask(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        System.out.println("currrent exec name is : " + name + " threadId: " + Thread.currentThread().getId());
    }
}
```
可以看到上面的代码中使用了timer.schedule()方法关联了Timer和TimerTask，实现了对自定义任务的调度。使用Timer对TimerTask调度主要有两种方式，一是使用timer.schedule()方法，这种方法被称为fixed-delay方式调度；二是使用Timer.scheduleAtFixedRate()方法进行调度，这种方式被称为fixed-rate方式。经过我的实验得知，这两种方式的主要区别有两种情况：  

一、首次计划执行时间早于当前时间：

<img src="/img/2019-1-21/diff1.png" width="600" height="600" alt="TimerUML" />

<img src="/img/2019-1-21/diff2.png" width="600" height="600" alt="TimerUML" />

二、任务执行时间超出执行周期间隔：   

1、schedule方法：    

例1：任务预计每隔3s执行一次，每次任务执行实际花费的时间是1s；scheduledExecutionTime开始为12:00:00,预计下次的scheduledExecutionTime为12:00:03(在开始执行时间基础上向后加3s)，任务执行过后时间为12:00:01，并没有超过预计的下次计划时间，所以等待下次执行。    
例2：任务预计每隔3s执行一次，每次任务执行实际花费的时间是5s；scheduledExecutionTime开始为12:00:00,预计下次的scheduledExecutionTime为12:00:03，任务执行过后时间为12:00:05，当前时间为12:00:05，已经超过了预计的下次计划时间，所以将下次计划执行时间设置为12:00:08(本次执行的开始时间12:00:05 + 3s，仅与本次执行开始时间有关，与上次计划时间无关)并开始执行。  

2、scheduleAtFixedRate方法：  

例1：任务预计每隔3s执行一次，每次任务执行实际花费的时间是1s；scheduledExecutionTime开始为12:00:00, 任务执行之前更新下次的scheduledExecutionTime为12:00:03，任务第一次执行过后时间为12:00:01，并没有超过预计的下次计划时间，所以正常等待下次执行。
例2：任务预计每隔3s执行一次，每次任务执行实际花费的时间是5s；scheduledExecutionTime开始为12:00:00, 任务执行之前更新下次的scheduledExecutionTime为12:00:03，任务第一次执行过后时间为12:00:05，已经超过预计的下次计划时间，所以不再等待，先更新下次执行时间为12:00:06(上次的计划时间12:00:03 + 3，与当前时间无关),并开始执行任务。  

**需要注意的是，不论在单线程还是多线程的情况下，被schedule方法调度的Task都仅会被单线程执行(Timer默认情况下内部使用的是单个后台线程)；但在使用多个后台线程调度任务的情况下，如果出现了上面介绍的两种情况，被scheduleAtFixedRate方法调度的Task是否可能被多线程并发执行呢？**  

**Schedule单后台线程调度实例：**  
1、 每隔2s执行一次，函数执行时间1s；首次计划执行时间：12.19.45首次实际执行时间：12.19.45  
<img src="/img/2019-1-21/diff3.png" width="600" height="600" alt="TimerUML" />
2、每隔2s执行一次，函数执行时间4s；首次计划执行时间：12.17.47首次实际执行时间：12.17.47
<img src="/img/2019-1-21/diff4.png" width="600" height="600" alt="TimerUML" />
3、每隔2s执行一次，函数第一次执行时间为4s，之后执行时间为1s：首次计划执行时间：12.25.45首次实际执行时间：12.25.45
<img src="/img/2019-1-21/diff5.png" width="600" height="600" alt="TimerUML" />

**scheduleAtFixedRate单后台线程调度实例：**  
1、每隔2s执行一次，函数执行时间1s；首次计划执行时间：12.28.18首次实际执行时间：12.28.18  
<img src="/img/2019-1-21/diff6.png" width="600" height="600" alt="TimerUML" />
2、每隔2s执行一次，函数执行时间4s，首次计划执行时间：12.30.57首次实际执行时间：12.30.57
<img src="/img/2019-1-21/diff7.png" width="600" height="600" alt="TimerUML" />
3、每隔2s执行一次，函数第一次执行时间为4s，之后执行时间为1s，首次计划执行时间：12.33.25首次实际执行时间：12.33.25
<img src="/img/2019-1-21/diff8.png" width="600" height="600" alt="TimerUML" />

下面就分别分析下schedule与scheduleAtFixedRate的实现原理。  

### **schedule与scheduleAtFixedRate实现原理分析**
在分析schedule实现原理之前，先来分析下TimerTask这个类，我们提交的任务就是使用这个类来描述的。  
```java
public abstract class TimerTask implements Runnable {
    /**
     * 用来控制对TimerTask内部变量的并发访问
     */
    final Object lock = new Object();

    /**
     * 代表当前Timer内部的状态，在下面几个常量当中取值
     */
    int state = VIRGIN;

    /**    
     * 代表这个任务在创建过后还没有交给任何一个Timer来调度
     */
    static final int VIRGIN = 0;

    /**
     * 代表这个任务下次要被执行的时间已经被Timer调度器计划好了。如果当前任务是一个不被重复执行的任务，
     * 那么在当前状态下，这个任务就还没有被执行过。  
     */
    static final int SCHEDULED   = 1;

    /**
     * 这个不被重复执行的任务已经被执行过了(或者正在被执行)，并且没有被cancel过。
     * 只有非重复执行的任务才会达到这个状态
     */
    static final int EXECUTED    = 2;

    /**
     * 这个任务已经被TimerTask.cancel方法调用取消了
     */
    static final int CANCELLED   = 3;

    /**
     * 这个任务下次被计划的执行时间，对于需要重复执行的任务来讲，这个成员变量在每次任务被实际执行之前更新
     */
    long nextExecutionTime;

    /**
     * Period in milliseconds for repeating tasks.  A positive value indicates
     * fixed-rate execution.  A negative value indicates fixed-delay execution.
     * A value of 0 indicates a non-repeating task.
     * 0代表当前任务是一个不会被重复执行的任务
     * 小于0代表当前任务是fixed-delay方式重复执行的任务(就是使用Timer.schedule()方法进行调度的任务)
     * 大于0代表当前任务是fixed-rate方式重复执行的任务(就是使用Timer.scheduleAtFixedRate()方法进行调度的任务)
     */
    long period = 0;

    /**
     * Creates a new timer task.
     */
    protected TimerTask() {
    }

    /**
     * 我们继承这个TimerTask抽象类，在run方法中自定义我们需要被定时调度的任务
     */
    public abstract void run();

    /** 
     * 取消这个任务。如果这个任务已经被计划好下次要执行的时间但是还没有被执行，或者当前任务还没有被交给Timer来调度，
     * 换句话说就是当前任务的状态为1，cancel方法调用之后，这个任务就再也不会被运行了。如果当前任务是一个会被
     * 重复执行的任务，这个任务不会再继续重复下去。但是如果这个重复类型的任务在这个方法被调用的时候正在被运行，那么这次
     * 运行会正常结束，不会受到cancel的影响。
     *   
     * 在重复执行类型的任务的run方法中调用这个方法可以完全确保这个TimerTask不会再被继续执行，是推荐的方式。
     *
     * cancel方法可以被重复调用，但只有第一次会起作用。第一次成功调用cancel方法其返回值为true，之后调用的返回值均为false。    
     *
     */
    public boolean cancel() {
        synchronized(lock) {
            boolean result = (state == SCHEDULED);
            state = CANCELLED;
            return result;
        }
    }

    /**
     * 在TimeTask.run方法中获取到的scheduledExecutionTime返回值是本次任务run方法的计划时间
     * 在别处调用scheduledExecutionTime，如果任务正在运行，则返回值是正在运行的任务的计划时间；
     * 否则返回值是上次任务的计划运行时间；
     * 
     * 推荐使用的方式：在run方法执行中先判断当前时间与本次任务计划执行的时间之间的差值，如果差值太大，超过了我们设置的阈值，
     * 那么意味着我们这次方法执行比预计的时间要晚很多，超过了我们能接受的范围，那么就直接跳过本次执行。      
     *   public void run() {
     *       if (System.currentTimeMillis() - scheduledExecutionTime() >=
     *           MAX_TARDINESS)
     *               return;  // Too late; skip this execution.
     *       // Perform the task
     *   }    
     * 
     * 这个方法通常不与fixed-delay方式(即使用schedule方法进行调度的任务)执行的重复任务结合使用，
     * 因为它们的计划执行时间允许随上次任务执行的结束时间向后推移，因此不是非常重要。
     *
     */
    public long scheduledExecutionTime() {
        synchronized(lock) {
            return (period < 0 ? nextExecutionTime + period
                               : nextExecutionTime - period);
        }
    }
}
```
下面来看schedule与scheduleAtFixedRate方法的实现：  
```java
public class Timer {
    
    // 提交的任务都会放入这个任务队列中
    private final TaskQueue queue = new TaskQueue();
    // 用来执行TimeTask的TimerThread线程，内部持有任务队列的引用
    private final TimerThread thread = new TimerThread(queue);

    public void schedule(TimerTask task, Date firstTime, long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        // 实际调用sched方法
        sched(task, firstTime.getTime(), -period);
    }

    public void scheduleAtFixedRate(TimerTask task, Date firstTime,
                                    long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        // 实际调用sched方法
        sched(task, firstTime.getTime(), period);
    }

    // 注意上面schedule给sched方法传入的period是一个负值，scheduleAtFixedRate传入的是正值
    private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");

        // Constrain value of period sufficiently to prevent numeric
        // overflow while still being effectively infinitely large.
        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;

        synchronized(queue) {
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");

            synchronized(task.lock) {
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                task.nextExecutionTime = time;
                task.period = period;
                task.state = TimerTask.SCHEDULED;
            }
            // 将任务添加到队列中
            queue.add(task);
            if (queue.getMin() == task)
                queue.notify();
        }
    }
}

```
可见schedule与scheduleAtFixedRate所做的实质性工作就是把当前任务添加到了任务队列中，并且schedule传入的period的值在用户传入的值的基础上乘了-1，scheduleAtFixedRate保持了用户传入的period的值，通过task中period值的正负可以区分当前任务通过哪种方式调度。  

下面先来看保存提交的任务的queue都有哪些功能，再来分析TimerThread是如何从queue中取出任务执行的。  
```java
/**
 * 这个类是Timer中的内部类，使用堆实现了一个优先队列，基于task.nextExecutionTime对队列中的任务进行排序
 * 下面只看优先队列提供的功能，对于其数据结构的内部实现不仔细分析
 */
class TaskQueue { 

    /**
     * 返回当前队列中的任务数量
     */
    int size() {
    }

    /**
     * 向队列中添加一个新的任务
     */
    void add(TimerTask task) {
    }

    /**
     * 返回nextExecutionTime值最小的任务
     */
    TimerTask getMin() {
    }

    /**
     * 返回指定第i个任务，按照nextExecutionTime值进行排序
     */
    TimerTask get(int i) {
    }

    /**
     * 移除第一个任务
     */
    void removeMin() {        
    }

    /**
     * 移除指定的第i个任务
     */
    void quickRemove(int i) {      
    }

    /**
     * 重新设置队列头部任务的nextExecutionTime，并且调整优先队列
     */
    void rescheduleMin(long newTime) {       
    }

    /**
     * 判空
     */
    boolean isEmpty() {
    }

    /**
     * 移除队列中所有元素
     */
    void clear() {
    }       
}
```
接下来看实际执行任务的后台线程TimerThread：  
```java
public class Timer {

    // 用来实现TimerThread的优雅退出，当外界没有指向Timer的引用，Timer中的threadReaper被GC回收时会
    // 设置thread.newTasksMayBeScheduled = false，之后timerThread在执行完队列中的所有任务之后就会自动退出
    private final Object threadReaper = new Object() {
        protected void finalize() throws Throwable {
            synchronized(queue) {
                thread.newTasksMayBeScheduled = false;
                queue.notify(); // In case queue is empty.
            }
        }
    };
}
/**
 * 是Timer的内部类，提供的几个主要功能：  
 *     1、在等待队列中取出符合条件的任务并执行
 *     2、对于重复执行的任务，计划他们的下次执行时间
 *     3、从队列中移除被cancel的任务和已经被执行过的不需要被重复执行的任务
 */
class TimerThread extends Thread {
    /**
     * 用来实现TimerThread的退出机制，TimerThread在newTasksMayBeScheduled为false并且任务队列为空时会自动退出。  
     * 
     * newTasksMayBeScheduled可能在下面几种情况中被设置为false
     *     1、Timer.cancel方法被调用
     *     2、Timer.threadReaper被GC回收，其finalized方法被调用时
     */
    boolean newTasksMayBeScheduled = true;

    /**
     * 与对应的Timer中的queue指向的是同一个任务队列
     */
    private TaskQueue queue;

    TimerThread(TaskQueue queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            mainLoop();
        } finally {
            // Someone killed this Thread, behave as if Timer cancelled
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();  // Eliminate obsolete references
            }
        }
    }

    /**
     * The main timer loop.  (See class comment.)
     */
    private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired;
                synchronized(queue) {
                    // Wait for queue to become non-empty
                    while (queue.isEmpty() && newTasksMayBeScheduled)
                        queue.wait();
                    if (queue.isEmpty())
                        break; // Queue is empty and will forever remain; die

                    // Queue nonempty; look at first evt and do the right thing
                    long currentTime, executionTime;
                    // 取出要执行的任务
                    task = queue.getMin();
                    synchronized(task.lock) {
                        if (task.state == TimerTask.CANCELLED) {
                            // 如果取出的任务状态为CANCELLED，则直接将其删除
                            queue.removeMin(); 
                            continue;  // No action required, poll queue again
                        }
                        currentTime = System.currentTimeMillis();
                        executionTime = task.nextExecutionTime;
                        // 判断当前任务是否满足执行条件
                        if (taskFired = (executionTime<=currentTime)) {
                            // 当前任务是一次性执行的任务
                            if (task.period == 0) { // Non-repeating, remove
                                queue.removeMin();
                                task.state = TimerTask.EXECUTED; // 状态设置为EXECUTED
                            } else { // Repeating task, reschedule
                                /**
                                 * 执行任务之前重新设置要被执行的任务的nextExecutionTime
                                 * 就是在这里对schedule与scheduleAtFixedRate两种方式做了区分
                                 * schedule是在当前任务执行时间的基础上加上period作为nextExecutionTime
                                 * 而scheduleAtFixedRate是在上次预计执行时间的基础上加上period作为nextExecutionTime
                                 */                              
                                queue.rescheduleMin(
                                  task.period<0 ? currentTime   - task.period
                                                : executionTime + task.period);
                            }
                        }
                    }
                    if (!taskFired) // Task hasn't yet fired; wait
                        queue.wait(executionTime - currentTime);
                }
                if (taskFired)  // Task fired; run it, holding no locks
                    task.run();
            } catch(InterruptedException e) {
            }
        }
    }
}
```
可以看到TimerThread基于wait-notify机制从队列中不断取出任务并运行。  

通过上面的分析过程，深入的理解了Timer任务调度的实现原理，但也可以看出使用Timer的一些缺陷：  
1. 并发任务数量多的话Timer管理不过来；JDK文档中描述的是在管理数千个任务的时候应该没什么问题，但我觉得这是在不存在较为耗时的定时任务的基础上的假设。  
2. 当有任何一个任务抛出了运行时异常导致TimerThread退出，就会终止所有定时任务的执行(因为Timer只有这一个后台服务线程)，这显然不是我们想要的结果。  

JDK官方文档中在Timer类的介绍中有提到java5之后juc包中的ScheduledThreadPoolExecutor相对于Timer是一个更好的实现定时任务调度的工具，它可以配置使用多个线程对任务进行调度(设置为1个后台线程的时候其实与Timer就大致相同了)，还可以接受更多的时间参数(Timer中只有ms)，并且不需要我们继承TimerTask实现想要的功能，只需要实现Runnable接口即可，下篇文章就对ScheduledThreadPoolExecutor实现原理进行分析。  