---
layout:     post
title:      Java定时任务调度工具之Spring定时任务实现原理分析
date:       2019-1-23
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Spring   
---
>定时任务是最近在做的项目中一个比较重要的功能，所以对Java定时任务调度的实现方案做下系统性的学习，接续上篇文章中对于ScheduledThreadPoolExecutor实现原理的分析-[Java定时任务调度工具之ScheduledThreadPoolExecutor实现原理分析](https://wang-michael.github.io/2019/01/22/Java%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%E5%B7%A5%E5%85%B7%E4%B9%8BScheduledThreadPoolExecutor%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)，本篇分析Spring框架中定时任务的实现原理。   

_ _ _
### **Spring框架中调度定时任务的两种方式**
方式一：被调度的定时任务在程序启动时就已固定，之后不会再变化，可以使用Spring中的@EnableScheduling注解配合@Scheduled来实现调度，优点是使用起来十分方便，缺点是不能对定时任务做动态的添加修改。Demo如下：  
```java
@SpringBootApplication
@EnableScheduling
public class WxOpenApplication {
    public static void main(String[] args) {
        SpringApplication.run(WxOpenApplication.class, args);
    }    
}

@Component
public class ScheduledTasks {

    private static final SimpleDateFormat dateFormat = new SimpleDateFormat("HH:mm:ss");

    // 以每5s一次的固定频率进行调度
    @Scheduled(fixedRate = 5000)
    public void reportCurrentTime() {
        System.out.println("现在时间：" + dateFormat.format(new Date()));
    }

}
```
方式二：Spring官方文档中有介绍到spring定时任务功能的实现主要基于两个接口：1、TaskExecutor 2、TaskScheduler，其中TaskScheduler接口的实现类只有两个，ThreadPoolTaskScheduler与ConcurrentTaskScheduler。下面就是一个使用ThreadPoolTaskScheduler方式可动态添加修改定时任务的Demo：  
```java
@RestController
@Component
public class DynamicTask {   
    @Autowired
    private ThreadPoolTaskScheduler threadPoolTaskScheduler;
    private ScheduledFuture<?> future;
    private static final SimpleDateFormat dateFormat = new SimpleDateFormat("HH:mm:ss");

    @Bean
    public ThreadPoolTaskScheduler threadPoolTaskScheduler() {
        return new ThreadPoolTaskScheduler();
    }

    @RequestMapping("/startCron")
    public String startCron() {
        // 任务重复执行，下次计划时间的更新策略在ReschedulingRunnable.run方法中实现
        future = threadPoolTaskScheduler.schedule(new MyRunnable(), new CronTrigger("0/10 * * * * ?"));
        System.out.println("DynamicTask.startCron()");
        return "startCron";
    }

    @RequestMapping("/stopCron")
    public String stopCron() {
        if (future != null) {
            future.cancel(true);
        }
        System.out.println("DynamicTask.stopCron()");
        return "stopCron";
    }

    @RequestMapping("/changeCron10")
    public String startCron10() {
        stopCron();// 先停止，在开启.
        future = threadPoolTaskScheduler.schedule(new MyRunnable(), new CronTrigger("*/10 * * * * *"));
        System.out.println("DynamicTask.startCron10()");
        return "changeCron10";

    }

    private class MyRunnable implements Runnable {
        @Override
        public void run() {
            System.out.println("DynamicTask.MyRunnable.run()，" + dateFormat.format(new Date()));
        }
    }
}
```
下面就来分析上面两种调度方式的实现原理。  

问题：TaskScheduler中的三种调度方式的异同？schedule、scheduleAtFixedRate、scheduleWithFixedDelay

### **原理分析**
### **方式二原理分析**
先来分析上面第二种方式ThreadPoolTaskScheduler的实现原理。  
<img src="/img/2019-1-22/ThreadPoolTaskSchedulerUML.png" width="700" height="700" alt="ThreadPoolTaskSchedulerUML" />
<center>ThreadPoolTaskSchedulerUML图示</center>  
```java
public class ThreadPoolTaskScheduler extends ExecutorConfigurationSupport implements AsyncListenableTaskExecutor, SchedulingTaskExecutor, TaskScheduler {

    private volatile ScheduledExecutorService scheduledExecutor;

    // 上面Demo中传入的Trigger是CronTrigger实现
    public ScheduledFuture<?> schedule(Runnable task, Trigger trigger) {
        ScheduledExecutorService executor = this.getScheduledExecutor();

        try {
            ErrorHandler errorHandler = this.errorHandler != null ? this.errorHandler : TaskUtils.getDefaultErrorHandler(true);
            // 这句代码的作用是把我们提交的任务交给executor来进行调度
            return (new ReschedulingRunnable(task, trigger, executor, errorHandler)).schedule();
        } catch (RejectedExecutionException var5) {
            throw new TaskRejectedException("Executor [" + executor + "] did not accept task: " + task, var5);
        }
    }

    public ScheduledExecutorService getScheduledExecutor() throws IllegalStateException {
        Assert.state(this.scheduledExecutor != null, "ThreadPoolTaskScheduler not initialized");
        return this.scheduledExecutor;
    }
}

class ReschedulingRunnable extends DelegatingErrorHandlingRunnable implements ScheduledFuture<Object> {     
    
    public ReschedulingRunnable(Runnable delegate, Trigger trigger, ScheduledExecutorService executor, ErrorHandler errorHandler) {
        super(delegate, errorHandler);
        this.trigger = trigger;
        this.executor = executor;
    }

    public ScheduledFuture<?> schedule() {
        Object var1 = this.triggerContextMonitor;
        synchronized(this.triggerContextMonitor) {
            // 根据传入的trigger解析得到当前任务下次计划执行的时间，这里调用的是CronTrigger.nextExecutionTime方法
            this.scheduledExecutionTime = this.trigger.nextExecutionTime(this.triggerContext);
            if (this.scheduledExecutionTime == null) {
                return null;
            } else {
                long initialDelay = this.scheduledExecutionTime.getTime() - System.currentTimeMillis();
                // 实际基于ScheduledExecutorService对任务进行调度
                this.currentFuture = this.executor.schedule(this, initialDelay, TimeUnit.MILLISECONDS);
                return this;
            }
        }
    }

    public void run() {
        Date actualExecutionTime = new Date();
        // 调用我们实际任务的run方法
        super.run();
        Date completionTime = new Date();
        Object var3 = this.triggerContextMonitor;
        synchronized(this.triggerContextMonitor) {
            // 一次调度完成之后使用SimpleTriggerContext.update更新各个时间
            this.triggerContext.update(this.scheduledExecutionTime, actualExecutionTime, completionTime);
            if (!this.currentFuture.isCancelled()) {
                // 重复调度
                this.schedule();
            }

        }
    }    
}

public class SimpleTriggerContext implements TriggerContext {

    public void update(Date lastScheduledExecutionTime, Date lastActualExecutionTime, Date lastCompletionTime) {
        this.lastScheduledExecutionTime = lastScheduledExecutionTime;
        this.lastActualExecutionTime = lastActualExecutionTime;
        this.lastCompletionTime = lastCompletionTime;
    }
}

// 实现有CronTrigger和PeriodicTrigger两种，分别是对cron表达式的支持以及与ScheduledExecutorService类似的period支持
public interface Trigger {
    Date nextExecutionTime(TriggerContext var1);
}
```
**可见ThreadPoolTaskScheduler实际上就是ScheduledExecutorService的对象适配器，在ScheduledExecutorService的基础上增加了对于Cron表达式的支持，易用性更强。**    

ThreadPoolTaskScheduler中的ScheduledExecutorService是何时被注入的呢？  
```java
public class ThreadPoolTaskScheduler extends ExecutorConfigurationSupport implements AsyncListenableTaskExecutor, SchedulingTaskExecutor, TaskScheduler {

    // 这个方法是被其父类ExecutorConfigurationSupport调用的
    @UsesJava7
    protected ExecutorService initializeExecutor(ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {
        this.scheduledExecutor = this.createExecutor(this.poolSize, threadFactory, rejectedExecutionHandler);
        if (this.removeOnCancelPolicy) {
            if (setRemoveOnCancelPolicyAvailable && this.scheduledExecutor instanceof ScheduledThreadPoolExecutor) {
                ((ScheduledThreadPoolExecutor)this.scheduledExecutor).setRemoveOnCancelPolicy(true);
            } else {
                this.logger.info("Could not apply remove-on-cancel policy - not a Java 7+ ScheduledThreadPoolExecutor");
            }
        }

        return this.scheduledExecutor;
    }

    protected ScheduledExecutorService createExecutor(int poolSize, ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {
        // 实际使用的还是我们上篇文章中介绍的juc包中的ScheduledThreadPoolExecutor
        return new ScheduledThreadPoolExecutor(poolSize, threadFactory, rejectedExecutionHandler);
    }
}

public abstract class ExecutorConfigurationSupport extends CustomizableThreadFactory implements BeanNameAware, InitializingBean, DisposableBean {
    
    public void afterPropertiesSet() {
        this.initialize();
    }
  
    public void initialize() {
        if (this.logger.isInfoEnabled()) {
            this.logger.info("Initializing ExecutorService " + (this.beanName != null ? " '" + this.beanName + "'" : ""));
        }

        if (!this.threadNamePrefixSet && this.beanName != null) {
            this.setThreadNamePrefix(this.beanName + "-");
        }
        // 调用子类ThreadPoolTaskScheduler的initializeExecutor方法
        this.executor = this.initializeExecutor(this.threadFactory, this.rejectedExecutionHandler);
    }
}
```
可见注入到ThreadPoolTaskScheduler中的ScheduledExecutorService实现类是ScheduledThreadPoolExecutor，是在ExecutorConfigurationSupport类中的afterPropertiesSet方法中触发注入过程的。所以我们在Spring框架中使用ThreadPoolTaskScheduler来动态管理定时任务的时候，**ThreadPoolTaskScheduler这个Bean实例一定要交给Spring框架来管理：**  
```java
@Bean
public ThreadPoolTaskScheduler threadPoolTaskScheduler() {
    return new ThreadPoolTaskScheduler();
}
```
否则上面ExecutorConfigurationSupport.afterPropertiesSet方法根本不会被框架调用，ThreadPoolTaskScheduler中的ScheduledExecutorService不会被注入，所以ThreadPoolTaskScheduler也就不会起作用了。     

接着再来来分析上面Demo中第一种方式的实现原理。
   
### **方式一原理分析**
方式一中使用了@EnableScheduling结合@Scheduled注解实现对定时任务的调度。@Scheduled注解的作用可以理解为在BeanDefinition中添加了被作为定时任务的方法的标记，下面来看@EnableScheduling注解的作用：  
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({SchedulingConfiguration.class}) // 导入配置类
@Documented
public @interface EnableScheduling {
}

@Configuration
@Role(2)
public class SchedulingConfiguration {
    public SchedulingConfiguration() {
    }

    @Bean(
        name = {"org.springframework.context.annotation.internalScheduledAnnotationProcessor"}
    )
    @Role(2)
    public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
        return new ScheduledAnnotationBeanPostProcessor();
    }
}
```
可见@EnableScheduling注解的实际作用就是引入了一个ScheduledAnnotationBeanPostProcessor的实例bean,这是一个Bean后置处理器，会在spring框架创建bean之后将相应的定时任务调度起来。  
```java
public class ScheduledAnnotationBeanPostProcessor implements MergedBeanDefinitionPostProcessor, DestructionAwareBeanPostProcessor, Ordered, EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, ApplicationContextAware, SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {

