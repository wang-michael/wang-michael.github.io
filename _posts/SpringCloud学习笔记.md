# SpringCloud学习笔记
单体系统变得庞大臃肿之后会产生难以维护的问题，将其拆分为微服务可以做到每个服务都运行在自己的进程内，在部署上有稳定的边界，每个服务的更新都不会影响到其它服务的运行。同时，由于是独立部署的，我们可以更准确的为每个服务评估性能容量，通过配合服务间的协作流程也可以更容易的发现系统的瓶颈位置，以及给出较为准确的系统级性能容量评估。**微服务还有一大好处在于服务组件化之后，整个架构中的各个组件就能针对其不同的业务特点选择不同的技术平台。**     

微服务理念带来的问题：   
* 运维上的挑战：如何有条不紊的将这些进程编排和组织起来，即如何进行更好的服务治理？？？    
* 分布式的复杂性：由于拆分后的各个微服务都是独立部署并运行在各自的进程内，他们只能通过通信来进行协作，所有分布式环境的问题都将是微服务架构系统设计时需要考虑的重要因素，比如网络延迟、分布式事务、异步消息等。  

**微服务演进式设计：**
随着系统的发展或业务的需要，架构师会将一些经常变动或是有一定时间效应的内容进行微服务处理，并逐渐将原来在单体系统中多变的模块拆分出来，那些稳定不太变化的模块就形成一个核心微服务存在于整个架构之中。  

