# SpringFramework学习笔记
待读：黄勇《架构探险-从零开始写Java Web框架》
看源码-》深入理解其功能实现-》学习其设计理念和设计模式，从宏观-》微观

## 一、Spring IOC容器实现部分学习
1. org.springframework.beans 和 org.springframework.context 包是 Spring Framework 的 IoC 容器的根本
2. 有两个主要的容器系列，一个是实现BeanFactory接口的简单容器系列，这系列容器只实现了容器的最基本功能；另一个是ApplicationContext应用上下文，它作为容器的高级形态而存在。ApplicationContext 是 BeanFactory 的一个子接口。ApplicationContext 使得和 Spring 的AOP 功能集成变得更简单；添加了信息资源处理（国际化中使用），事件发布；还添加了应用程序层的特殊上下文 ，如用于 web 应用程序的 WebApplicationContext。**简而言之，BeanFactory提供了配置框架和基本功能，而ApplicationContext 添加了更多企业应用功能。**

**学完这部分之后要解决的问题：**  
* IOC容器类库的整体设计思想，BeanFactory各个子类作用分析，为何分成这么多子类；ApplicationContext各个子类作用分析，为何分成这么多子类；
* IOC容器(BeanFactory、ApplicationContext)实现原理、启动流程、带来的好处
* 在创建各种Bean的时候如何处理Bean间各种依赖关系？如何做到以xml、注解、javaconfig等多种方式读入Bean？
* Spring中BeanFactory的生命周期、Bean的生命周期、各种作用域区分、作用于Bean各个作用域的回调函数使用
* Spring容器级别回调使用及实现，比如BeanPostProcessor、BeanFactoryPostProcessor
* Spring事件处理机制(接口ApplicationEventPublisher)

### Bean生命周期回调
Bean生命周期图示参见《Spring实战》20页。

在Spring Bean生命周期中，Spring提供用来扩展的抽象类及接口，有的是对单个Bean有效，有的则是对所有Bean都有效。  

比如InitializingBean、DisposableBean、FactoryBean及init-method、destory-method针对的都是某个Bean控制及其初始化的操作，而BeanPostProcessor、BeanFactoryPostProcessor、InstantiationAwareBeanPostProcessor则是对所有Bean都生效。  

### BeanFactory-屌丝容器
这个类体现了Spring为提供给用户使用的IOC容器所设定的最基本的功能规范，它是作为一个最基本的接口类出现在Spring的IOC容器体系中的。  

