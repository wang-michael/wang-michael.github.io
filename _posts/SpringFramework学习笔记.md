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
* 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象动态的创建一个代理对象。SpringAOP就是以这种
方式织入切面的。  

**aop:aspectj-autoproxy标签处理：**
通过aop命名空间的<aop:aspectj-autoproxy />声明自动为spring容器中那些配置@aspectJ切面的bean创建代理，织入切面。当然，spring
在内部依旧采用AnnotationAwareAspectJAutoProxyCreator进行自动代理的创建工作，但具体实现的细节已经被<aop:aspectj-autoproxy />
隐藏起来了，对这个自定义标签的处理的主要任务就是在WAC中注册一个BeanPostProcessor，也就是AnnotationAwareAspectJAutoProxyCreator类的实例对象，之后使用这个processor对符合切面要求的bean进行代理增强。
具体处理过程可参考：[SpringAOP源码解析之aop:aspectj-autoproxy标签解析](https://blog.csdn.net/heroqiang/article/details/79037741)
注意父WAC中的切面可以对子WAC中的bean进行增强，子WAC中的切面不能对父WAC中的bean进行增强。要被增强的bean的在哪个容器中，哪个容器中就要注册相应的BeanPostProcessor来对要被切面增强的bean进行代理对象的生成。    

父容器中的BeanPostProcessor对于子容器中的bean是不生效的，子容器中的BeanPostProcessor对于父容器中的bean也是这样。  

<aop:aspectj-autoproxy/> 


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

Spring MVC中含有多种HandlerMapping，用来负责分发不同类型的请求到对应的Controller，RequestMappingHandlerMapping是其中一种，用来将@RequestMapping中url对应的请求分发到对应的方法，每个DispatcherServlet对应一个RequestMappingHandlerMapping；每个RequestMappingHandlerMapping默认会处理自己所在的WebApplicationContext及父容器中的所有的带有@RequestMapping的注解，当然也可以设置只处理自己所在的WebApplicationContext中的@RequestMapping注解，建立起Controller中处理方法与请求url之间的对应关系。  

**问题：为什么要设置一个公用上下文和一个子上下文？都使用这个公用上下文不就好了？**
在一个SpringMVC的web应用中，可以有一个公用的父上下文和多个DispatcherServlet，其中每个DispatcherServlet都可以有自己的上下文，可以只实例化自己想要控制的Controller，对于那些不想被公用的bean定义比如这里只想自己管的Controller)，需要在自己的上下文配置文件中定义。  

父上下文与子上下文中的上下文是分开管理的吗？比如在父上下文中创建的BeanPostProcessor对于子上下文中创建的bean是不是不生效呢？
答案：是分开管理的，父WAC中的BeanPostProcessor不会拦截子WAC中的bean。  

## 四、Spring JDBCTemplate及Mybatis适配实现部分学习
### Java原生JDBC部分
Java原生JDBC：java访问数据库的基石，其它框架如Hibernate、MyBatis等ORM只是更好的封装了Jdbc。
  
待解决问题：
* JDBC如何实现Java程序和JDBC驱动松耦合
* 如何创建一个JDBC连接
* JDBC的DriverManager是用来做什么的
* JDBC三种statement各自区别、功能
* JDBC中批处理好处
* JDBC事务控制：提交、回滚、隔离级别