分布式系统CAP定理介绍参考：[CAP 定理的含义-阮一峰](http://www.ruanyifeng.com/blog/2018/07/cap.html)
  
_ _ _
## Eureka
**核心概念**
* 服务注册为中心：Eureka Server实现
* 服务提供者：通过Eureka Client实现
* 服务消费者：通过Eureka Client和Spring Cloud Ribbon(或者Spring Cloud Feign)实现服务消费功能，Zuul可以理解为在Eureka Client和Spring Cloud Ribbon的基础之上又增加了网关功能(即将路径请求映射到对应的微服务上)。  

很多时候，客户端既是服务提供者，也是服务消费者。  

**自我保护模式**  
如果Eureka Server最近1分钟收到renew的次数小于阈值（即预期的最小值），则会触发自我保护模式，此时Eureka Server此时会认为这是网络问题，它不会注销任何过期的实例。等到最近收到renew的次数大于阈值后，则Eureka Server退出自我保护模式。  
  
**Eureka自我保护模式被激活的条件是：在 1 分钟后，Renews (last min) < Renews threshold**  

* Renews (last min)：Eureka Server 最后 1 分钟收到客户端实例心跳的总数。
* Renews threshold：Eureka Server 期望每分钟收到客户端实例心跳的总数。

自我保护模式阈值(Renews threshold)计算：

* 每个instance的预期心跳数目 = 60/每个instance的心跳间隔秒数，其实就是2
* 阈值 = 所有注册到服务的instance的数量的预期心跳之和 *自我保护系数

以上的参数都可配置的：

* instance的心跳间隔秒数：eureka.instance.lease-renewal-interval-in-seconds，设置这个貌似是不起作用的，Spring Cloud将其固定在了30s
* 自我保护系数：eureka.server.renewal-percent-threshold（默认是0.85）

如果Eureka上注册了10个实例的话，Renews threshold = 20 * 0.85 = 17，如果Renews (last min)：即Eureka Server 最后 1 分钟收到客户端实例心跳的总数小于这个阈值的话，就认为丢包是由于网络问题导致的，不会从服务列表上删除任何服务。直到Renews (last min) > Renews threshold后从自我保护模式中自动退出。  

如果我们的实例比较少且是内部网络时，推荐关掉此选项。我们也可以通过eureka.server.enable-self-preservation = false来禁用自我保护机制。  

Eureka 的自我保护模式是有意义的，该模式被激活后，它不会从注册列表中剔除因长时间没收到心跳导致租期过期的服务，而是等待修复，直到心跳恢复正常之后，它自动退出自我保护模式。这种模式旨在避免因网络分区故障导致服务不可用的问题。例如，两个客户端实例 C1 和 C2 的连通性是良好的，但是由于C2与Eureka之间的网络故障，C2 未能及时向 Eureka 发送心跳续约，这时候 Eureka 不能简单的将 C2 从注册表中剔除。因为如果剔除了，C1 就无法从 Eureka 服务器中获取 C2 注册的服务(此时C1与Eureka之间的网络连接是良好的)，但是这时候 C2 服务是可用的。  

Eureka进入自我保护模式通常是因为多个服务实例与此Eureka之间的网络连接都出现了问题，此时问题可能出在网络连接，而不是在于注册到Eureka上的多个微服务，所以Eureka会进入自我保护模式，不会删除任何已经注册的服务实例。之后成功从此Eureka Server上获取到的服务可能是有问题的微服务，消费客户端需要注意处理这种问题。    

综上，自我保护模式是一种应对网络异常的安全保护措施。**它的架构哲学是宁可同时保留所有微服务（健康的微服务和不健康的微服务都会保留），也不盲目注销任何健康的微服务。**使用自我保护模式，可以让Eureka集群更加的健壮、稳定。  

**单机条件下进行调试相比于线上更容易进入自我保护模式的原因是单机会频繁的关闭开启eureka client，导致心跳丢失很多，并不是由于网络问题导致的心跳丢失。所以建议在本地环境关闭自我保护模式，线上再开启。**  

**问题：**
1. Eureka界面中的DS Replicas左右是什么呢？？？  为什么将eureka.instance.hostname=localhost这样设置在DS Replicas中就没有任何信息显示了呢？？？eureka.instance.hostname采用默认设置，两个EurekaServer时DS Replicas中会显示localhost。。3个EurekaServer时会显示什么呢？？？  
2. 下面这篇文章中（[Spring Cloud Eureka 的自我保护模式及相关问题]）说道如果关闭自我保护模式，会有一个可能的问题，即隔一段时间后，可能会发生实例并未关闭，却无法通过网关访问了，此时很可能是由于网络问题，导致实例（或网关）与 Eureka Server 断开了连接，Eureka Server 已经将其注销（网络恢复后，实例并不会再次注册），此时重启 Eureka Server 节点或实例，并等待一小段时间即可。**问题是Eureka在关闭自我保护模式的情况下由于网络问题剔除了服务实例之后，为什么网络恢复不能把这个实例在重新添加到可用服务列表中呢？在Eureka打开自我保护模式的情况下网络恢复后会自动将实例重新添加吗？**

参考：[Spring Cloud Eureka 的自我保护模式及相关问题](https://blog.csdn.net/t894690230/article/details/78207495)

_ _ _
### Eureka集群高可用配置
**Eureka集群available-replicas不为空需满足如下条件：** 
1. 不能用localhost比如： 
eureka.client.serviceUrl.defaultZone=http://localhost:2222/eureka/ 
要采用： 
eureka.instance.hostname=master  // 这个配置的作用是注册到eureka上时使用这个主机名，不会去尝试在程序中获取本机的ip之后去注册。  
eureka.client.serviceUrl.defaultZone=http://backup:2222/eureka/ 
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝ 
eureka.instance.hostname=backup 
eureka.client.serviceUrl.defaultZone=http://master:1111/eureka/ 
2. spring.application.name或eureka.instance.appname必须一致。 
3. 相互注册要开启： 
eureka.client.register-with-eureka=true(默认即为true)

**Eureka集群中Eureka Client与Eureka Server之间肯定会涉及到的三种骚操作：**
1. eureka.client.service-url.defaultZone:此Eureka Server作为Eureka Client向集群中的哪些Eureka Server发送服务列表变化消息从而进行服务列表同步。比如当前Eureka Server新注册上来了一个服务，会将这个服务的信息发送到所有在defaultZone中配置的Eureka Server上进行同步(这里会将消息发送给每一个defaultZone的url中包含的Eureka Server吗？)，属于主动告知式同步。  
2. eureka.client.fetch-registry：是否从上面注册的所有Eureka Server上获取注册信息，这里获取到的注册信息可以给当前Eureka Server上进行同步，对吗？？？    
3. eureka.client.register-with-eureka：是否将当前Eureka Server注册到defaultZone中其它的Eureka Server中去。为true则会在自己所在的Eureka Server和defaultZone中包含的其它Eureka Server中都进行注册，不开启的话则都不会注册。  

Eureka Client向Eureka Server注册中，client端配置defaultZone通常都会将所有Eureka Server的url都写上，我们知道只要client端在一个Eureka Server上注册成功，服务信息就会自动同步到其它Eureka Server上，那么这里为什么还要写多个Eureka Server的url呢？写一个不就好了？我认为这可能是防止Eureka Client启动时注册失败，如果写的第一个Eureka Server url不可用，再尝试第二个，直到注册成功为止。   


**问题：**
1. Eureka会从服务列表中删除已经过期的服务，那么Eureka认定服务过期的条件是什么呢？即如何通过心跳机制认定服务是否过期呢？
2. Eureka Client向Eureka Server注册中，client端配置defaultZone通常都会将所有Eureka Server的url都写上，Eureka Client启动时是在一个Eureka Server上注册成功就停止注册还是即使注册成功了也继续在其他尚未尝试过注册操作的Eureka Server上继续注册？  
3. 我们知道Eureka Client只要注册到Eureka Server集群中的一个Eureka Server上，在其他Eureka Server上也可以发现这个client，此时这个client是只与他第一个注册成功的Eureka Server维持心跳检测还是与所有的Eureka Server维持心跳检测呢？  
4. Eureka Client会在本地缓存注册到Eureka Server上的其它client，并使用更新策略定期更新，所以可能出现所有Eureka Server都挂掉的情况下Eureka Client之间还是可以互联，Eureka Client获取Eureka Server上的注册信息，是只从一个Eureka Server上获取吗？  
5. Eureka集群available-replicas与unavailable-replicas的实质性差别在哪呢？  

_ _ _
### Eureka中服务治理核心要素
* 服务注册中心
* 服务提供者
* 服务消费者

#### 服务提供者
1、服务注册：将自己注册eureka中，每个服务名称(spring.application.name)可以对应多个服务，客户端调用时会使用ribbon进行自动负载均衡。  
  
2、服务同步：当eureka配置为高可用时，由于服务中心之间互相注册为服务，当服务提供者发送注册请求到一个服务注册中心时，他会将该请求转发给集群中相连的其它注册中心(这里采用的是推送方式还是拉取方式？)，从而实现注册中心之间的服务同步。  

3、服务续约：即服务提供者与Eureka注册中心之间的心跳检测机制，用来防止Eureka Server的剔除任务将该服务实例从服务列表中排除出去。服务
续约的心跳检测时间默认为30秒，通过eureka.instance.lease-renewal-interval-in-seconds来配置，默认是30秒；使用eureka.instance.lease-expiration-duration-in-seconds配置Eureka定义服务失效的时间，默认为90s，即从最后一次收到服务提供者的心跳开始如果超过90s还没有收到下一次心跳，则将此服务从服务列表中剔除。  

4、服务下线：在服务提供者进行正常的关闭操作时，会触发一个服务下线的REST请求给Eureka Server，eureka 服务端在接收到请求之后，将该服务状态设置为down，并把该下线事件传播出去，防止客户端调用失效的服务实例。  

#### 服务消费者
1、服务获取：通过rest请求从服务注册中心中获取可用的服务提供者的列表，每隔30s更新一次。  
2、服务调用：客户端使用Ribbon根据需求调用服务提供者提供的服务。  

#### 服务注册中心
1、失效剔除：当服务实例非正常关闭的时候，eureka server并不能收到服务下线的请求，为了删除这些故障的服务，eureka server在启动时会创建一个定时任务，默认没隔60s将当前清单中超时(默认为90s)没有续约的服务剔除出去。  
2、自我保护：eureka在自我保护机制被触发之后，客户端很可能拿到实际已经不存在的服务实例，这就要求客户端要有容错机制。比如使用重试、断路器等机制。  

#### 源码分析
1、@EnableDiscoveryClient是如何将eureka client引入的：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)  // 引入了EnableDiscoveryClientImportSelector类
public @interface EnableDiscoveryClient {

	/**
	 * If true, the ServiceRegistry will automatically register the local server.
	 */
	boolean autoRegister() default true;
}

@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableDiscoveryClientImportSelector
		extends SpringFactoryImportSelector<EnableDiscoveryClient> {

    // 其selectImports会被框架自动调用
    @Override
	public String[] selectImports(AnnotationMetadata metadata) {
        // 调用父类的selectImports
		String[] imports = super.selectImports(metadata);
        ...
    }
}

public abstract class SpringFactoryImportSelector<T>
		implements DeferredImportSelector, BeanClassLoaderAware, EnvironmentAware {
    
    @Override
	public String[] selectImports(AnnotationMetadata metadata) {
        ...
        // 这句代码会自动扫描我们引入的lib中具有META-INF/spring.factories的jar包，将其中的自动配置类引入
        // 比如通过spring-cloud-netflix-eureka-client的jar包引入的几个配置类就完成了DiscoveryClient类的引入
        // 这是这个DiscoveryClient类完成了eureka client的一系列功能(服务注册、服务列表获取、服务续约等功能)
        // Find all possible auto configuration classes, filtering duplicates
		List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader
				.loadFactoryNames(this.annotationClass, this.beanClassLoader)));
        ...
    }
}
```
Eureka Client与Eureka Server之间的所有交互都是通过REST请求来发起的。  

_ _ _
## Ribbon
Ribbon实现的是客户端负载均衡，所有客户端节点都维护着自己要访问的服务端清单，并定期从注册中心拉取更新。在Spring Cloud实现的服务治理框架中，默认会创建针对各个服务治理框架的Ribbon自动化整合配置，比如Eureka中的 org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration(在spring-cloud-netflix-eureka-client-1.4.6.RELEASE.jar包中)。在实际使用的时候，可以通过查看这个类的实现，可以找到它的配置详情。    

通常使用使用Ribbon只需要两步：    
* 服务提供者启动多个服务实例并注册到一个注册中心或是多个相关联的注册中心
* 服务消费者直接通过调用被@LoadBalanced注解修饰过的RestTemplate来实现面向服务的接口调用

## Ribbon实现原理分析
因为要使用Ribbon提供的功能，必须使用@LoadBalanced注解的RestTemplate，从@LoadBalanced注解源码的注释中可以知道，该注解的作用是使得LoadBalancerClient可以配置RestTemplate标记的注解。LoadBalancerClient是Spring Cloud中定义的一个接口，顺着LoadBalancerClient接口的所属包，可以初步判断其中的LoadBalancerAutoConfiguration为实现客户端负载均衡器的自动化配置类。  

LoadBalancerAutoConfiguration起作用需要满足两个条件：
* RestTemplate类必须存在于当前工程的环境中
* 在Spring的Bean工程中必须有LoadBalancerClient的实现Bean

**LoadBalancerAutoConfiguration主要做的就是一件事：获取所有用户使用@LoadBanlanced注解标注的RestTemplate对象并为其添加LoadBalancerInterceptor拦截器。**  

当一个被@LoadBalanced注解修饰的RestTemplate对象向外发起Http请求时，就会被LoadBalancerInterceptor.intercept方法拦截，之后会调用LoadBalancerClient的实现类RibbonLoadBalancerClient代理发起请求。  

RibbonLoadBalancerClient类通过ZoneAwareLoadBalancer(默认使用方案，继承自DynamicServerListLoadBalancer)或者DynamicServerListLoadBalancer中的负载均衡策略选出一个服务提供者，将用户提供的以逻辑服务名为host的URI转换成具体的服务实例地址后发起请求。    

更具体的来说，DynamicServerListLoadBalancer使用EurekaClient动态更新Server List(默认每10s向EurekaClient发送一次ping，进而检查是否需要更新服务的注册列表信息)，RibbonLoadBalancerClient类通过DynamicServerListLoadBalancer获取ServerList，之后使用ServerListFilter选出一部分符合条件的Server List,然后使用IRule选择出最终要访问的服务提供者地址。  

我们可以通过配置users.ribbon.NFLoadBalancerRuleClassName= com.netflix.loadbalancer.WeightedResponseTimeRule来自定义负载均衡规则。  

_ _ _
## Hystrix
**出现背景：**在微服务架构中往往存在许多个服务单元，若一个服务单元出现故障，就很容易因为依赖关系引发故障的蔓延，最终导致整个系统的瘫痪，也就是常说的雪崩效应，为了解决这个问题，产生了断路器等一系列的服务保护机制。  

Hystrix具备服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并以及服务监控等强大功能。  

### 服务降级
将要被降级的方法使用@HystrixCommand注解标注，并指定降级后要调用的回调方法，还可指定触发服务降级的超时时间。       

### 服务熔断
一个服务在一段时间内被降级的次数太多，超过了我们设置的阈值，这个服务就会被熔断，之后到来的请求直接调用fallback函数。一段时间之后(有一个窗口的概念)这个服务又会处于半打开状态，尝试让一部分请求调用真正的服务，如果发现服务已经恢复的话让服务切换到正常状态，否则继续处于熔断状态。  

熔断器计算统计数据时以窗口为单位，统计的太频繁会影响性能。默认统计的间隔时间为1s还是500ms？  
  
### 请求隔离
* 信号隔离：默认每个方法共享一个信号量，默认限制数量为10，信号隔离中的信号计数Hystrix底层是通过AtomicInteger实现的。    
* 线程池隔离：默认每个类下的所有方法调用共享一个线程池，默认线程池corePoolSize与maximumSize都为10.线程池底层默认实现方式是ThreadPoolExecutor。  

对于不依赖网络访问的服务，比如只依赖内存缓存这种情况下，就不适合用线程池隔离技术，而是采用信号量隔离。  

spring cloud中可以通过@HystrixCommand注解的threadPoolKey属性进行线程池划分，使用commandKey进行信号量分组划分，像下面这样：  
```java
@HystrixCommand(commandKey = "", threadPoolKey = "")
```

[Hystrix之信号量隔离熔断实现原理](https://www.jianshu.com/p/3c47e681bf0a)  
### 请求合并
微服务架构中的依赖通常用过远程调用实现，而远程调用最常见的问题就是通信消耗与连接数占用。在**高并发**的情况下，因通信次数的增加，总的通信时间消耗将会变得不那么理想。同时，因为依赖服务的线程池资源有限，将出现排队等待与相应延迟的情况。为了优化这两个问题，Hystrix提供了HystrixCollapser来实现请求的合并，来减少通信消耗和线程数的占用。  

请求合并有一个合并时间窗的概念，在一个时间窗内的请求会被收集之后批量发送给服务提供者，这就要求服务提供者提供批量处理的方法并且服务消费者在进行服务请求时指定单次请求被收集之后对应的批量请求方法。可以使用@HystrixCollapsr注解指定要被收集的方法，配合@HystrixCommand注解调用批量处理方法完成服务调用。  

**请求合并的额外开销：**
比如某个请求不通过请求合并器访问的平均耗时为5ms，请求合并的延迟时间窗为10ms(默认值)，那么当该请求设置了请求合并器之后，最坏情况下(在延迟时间窗结束时才发起请求)该请求需要15ms才能完成。  

**是否使用请求合并需要依赖服务调用的实际情况来选择，主要考虑下面两个方面：**  
* 请求命令本身的延迟：如果依赖服务的请求命令本身是一个高延迟的命令，那么可以使用请求合并器，因为延迟时间窗的时间消耗显得微不足道了。
* 延迟时间窗内的并发量：如果一个时间窗内只有1~2个请求，那么这样的依赖服务不适合使用请求合并器。这种情况不但不能提升系统性能，反而会成为系统瓶颈，因为每个请求都要多消耗一个时间窗才响应。相反，如果一个时间窗内具有很高的并发量，并且服务提供方也实现了批量处理接口，那么使用请求合并器可以有效减少网络链接数量并极大提升系统吞吐量，此时延迟时间窗所增加的消耗就可以忽略不计了。    


### 结果缓存
在spring cloud中，压力通常来源于对依赖服务的调用，因为请求依赖服务的资源需要通过通信来实现，这样的依赖方式比起进程内的调用方式会引起一部分的性能损失，同时HTTP相比于其他高性能的通信协议在速度上没有任何优势，所以它有些类似于对数据库这样的外部资源进行读写操作，在高并发的情况下可能会成为系统的瓶颈。所以在Hystrix提供了请求缓存的功能，请求缓存的好处有：  
* 减少重复的请求数，降低依赖服务的并发度
* 在同一用户请求的上下文中，相同依赖服务的返回数据始终保持一致
* 请求缓存在run()和construct()执行之前生效，可以有效减少不必要的线程隔离开销

Hystrix使用了一个线程安全的Map来保存请求的响应结果，以我们重写的getCacheKey方法返回的结果作为Map的key值。    

在Spring Cloud中可以使用@HystrixCommand与@CahceResult、@CacheKey联合使用将请求结果加入缓存，也可以使用@HystrixCommand与@CahceRemove、@CacheKey联合使用将请求结果从缓存中剔除。  

参考文章：  
[Hystrix原理与实战（文章略长）](https://my.oschina.net/7001/blog/1619842)   
[Hystrix技术解析(图片版)](https://www.jianshu.com/p/b6e8d91b2a96)  
[Hystrix 配置参数全解析](https://www.cnblogs.com/zhenbianshu/p/9630167.html)  

_ _ _
## Feign

### 集成Ribbon的功能
Feign对Ribbon与Hystrix进行了进一步的封装，使得服务之间的依赖调用非常优雅(伪RPC)。      

默认情况下，Feign会将所有Feign客户端的方法都封装到Hystrix命令中进行服务保护，当然也可以通过feign.hystrix.enabled=false来关闭Hystrix功能。  

在Feign中配置Ribbon的超时重试时间时要注意Ribbon的超时重试时间设置一定要比Hystrix的超时时间要小，要不然Hystrix超时后直接就fallback了，ribbon的超时重试机制根本没有发挥作用的余地。  

### 请求压缩
可配置，比如合一指定请求压缩的大小下限，只有超过这个大小的请求才会对其进行压缩。    

通过请求压缩可减少通信过程中的性能损耗。  

### 工作原理
Feign的实现原理也是动态代理，通过扫描@Feign注解标注的接口，通过spring的IOC方式注入到我们的应用中。之后生成接口对应的代理RequestTemplate模板对象，通过代理类实现对依赖服务的伪RPC调用以及上述Feign的核心功能。  

_ _ _
## Zuul
作为微服务系统的网关组件，用于构建边界服务，核心功能是基础服务进行聚合提供用户所需要的功能，在此基础之上还提供了动态路由、过滤、监控、弹性伸缩和安全等能力。   

Zuul中自身就包含了对于hystrix和ribbon模块的依赖，所以Zuul天生就有线程隔离和断路器的自我保护功能，以及对服务调用的客户端负载均衡功能。但是需要注意，当使用path与url的映射关系来配置路由规则的时候，对于路由转发的请求不会采用HystrixCommand来包装，所以这类路由请求没有线程隔离和断路器的保护，并且也不会有负载均衡的能力。我们在使用Zuul的时候尽量使用path与serviceId的组合来进行配置，这样不仅能保证API网关的健壮和稳定，也能用到Ribbon的客户端负载均衡功能。    

当然我们在使用Zuul搭建API网关的时候，也可以通过Hystrix和Ribbon的参数来调整路由请求的各种超时时间等配置。  

### 请求路由
核心功能，依据我们的配置将客户端的请求转发到对应的微服务上。  


### 请求过滤
当需要鉴权时，如果为每个基础类型的微服务都实现一套用于校验签名和鉴别权限的过滤器或拦截器不可取，这会增加日后系统的维护难度。更好的做法是通过前置的网关服务来完成这些非业务性质的校验。  

Zuul中实现请求过滤时需要继承ZuulFilter。  

**核心过滤器：**在Zuul中，为了让API网关组件可以被更方便的使用，它在HTTP请求生命周期的各个阶段默认实现了一批核心过滤器，他们会在API网关服务启动的时候被自动加载和启用。正是通过这些核心过滤器的设计，完成了Zuul的请求路由功能。  

在Spring Cloud Zuul中实现的过滤器必须包含四个基本特性：**过滤类型、执行顺序、执行条件、具体操作。**他们的含义与功能总结如下：  
* filteType：代表过滤器的类型，zuul中默认定义了4种不同生命周期(pre、routing、post、error)的过滤器类型，穿插于HTTP声明周期请求的各个阶段。  
* filterOrder：通过int值来定义过滤器的执行顺序，数值越小优先级越高
* shouldFilter：返回一个boolean值来判断过滤器是否需要执行，可以通过此方法来指定过滤器的有效范围
* run：过滤器的具体逻辑。在该函数中可以实现自定义的过滤逻辑，来确定是否要拦截当前的请求，或是在请求返回结果之后对其进行一些加工等等

**在我们实现自定义过滤器时，要注意对自定义过滤器中出现的异常的处理，可以参考zuul中核心过滤器中异常的处理过程以及springcloud微服务实战书中的介绍。**   

在zuul中，对于我们在application.properties中配置的url匹配关系是通过ZuulServlet来处理的，其余的还是通过DispatcherServlet来处理的。    

_ _ _
## 微服务架构下的分布式事务解决方案  

分布式事务产生的原因：
* Service 产生多个节点，即微服务架构，订单服务下单接口需要调用商品服务的扣库存接口，如果没有分布式事务，可能出现扣库存成功但是下单失败的情况，这就出现了库存减少了，但商品没有卖出去的情况，这在分布式系统会出现业务逻辑问题。  
* Resource 产生多个节点：举例来说，可以是数据库的一个表中的数据分布在多个不同Mysql实例中，如果一条数据库命令涉及到多个数据库实例上的数据操作，要通过数据库层面上的分布式事务来保证事务一致性。  

[阿里中间件部门研发的GTS-微服务架构下分布式事务解决方案](https://yq.aliyun.com/articles/542020?utm_content=m_43990)  
[再有人问你分布式事务，把这篇扔给他](https://juejin.im/post/5b5a0bf9f265da0f6523913b)   
[(精品) 分布式事务实现方式介绍之-2PC与3PC](https://www.hollischuang.com/archives/1580)   