API文档翻译：  
```java
用于访问Spring bean容器的根接口。 这是bean容器的基本客户端视图; 诸如ListableBeanFactory和ConfigurableBeanFactory之类的其他接口可用于特定目的。

此接口由包含多个bean定义的对象实现，每个bean定义由String名称唯一标识。 根据bean定义，工厂将返回包含对象的独立实例（Prototype设计模式）或单个共享实例（Singleton设计模式的替代实例，其中实例是范围中的单例 工厂）。 将返回哪种类型的实例取决于bean工厂配置：API是相同的。 从Spring 2.0开始，根据具体的应用程序上下文（例如，Web环境中的“请求”和“会话”范围），可以使用更多范围。

这种方法的重点是BeanFactory是应用程序组件的中央注册表，并集中应用程序组件的配置（例如，不再单个创建对象，而是从属性文件中读取并配置创建）。 有关此方法的优点的讨论，请参阅“Expert One-on-One J2EE设计和开发”的第4章和第11章。

请注意，通常最好依靠依赖注入（“push” configuration）通过setter或构造函数来配置应用程序对象，而不是像BeanFactory查找一样使用任何形式的“pull” configuration。 Spring的依赖注入功能是使用这个BeanFactory接口及其子接口实现的。

通常，BeanFactory将加载存储在配置源（例如XML文档）中的bean定义，并使用org.springframework.beans包来配置bean。 但是，实现可以直接在Java代码中直接返回它创建的Java对象。 对如何存储定义没有限制：LDAP，RDBMS，XML，属性文件等。鼓励实现支持bean之间的引用（依赖注入）。

与ListableBeanFactory中的方法相反，如果这是HierarchicalBeanFactory，则此接口中的所有操作也将检查父工厂。 如果在此工厂实例中找不到bean，则会询问直接父工厂。 此工厂实例中的Bean应该在任何父工厂中覆盖同名的Bean。

Bean工厂实现应尽可能支持标准bean生命周期接口。 完整的初始化方法及其标准顺序是：
1. BeanNameAware's setBeanName
2. BeanClassLoaderAware's setBeanClassLoader
3. BeanFactoryAware's setBeanFactory
4. EnvironmentAware's setEnvironment
5. EmbeddedValueResolverAware's setEmbeddedValueResolver
6. ResourceLoaderAware's setResourceLoader (only applicable when running in an application context)
7. ApplicationEventPublisherAware's setApplicationEventPublisher (only applicable when running in an application context)
8. MessageSourceAware's setMessageSource (only applicable when running in an application context)
9. ApplicationContextAware's setApplicationContext (only applicable when running in an application context)
10. ServletContextAware's setServletContext (only applicable when running in a web application context)
11. postProcessBeforeInitialization methods of BeanPostProcessors
12. InitializingBean's afterPropertiesSet
13. a custom init-method definition
14. postProcessAfterInitialization methods of BeanPostProcessors

关闭bean工厂时，以下生命周期方法适用：
1. postProcessBeforeDestruction methods of DestructionAwareBeanPostProcessors
2. DisposableBean's destroy
3. a custom destroy-method definition
```