参考文章：[JDBC常见面试题](https://segmentfault.com/a/1190000013312766)  



## 五、Spring 事务控制实现部分学习

## 六、Spring 标签解析
默认标签包含4个：alias、import、bean、beans（当然这里所说的标签不包括那些以子标签形式存在的如property、value等标签）  
自定义标签：如我们熟知的事务标签<tx:annotation-driven/>、注解扫描标签<context:component-scan/>等都属于自定义标签  
参考文章：
[Spring源码解析之默认标签的解析](https://blog.csdn.net/heroqiang/article/details/78599052)  
[Spring源码解析之自定义标签的解析](https://blog.csdn.net/heroqiang/article/details/78611213)  
[Spring源码解析之context:component-scan标签解析](https://blog.csdn.net/heroqiang/article/details/79019359)  
[Spring Component注解处理过程](https://www.cnblogs.com/leodaxin/p/9593793.html)

先找到配置文件(xml文件即application-context.xml,java文件即含有@Configuration注解标注的文件)，加载配置文件，对配置文件中的默认标签(默认注解)以及自定义标签(自定义注解)进行解析，解析过程如上述链接所述。  

每一个Java注解都有自己对应的注解处理器，在注解处理器中进行注解的相关处理工作，实现注解的作用。@EnableScheduling注解和<task:annotation-driven>标签实现的功能是一样的，<task:annotation-driven>由在spring-tx包下的META-INF/Spring.handlers：http\:// www.springframework.org/schema/tx=org.springframework.transaction.config.TxNamespaceHandler规定了对应关系之后反射加载TxNamespaceHandler进行处理，可以认为@EnableScheduling注解对应的注解处理器实现的功能与TxNamespaceHandler是一致的，这里不再进行深究，关于java注解的处理以及如何自己创建一个注解以及编写对应的注解处理器，可以参考：[Java中的注解(Annotation)处理器解析](https://blog.csdn.net/yang_yang1994/article/details/79729621)   

@EnableScheduling注解与@Schedule注解配合使用(或者<task:annotation-driven>与Schedule标签配合使用)实现了Spring定时任务功能的应用上的简易性。@EnableScheduling注解处理器做的最重要一个工作就是注册了一个ScheduledAnnotationBeanPostProcessor，@Schedule注解处理器的作用是在BeanDefinition中标记需要被ScheduledAnnotationBeanPostProcessor处理的方法，之后在Bean实际被创建时由ScheduledAnnotationBeanPostProcessor开启相应的定时任务。  

## 代码分析心得
以组件的方式对每种框架进行分析。从Demo开始进行分析。    

## SpringBoot部分学习
SpringBoot在启动的时候会将@SpringBootApplication注解标注的类作为配置类来加载，SpringBoot遵循约定大于配置的方式，即需要改配置的时候再去改，否则使用其默认配置就好。  

SpringBoot默认配置之一就是对于Controller以及其他Component注解类的扫描，会扫描与Application类同级下的包中的所有用注解标注的类。其默认配置中对于父AC容器会扫描除了@Controller注解外的其它所有@Component类型的注解，对于子AC容器会扫描@Controller注解，所以@Controlller注解标注的类并不会被实例化两次。

Springboot关于自动配置部分的源码在spring-boot-autoconfigure-1.5.10.jar中，若想直到SpringBoot为我们做了哪些自动配置，可以查看这里的源码。    


### 各种注解学习
**元注解：**可以注解到别的注解上的注解，Spring的很多注解都可以作为元注解，比如@Component；
**组合注解：**被元注解注解的注解，比如@Configuration就是一个@Component注解，表明这个类其实也是一个Bean。

使用示例：
```java
// 自定义一个组合注解，同时具有@Configuration与@ComponentScan的功能
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration //1
@ComponentScan //2
public @interface WiselyConfiguration {
	
	String[] value() default {}; //3

}
```  

**@Conditional注解：**在满足@Conditional中的特定条件的时候才创建此特定的Bean。使用示例：
```java
public class WindowsCondition implements Condition {
    // 调用其matches方法判断是否满足其条件
    public boolean matches(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        return context.getEnvironment().getProperty("os.name").contains("Windows");
    }
}
public class LinuxCondition implements Condition {

    public boolean matches(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        return context.getEnvironment().getProperty("os.name").contains("Linux");
    }

}
public class WindowsListService implements ListService {
	@Override
	public String showListCmd() {
		return "dir";
	}
}
public class LinuxListService implements ListService{
	@Override
	public String showListCmd() {
		return "ls";
	}
}

@Configuration
public class ConditionConifg {
    // 调用其WindowsCondition的matches方法判断是否满足其条件，满足才创建此bean
	@Bean
    @Conditional(WindowsCondition.class) //1
    public ListService windowsListService() {
        return new WindowsListService();
    }
    @Bean
    @Conditional(LinuxCondition.class) //2
    public ListService linuxListService() {
        return new LinuxListService();
    }
}
public class Main {
	public static void main(String[] args) {
		AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(ConditionConifg.class);
		
		ListService listService = context.getBean(ListService.class);		
		
		System.out.println(context.getEnvironment().getProperty("os.name") 
				+ "cmd: " 
				+ listService.showListCmd());
		
		context.close();
	}
}
```
**@Import注解的工作原理：**在应用中，有时没有把某个类注入到IOC容器中，但在运用的时候需要获取该类对应的bean，此时就需要用到@Import注解。既可以使用@Import导入普通的Bean，也可以导入一个配置类，导入配置类的示例如下：  
```java
package com.example.demo;
import org.springframework.context.annotation.Bean;

public class MyConfig {

    @Bean
    public Dog getDog(){
        return new Dog();
    }

    @Bean
    public Cat getCat(){
        return new Cat();
    }

}

package com.example.demo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Import;

//@SpringBootApplication
@ComponentScan
/*导入配置类就可以了*/
@Import(MyConfig.class)
public class App {

    public static void main(String[] args) throws Exception {

        ConfigurableApplicationContext context = SpringApplication.run(App.class, args);
        System.out.println(context.getBean(Dog.class));
        System.out.println(context.getBean(Cat.class));
        context.close();
    }
}
```

**@Enable注解的工作原理：**比如使用@EnableAspectJAutoProxy开启对AspectJ自动代理的支持，使用@EnableScheduling开启计划任务的支持，使用@EnableWebMVC开启Web MVC的配置支持。  

通过这种注解配置方式，可以避免我们配置大量的代码。通过观察这些@Enable注解的源码可以发现这些注解的共性就是都有一个@Import注解，它是用来导入配置类的，这也就意味着这些自动开启的实现其实是导入了一些自动配置的Bean。这些导入的配置方式主要分为以下三种类型：  

* 直接导入配置类：

```java
@EnableScheduling
public class Main {	
}

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Import({SchedulingConfiguration.class}) //导入了一个配置类 SchedulingConfiguration
@Documented
public @interface EnableScheduling {
}

@Configuration
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
* 依据条件选择配置类：

```java
@EnableAsync
public class Main {	
}

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AsyncConfigurationSelector.class}) // 在AsyncConfigurationSelector根据条件选择需要导入的配置类
public @interface EnableAsync {
    Class<? extends Annotation> annotation() default Annotation.class;

    boolean proxyTargetClass() default false;

    AdviceMode mode() default AdviceMode.PROXY;

    int order() default 2147483647;
}

public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {
    private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME = "org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";

    public AsyncConfigurationSelector() {
    }

    // 若adviceMode为PORXY，则返回ProxyAsyncConfiguration这个配置类，否则返回AspectJAsyncConfiguration
    public String[] selectImports(AdviceMode adviceMode) {
        switch(null.$SwitchMap$org$springframework$context$annotation$AdviceMode[adviceMode.ordinal()]) {
        case 1:
            return new String[]{ProxyAsyncConfiguration.class.getName()};
        case 2:
            return new String[]{"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration"};
        default:
            return null;
        }
    }
}

@Configuration
public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {
    ... 
}
```
* 动态注册Bean：

```java
@EnableAspectJAutoProxy
public class Main {	
}

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AspectJAutoProxyRegistrar.class})
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
}

