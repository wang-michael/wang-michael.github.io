# SpringCloud学习笔记
单体系统变得庞大臃肿之后会产生难以维护的问题，将其拆分为微服务可以做到每个服务都运行在自己的进程内，在部署上有稳定的边界，每个服务的更新都不会影响到其它服务的运行。同时，由于是独立部署的，我们可以更准确的为每个服务评估性能容量，通过配合服务间的协作流程也可以更容易的发现系统的瓶颈位置，以及给出较为准确的系统级性能容量评估。**微服务还有一大好处在于服务组件化之后，整个架构中的各个组件就能针对其不同的业务特点选择不同的技术平台。**     

微服务理念带来的问题：   
* 运维上的挑战：如何有条不紊的将这些进程编排和组织起来，即如何进行更好的服务治理？？？    
* 分布式的复杂性：由于拆分后的各个微服务都是独立部署并运行在各自的进程内，他们只能通过通信来进行协作，所有分布式环境的问题都将是微服务架构系统设计时需要考虑的重要因素，比如网络延迟、分布式事务、异步消息等。  

**微服务演进式设计：**
随着系统的发展或业务的需要，架构师会将一些经常变动或是有一定时间效应的内容进行微服务处理，并逐渐将原来在单体系统中多变的模块拆分出来，那些稳定不太变化的模块就形成一个核心微服务存在于整个架构之中。  

分布式系统CAP定理介绍参考：[CAP 定理的含义-阮一峰](http://www.ruanyifeng.com/blog/2018/07/cap.html)

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

### Eureka集群高可用配置
**Eureka集群available-replicas不为空需满足如下条件：** 
1. 不能用localhost比如： 
eureka.client.serviceUrl.defaultZone=http://localhost:2222/eureka/ 
要采用： 
eureka.instance.hostname=master 
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



## 微服务架构下的分布式事务解决方案  
分布式事务产生的原因：
* Service 产生多个节点，即微服务架构，订单服务下单接口需要调用商品服务的扣库存接口，如果没有分布式事务，可能出现扣库存成功但是下单失败的情况，这就出现了库存减少了，但商品没有卖出去的情况，这在分布式系统会出现业务逻辑问题。  
* Resource 产生多个节点：举例来说，可以是数据库的一个表中的数据分布在多个不同Mysql实例中，如果一条数据库命令涉及到多个数据库实例上的数据操作，要通过数据库层面上的分布式事务来保证事务一致性。  

[阿里中间件部门研发的GTS-微服务架构下分布式事务解决方案](https://yq.aliyun.com/articles/542020?utm_content=m_43990)  
[再有人问你分布式事务，把这篇扔给他](https://juejin.im/post/5b5a0bf9f265da0f6523913b)   
[(精品) 分布式事务实现方式介绍之-2PC与3PC](https://www.hollischuang.com/archives/1580)   