    private final ScheduledTaskRegistrar registrar = new ScheduledTaskRegistrar();

    @Override
	public Object postProcessAfterInitialization(final Object bean, String beanName) {
			  //省略多个判断条件代码
			 for (Map.Entry<Method, Set<Scheduled>> entry : annotatedMethods.entrySet()) {
				Method method = entry.getKey();
				for (Scheduled scheduled : entry.getValue()) {
				   processScheduled(scheduled, method, bean);
				}
			 }
	   }
	   return bean;
	}

    //说明：ScheduledAnnotationBeanPostProcessor继承BeanPostProcessor。
	//获取scheduled类参数，之后根据参数类型、相应的延时时间、对应的时区放入不同的任务列表中
	protected void processScheduled(Scheduled scheduled, Method method, Object bean) {   
		 //获取corn类型
		String cron = scheduled.cron();
        if (StringUtils.hasText(cron)) {
            Assert.isTrue(initialDelay == -1L, "'initialDelay' not supported for cron triggers");
            processedSchedule = true;
            String zone = scheduled.zone();
            if (this.embeddedValueResolver != null) {
                cron = this.embeddedValueResolver.resolveStringValue(cron);
                zone = this.embeddedValueResolver.resolveStringValue(zone);
            }

            TimeZone timeZone;
            if (StringUtils.hasText(zone)) {
                timeZone = StringUtils.parseTimeZoneString(zone);
            } else {
                timeZone = TimeZone.getDefault();
            }
            // 放入任务列表，先不执行
            tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
        }
        ...
	}
}