// ImportBeanDefinitionRegistrar的作用是在运行时自动添加Bean到已有的配置类：  
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    AspectJAutoProxyRegistrar() {
    }

    // AnnotationMetadata用来获取当前配置类上的注解；BeanDefinitionRegistry参数用来注册Bean
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        AnnotationAttributes enableAJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if(enableAJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }

    }
}
```
### SpringBoot运作原理分析
关于SpringBoot运作原理，我们还是从@SpringBootApplication注解开始分析，这个注解是一个组合注解，它的核心功能是由@EnableAutoConfiguration注解提供的。  
```java
@SpringBootApplication
public class Chapter1Application {
    public static void main(String[] args) {
        SpringApplication.run(Chapter1Application.class, args);
    }
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ...
}

@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class) // 关键是使用@Import注解导入的配置功能
public @interface EnableAutoConfiguration {
    ...
}

// EnableAutoConfigurationImportSelector使用SpringFactoriesLoader.loadFactoryNames来扫描具有META-INF/spring.factories
// 的jar包，而spring-boot-autoconfigure-1.3.0.jar中就有一个spring.facories文件，此文件中声明了有哪些自动配置类，比如
// org.springframework.boot.autoconfigure.aop.AopAutoConfiguration，并对这些自动配置类进行注册
@Deprecated
public class EnableAutoConfigurationImportSelector
		extends AutoConfigurationImportSelector {
	@Override
	protected boolean isEnabled(AnnotationMetadata metadata) {
		if (getClass().equals(EnableAutoConfigurationImportSelector.class)) {
			return getEnvironment().getProperty(
					EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
					true);
		}
		return true;
	}
}