**BeanFactory与FactoryBean比较：**  
[spring中BeanFactory和FactoryBean的区别](https://blog.csdn.net/qiesheng/article/details/72875315)  

### ApplicationContext-牛b容器
Spring ApplicationContext 接口提供了几种即装即用的实现方式。在独立应用中，通常以创建 ClassPathXmlApplicationContext 或FileSystemXmlApplicationContext 的实例。虽然 XML 一直是传统的格式来定义配置元数据，但也可以指示容器使用 Java 注解或代码作为元数据格式，并通过提供少量的XML配置以声明方式启用这些额外的元数据格式的支持。  

基于 XML 的元数据并不是配置元数据的唯一允许的形式。Spring IoC容器本身是完全与配置元数据的实际写入格式分离的。  

API文档翻译：
```java
用于为应用程序提供配置的中央接口。 这在应用程序运行时是只读的，但如果实现支持，则可以重新加载。

ApplicationContext提供：
1. 用来获取应用中其它组件的Bean factory类的方法，继承自ListableBeanFactory.
2. 通过通常方式加载文件资源的能力，继承自ResourceLoader interface.
3. 发布事件到监听的监听器上的能力，继承自ApplicationEventPublisher interface.
4. 解析消息，支持国际化的能力。继承自MessageSource接口.
5. 从父上下文继承。 后代上下文中的定义始终优先。 这意味着，例如，整个Web应用程序可以使用单个父上下文，而每个servlet都有自己的子上下文，该上下文独立于任何其他servlet的子上下文.

总而言之，ApplicationContext除了标准的BeanFactory生命周期功能之外，ApplicationContext实现还检测并调用ApplicationContextAware bean以及ResourceLoaderAware，ApplicationEventPublisherAware和MessageSourceAware bean。
```

ApplicationContext的实现类其实是包装了DefaultListableBeanFactory类，所有的有关bean的获取等操作都会委托到DefaultListableBeanFactory。  

**只有在不得不使用BeanFactory的时候才使用它(比如应用对内存使用要求很严格)，否则都应当使用ApplicationContext。Spring容器ApplicationContext启动时内部开启了许多BeanPostProcessor扩展点，比如对于事务处理和AOP的支持都是通过BeanPostProcessor实现的，如果仅仅使用BeanFactory，那这些特性就不能使用了。**  

<img src="/img/otherblog/BeanFactoryAndApplicationContextContrast.png" width="700" height="700" alt="BeanFactory与ApplicationContext对比图" />
<center>BeanFactory与ApplicationContext对比图</center>    

<img src="/img/otherblog/BeanFactoryAndApplicationContextContrast1.png" width="700" height="700" alt="BeanFactory与ApplicationContext类图对比" />
<center>BeanFactory与ApplicationContext类图对比</center>    

总结：

bean工厂后置处理器的主要作用是针对beanDefinition，bean后置处理器的主要作用是针对bean实例化及初始化。

根据两种处理器的特点及切入点在实际应用中我们可以自定义处理器开发各种插件功能。

## 二、Spring AOP实现部分学习
AOP虽然与设计公共子模块有几分相似，但在传统的公共子模块调用中，除了直接硬调用之外并没有其他的手段，而AOP为处理这一类问题提供了一套完整的理论和灵活多样的实现方法。  

AOP思想可以总结为基础+切面+配置(编织)；

Spring AOP的实现和其他特性的实现一样，除了可以使用Spring本身提供的AOP实现之外，还封装了业界优秀的AOP解决方案AspectJ来供应用使用。在Spring自身对于Aop的实现中，充分利用了**IOC容器Proxy代理对象以及AOP拦截器的功能特性**，通过这些对AOP基本功能的封装机制，为用户提供了AOP的实现框架。  

**Spring AOP使用两个关键：**  

* 切面：定义具体织入方式(前置通知、后置通知、返回通知、异常通知、环绕通知)以及具体织入的程序代码是什么
* 切入点表达式：定义在满足什么条件时(即何时)进行织入，比如expression="execution(* com.michael.controller.*.*(..))"

**由切入点表达式所匹配的连接点的概念是AOP的核心，默认情况下Spring使用AspectJ切入点表达式语言。Spring只支持方法级别的切入点(因为Spring AOP构建在动态代理基础之上)，对于构造器和字段级别的切入点不支持**

**Spring环绕通知的设计是为了防止解决这个问题：**    
在多线程环境下，在joinpoint方法调用前后的处理中需要共享一些数据。如果使用Before advice和After advice也可以达到目的，但是就需要在aspect里面创建一个存储共享信息的field，而且这种做法并不是线程安全的。

具体可参考：  
[正确理解Spring AOP中的Around advice](http://www.cnblogs.com/csniper/p/5499248.html)  

**AOP框架均可以将切面在切入点织入到目标对象中，在目标对象的生命周期里有多个点可以进行织入：**  
* 编译期：切面在目标类编译时被织入。这种方式需要特殊的编译器。AspectJ的织入编译器就是以这种方式织入切面的。
* 类加载期：切面在目标类加载到JVM时被织入。这种方式需要特殊的类加载器(ClassLoader)，它可以在目标类被引入应用之前增强该目标类的字节码。
* 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象动态的创建一个代理对象。SpringAOP就是以这种方式织入切面的。  

**学完这部分之后要解决的问题：** 
* Spring AOP带来的好处
* Spring AOP几种配置方式
* Spring AOP实现原理(JDK动态代理、CGLIB) 两种方式具体区别在哪？？？  
* 待完成：SpringAOP整个设计类图意图分析，SpringAOP官方文档学习  


## 三、Spring MVC实现部分学习及servlet知识再总结
**学完这部分之后要解决的问题：** 
* Spring MVC 应用上下文(WebApplicationContext)在web中的启动与销毁
* Spring MVC DispatcherServlet启动与销毁
* Spring MVC 请求处理流程，涉及的mvc框架组件，应用的设计模式
* struts2是如何与Spring结合到一起的？？？  

## 四、Spring JDBCTemplate及Mybatis适配实现部分学习

## 五、Spring 事务控制实现部分学习


## 代码分析心得
以组件的方式对每种框架进行分析。从Demo开始进行分析。    