public class ScheduledTaskRegistrar implements InitializingBean, DisposableBean {

    private TaskScheduler taskScheduler;

    public ScheduledTask scheduleCronTask(CronTask task) {
        ScheduledTask scheduledTask = (ScheduledTask)this.unresolvedTasks.remove(task);
        boolean newTask = false;
        if (scheduledTask == null) {
            scheduledTask = new ScheduledTask();
            newTask = true;
        }
        // 在上面processScheduled方法调用scheduleCronTask时，taskScheduler为null，所以会进入else
        if (this.taskScheduler != null) {
            scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
        } else {
            // 放入任务列表，先不执行
            this.addCronTask(task);
            this.unresolvedTasks.put(task, scheduledTask);
        }

        return newTask ? scheduledTask : null;
    }
}
```
通过上面的processScheduled方法，不同类型的定时任务被放入ScheduledTaskRegistrar中相应的任务列表中,但是由于到目前为止ScheduledTaskRegistrar.taskScheduler为null，所以任务只是先被放入了任务队列，没有执行，那么任务是何时开始执行的呢？  
```java
public class ScheduledAnnotationBeanPostProcessor implements MergedBeanDefinitionPostProcessor, DestructionAwareBeanPostProcessor, Ordered, EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, ApplicationContextAware, SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {

    // 这个方法是在postProcessAfterInitialization之后被调用的
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext() == this.applicationContext) {
            this.finishRegistration();
        }
    }

    private void finishRegistration() {                   
        ...
        // 调用ScheduledTaskRegistrar.afterPropertiesSet方法
        this.registrar.afterPropertiesSet();
    }
}

public class ScheduledTaskRegistrar implements InitializingBean, DisposableBean {

    private ScheduledExecutorService localExecutor;

    public void afterPropertiesSet() {
        this.scheduleTasks();
    }

     protected void scheduleTasks() {
        // 此时 taskScheduler == null 成立
        if (this.taskScheduler == null) {
            // 这里使用的localExecutor实质上是corePoolSize为1的ScheduledThreadPoolExecutor
            this.localExecutor = Executors.newSingleThreadScheduledExecutor();
            this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
        }        

        if (this.cronTasks != null) {
            var1 = this.cronTasks.iterator();
        
            while(var1.hasNext()) {
                // 从相应的任务队列中取出任务
                CronTask task = (CronTask)var1.next();
                // 交由taskScheduler来执行
                this.addScheduledTask(this.scheduleCronTask(task));
            }
        }        
        ...
    }

    public ScheduledTask scheduleCronTask(CronTask task) {
        ScheduledTask scheduledTask = (ScheduledTask)this.unresolvedTasks.remove(task);
        boolean newTask = false;
        if (scheduledTask == null) {
            scheduledTask = new ScheduledTask();
            newTask = true;
        }

        if (this.taskScheduler != null) {
            // 使用taskScheduler来调度任务，底层使用的还是ScheduledThreadPoolExecutor
            scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
        } else {
            this.addCronTask(task);
            this.unresolvedTasks.put(task, scheduledTask);
        }

        return newTask ? scheduledTask : null;
    }
}
```
经过上面分析可见，在默认情况下，结合使用@EnableScheduling与@Scheduled注解进行任务调度，实际进行调度操作的是corePoolSize为1的ScheduledThreadPoolExecutor。  

### **总结**
经过上面的分析过程可以看到，Spring定时任务调度是在ScheduledThreadPoolExecutor使用适配器模式进行封装，引入cron表达式功能，使得定时任务调度功能使用起来更加方便。  

但是到目前为止介绍的几种定时任务调度都仅仅适用于单机情况下，在集群应用当中或者分布式应用当中，如果我们想确保集群中的某个任务仅被执行一次(不能仅把任务添加到集群中的一台服务器的任务队列中，因为这样集群就没有意义了，如果这台服务器宕机了，这个任务就不会被执行)，还需要在Spring定时任务调度的基础之上进行封装，比如结合Redis的分布式锁功能来实现，也可以使用下篇文章中要介绍的Quartz框架结合Spring使用。  