public class AutoConfigurationImportSelector
		implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
		BeanFactoryAware, EnvironmentAware, Ordered {
    @Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		try {
			AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
					.loadMetadata(this.beanClassLoader);
			AnnotationAttributes attributes = getAttributes(annotationMetadata);
			List<String> configurations = getCandidateConfigurations(annotationMetadata,
					attributes);
			configurations = removeDuplicates(configurations);
			configurations = sort(configurations, autoConfigurationMetadata);
			Set<String> exclusions = getExclusions(annotationMetadata, attributes);
			checkExcludedClasses(configurations, exclusions);
			configurations.removeAll(exclusions);
			configurations = filter(configurations, autoConfigurationMetadata);
			fireAutoConfigurationImportEvents(configurations, exclusions);
			return configurations.toArray(new String[configurations.size()]);
		}
		catch (IOException ex) {
			throw new IllegalStateException(ex);
		}
	}

    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
        // 使用SpringFactoriesLoader.loadFactoryNames来扫描具有META-INF/spring.factories
        // 的jar包，而spring-boot-autoconfigure-1.3.0.jar中就有一个spring.facories文件，此文件中声明了有哪些自动配置类
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
}
```
观察spring-boot-autoconfigure-1.3.0.jar中的spring.facories文件中声明的自动配置类，一般都有一些条件注解，比如@ConditionalOnBean，@ConditionalOnWebApplication(org.springframework.boot.autoconfigure.web.WebMvcAutoConfiguration中就含有此注解)，正是由这些条件注解来决定这些自动配置要不要起作用。下面就以@ConditionalOnWebApplication为例进行分析。  
```java
// ConditionalOnWebApplication注解是一个组合注解，具有@Conditional的作用，需要满足的条件时
// OnWebApplicationCondition.matches方法
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnWebApplicationCondition.class)
public @interface ConditionalOnWebApplication {

}

public abstract class SpringBootCondition implements Condition {
    @Override
	public final boolean matches(ConditionContext context,
			AnnotatedTypeMetadata metadata) {
		String classOrMethodName = getClassOrMethodName(metadata);
		try {
            // 调用OnWebApplicationCondition类的getMatchOutcome方法
			ConditionOutcome outcome = getMatchOutcome(context, metadata);
			...
	}
}

@Order(Ordered.HIGHEST_PRECEDENCE + 20)
class OnWebApplicationCondition extends SpringBootCondition {
    @Override
	public ConditionOutcome getMatchOutcome(ConditionContext context,
			AnnotatedTypeMetadata metadata) {
		boolean required = metadata
				.isAnnotated(ConditionalOnWebApplication.class.getName());
		ConditionOutcome outcome = isWebApplication(context, metadata, required);
		if (required && !outcome.isMatch()) {
			return ConditionOutcome.noMatch(outcome.getConditionMessage());
		}
		if (!required && outcome.isMatch()) {
			return ConditionOutcome.noMatch(outcome.getConditionMessage());
		}
		return ConditionOutcome.match(outcome.getConditionMessage());
	}

	private ConditionOutcome isWebApplication(ConditionContext context,
			AnnotatedTypeMetadata metadata, boolean required) {
		ConditionMessage.Builder message = ConditionMessage.forCondition(
				ConditionalOnWebApplication.class, required ? "(required)" : "");
		if (!ClassUtils.isPresent(WEB_CONTEXT_CLASS, context.getClassLoader())) {
			return ConditionOutcome
					.noMatch(message.didNotFind("web application classes").atAll());
		}
		if (context.getBeanFactory() != null) {
			String[] scopes = context.getBeanFactory().getRegisteredScopeNames();
			if (ObjectUtils.containsElement(scopes, "session")) {
				return ConditionOutcome.match(message.foundExactly("'session' scope"));
			}
		}
		if (context.getEnvironment() instanceof StandardServletEnvironment) {
			return ConditionOutcome
					.match(message.foundExactly("StandardServletEnvironment"));
		}
		if (context.getResourceLoader() instanceof WebApplicationContext) {
			return ConditionOutcome.match(message.foundExactly("WebApplicationContext"));
		}
		return ConditionOutcome.noMatch(message.because("not a web application"));
	}
}
```
从isWebApplication方法可以看出，判断条件是：
1. GenericWebApplicationContext是否在类路径中
2. 容器中是否有名为session的scope
3. 当前容器的Environment是否为StandardServletEnvironment
4. 当前的ResourceLoader(ResourceLoader是ApplicationContext的顶级接口之一)是否为WebApplicationContext
5. 我们需要构造ConditionOutCome类的对象来帮助我们，最终通过ConditionOutCome.isMatch方法返回布尔值来确定条件。  

下面就来分析一个Spring Boot内置的自动配置功能的实现：http编码配置(org.springframework.boot.autoconfigure.web.HttpEncodingAutoConfiguration类)。  
```java
@Configuration
@EnableConfigurationProperties(HttpEncodingProperties.class)
@ConditionalOnWebApplication
@ConditionalOnClass(CharacterEncodingFilter.class)
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration {

	private final HttpEncodingProperties properties;

	public HttpEncodingAutoConfiguration(HttpEncodingProperties properties) {
		this.properties = properties;
	}

	@Bean
	@ConditionalOnMissingBean(CharacterEncodingFilter.class)
	public CharacterEncodingFilter characterEncodingFilter() {
        // 读取HttpEncodingProperties中的配置来完成编码配置
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
		return filter;
	}

	@Bean
	public LocaleCharsetMappingsCustomizer localeCharsetMappingsCustomizer() {
		return new LocaleCharsetMappingsCustomizer(this.properties);
	}

	private static class LocaleCharsetMappingsCustomizer
			implements EmbeddedServletContainerCustomizer, Ordered {
		private final HttpEncodingProperties properties;
		LocaleCharsetMappingsCustomizer(HttpEncodingProperties properties) {
			this.properties = properties;
		}
		@Override
		public void customize(ConfigurableEmbeddedServletContainer container) {
			if (this.properties.getMapping() != null) {
				container.setLocaleCharsetMappings(this.properties.getMapping());
			}
		}
		@Override
		public int getOrder() {
			return 0;
		}
	}
}

@ConfigurationProperties(prefix = "spring.http.encoding")
public class HttpEncodingProperties {

	public static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");

	/**
	 * Charset of HTTP requests and responses. Added to the "Content-Type" header if not
	 * set explicitly.
	 */
	private Charset charset = DEFAULT_CHARSET;

	/**
	 * Force the encoding to the configured charset on HTTP requests and responses.
	 */
	private Boolean force;

	/**
	 * Force the encoding to the configured charset on HTTP requests. Defaults to true
	 * when "force" has not been specified.
	 */
	private Boolean forceRequest;

	/**
	 * Force the encoding to the configured charset on HTTP responses.
	 */
	private Boolean forceResponse;

	/**
	 * Locale to Encoding mapping.
	 */
	private Map<Locale, Charset> mapping;

	public Charset getCharset() {
		return this.charset;
	}

	public void setCharset(Charset charset) {
		this.charset = charset;
	}

	public boolean isForce() {
		return Boolean.TRUE.equals(this.force);
	}

	public void setForce(boolean force) {
		this.force = force;
	}

	public boolean isForceRequest() {
		return Boolean.TRUE.equals(this.forceRequest);
	}

	public void setForceRequest(boolean forceRequest) {
		this.forceRequest = forceRequest;
	}

	public boolean isForceResponse() {
		return Boolean.TRUE.equals(this.forceResponse);
	}

	public void setForceResponse(boolean forceResponse) {
		this.forceResponse = forceResponse;
	}

	public Map<Locale, Charset> getMapping() {
		return this.mapping;
	}

	public void setMapping(Map<Locale, Charset> mapping) {
		this.mapping = mapping;
	}

	boolean shouldForce(Type type) {
		Boolean force = (type == Type.REQUEST ? this.forceRequest : this.forceResponse);
		if (force == null) {
			force = this.force;
		}
		if (force == null) {
			force = (type == Type.REQUEST);
		}
		return force;
	}

	enum Type {

		REQUEST, RESPONSE

	}

}

```  
可以看到HttpEncodingAutoConfiguration类中含有一个characterEncodingFilter bean的配置声明，就是通过这个bean来完成http encoding的设置的。   

HttpEncodingAutoConfiguration在满足其上的几个@Conditional的几个注解的条件时才会生效，比如@ConditionalOnClass(CharacterEncodingFilter.class)需要满足的条件是在classpath路径下含有CharacterEncodingFilter类。  

### SpringBoot自动配置包含哪些
**web相关配置：**
1. 把类路径下的/static，/public，/resources和/META-INF/resources文件夹下的静态文件直接映射为/**,可以通过localhost:8080/**来访问。  
2. HttpMessageConverters支持:比如负责json消息转换的MessageConverter。
3. 静态首页的支持：比如把index.html文件放置在如下目录，当我们访问应用根目录localhost:8080/时，会直接映射。  
4. DispatcherServlet自动配置实现：org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration，**Spring所有自动配置的实现类在spring-boot-autoconfigure-xxx.jar中的META-INF/spring.facories文件中声明。**  

当SpringBoot提供的Spring MVC默认配置不符合当前需求时，则可以通过一个配置类(注解含有@Configuration的类)加上@EnableWebMvc注解来实现完全自己控制的MVC配置。注意**使用了@EnableWebMvc注解就会屏蔽SpringBoot的自动配置**，通常情况下我们只需要自定义Spring MVC的部分配置，其余使用Spring Boot的默认配置就好，这时可以自定义一个配置类并继承WebMvcConfigureAdapter，无需使用@EnableWebMvc注解，然后添加我们自定义的SpringMVC配置。    

**问题：**
* 如何在SpringBoot中配置aop区分对SpringMVC中管理的bean和Spring管理的bean的拦截？

### SpringBootActuator
启用actuator之后：
1、在http://localhost:8083/beans可以查看到应用程序上下文里都有哪些Bean；
2、可以通过http://localhost:8083/autoconfig查看到应用程序里生效的和未生效的自动配置；
3、可以通过http://localhost:8083/env查看应用程序可用的所有环境属性的列表，无论这些属性是否用到，这其中包括
环境变量、JVM属性、命令行参数以及application.properties或application.yml文件提供的属性；
4、可以通过http://localhost:8083/configprops端点生成的报告查看我们使用ConfigurationProperties注解标注
的读取配置文件的类的属性设置情况，比如项目中的ConstantNumConfig类；
5、可以通过http://localhost:8083/mappings生成控制器都映射到了哪些端点上(@RequestMapping注解上设置的属性)

运行时度量：
1、可以通过http://localhost:8083/metrics查看运行中的应用程序的相关信息，比如通过获取的应用程序的内存情况
有助于决定给JVM分配多少内存。metrics中包含的信息有：  
* 系统内存总量（mem），单位:Kb
* 空闲内存数量（mem.free），单位:Kb
* 处理器数量（processors）
* 系统正常运行时间（uptime），单位:毫秒
* 应用上下文（就是一个应用实例）正常运行时间（instance.uptime），单位:毫秒
* 系统平均负载（systemload.average）
* 堆信息（heap，heap.committed，heap.init，heap.used），单位:Kb，heap代表堆可申请的最大内存，
* 线程信息（threads，thread.peak，thead.daemon）
* 类加载信息（classes，classes.loaded，classes.unloaded）
* 垃圾收集信息（gc.xxx.count, gc.xxx.time）
* 最大连接数（datasource.xxx.max）
* 最小连接数（datasource.xxx.min）
* 活动连接数（datasource.xxx.active）
* 连接池的使用情况（datasource.xxx.usage）

其中，mem代表JVM当前使用的总虚拟内存，mem=mem.free+heap.used+nonheap.used=heap.committed+nonheap.used。  

mem不等于在任务管理器上看到的进程占用的内存大小，因为虚拟内存对应的物理内存有可能还没被分配。  

指定-server -Xms128m -Xmx128m之后，jvm申请的堆初始内存(heap.init)即为128M,heap.committed可能比其略小；但是这只是占用的虚拟内存，实际的物理内存占用比它要小的多(操作系统的延时分配策略)。  

**metrics中展示的jvm当前使用的总内存仅仅是由JVM管理的内存，包括堆和非堆，DirectBuffer，MappedByteBuffer(任务管理器上可以看见DirectBuffer的内存分配，看不到MappedByteBuffer的)这种堆外内存在mem中是不包含的。**

要想查看DirectBuffer和MappedByteBuffer的内存分配情况，可以通过jconsole中MBean一栏中的java.nio.BufferPool.direct查看直接内存的分配情况，可以通过java.nio.BufferPool.mapped查看mappedByteBuffer的分配情况。    


**问题：**   



具体参考：[Spring Boot Actuator metrics mem and mem.free](https://stackoverflow.com/questions/35354693/spring-boot-actuator-metrics-mem-and-mem-free)  
[SpringBoot关于Metrics返回值的源码-org.springframework.boot.actuate.endpoint.SystemPublicMetrics类](https://github.com/spring-projects/spring-boot/blob/v1.3.2.RELEASE/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/endpoint/SystemPublicMetrics.java)

### SpringBoot web应用部署
SpringBoot提供的SpringBootServletInitializer是一个支持SpringBoot的Spring WebApplicationInitializer实现，除了配置Spring的DispatcherServlet，他还会在Spring应用程序上下文中查找Filter、Servlet或者ServletContextInitializer类型的bean，将它们绑定到Servlet容器中。    

Spring WebApplicationInitializer实现就是用来替代web.xml配置的，具体如何实现替代的，参看下面这篇文章：[SpringBoot 中的 ServletInitializer](https://blog.csdn.net/qq_28289405/article/details/81279742)    
[在spring boot中配置多个DispatcherServlet](https://blog.csdn.net/tiger0709/article/details/78909417)


