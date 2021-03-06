---
layout:     post
title:      SpringAOP使用方法分析总结
date:       2017-07-24
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Spring
---
AOP(Aspect Orient Programming),面向切面编程。作为面向对象编程的一种补充，广泛应用于处理一些具有横切性质的系统级服务，如事务管理，安全检查，异常处理，日志记录等。AOP实现的关键就在于AOP框架自动创建的AOP代理，AOP代理又可分为静态代理与动态代理两大类，如下图所示：
![AOP代理图](/img/2017-07-24/aop代理图.png)
&emsp;&emsp;静态代理是指使用AOP框架提供的命令进行编译，从而在编译阶段生成AOP代理类，也称为编译时增强，常用的框架有AspectJ；而动态代理则在运行时借助于JDK动态代理、CGLIB等在内存中临时生成AOP代理类，也被称为运行时增强。  
&emsp;&emsp;Spring在使用注解方式配置AOP时会使用到AspectJ相关的jar包，这并不代表Spring使用了静态代理。使用AspectJ相关jar包的原因是有些注解的实现需要其支持，真正在代理的时候采用的实现方式还是动态代理。  
&emsp;&emsp;被代理的类如果实现了某个接口，Spring会采用JDK反射方式进行动态代理，否则使用CGLib动态代理。  
&emsp;&emsp;对于静态代理AspectJ使用方式不再过多介绍，接下来主要介绍Spring框架中AOP的使用方法及我使用过程中发现的问题，并从AOP动态代理实现角度分析为何会出现这样的问题。  

_ _ _
### **1. SSM整合框架中使用AOP对Controller，Service层进行拦截**

&emsp;&emsp;Spring框架中AOP配置方式有两种，一是通过注解方式的配置，二是通过xml方式的配置，主要介绍基于注解方式的配置。  
- 注解配置拦截Controller层  
   
&emsp;&emsp;在SSM整合框架中，我把Controller层扫描加载的配置放在了SpringMVC-servlet.xml中，所以若想拦截Controller层进行相关操作，对于AOP的配置代码必须放在SpringMVC-servlet.xml中。对于注解方式的配置，只需在
SpringMVC-servlet.xml文件中增加一句代码：
```xml
	<aop:aspectj-autoproxy/>
```  
&emsp;&emsp;这句代码的意思是开启AOP的自动代理方式，即在被代理类是某个接口的实现类时采用JDK动态代理方式，否则使用CGLib进行动态代理，两种代理方式的区别下文会进行介绍。若想一直采用CGLib代理方式，可将此句代码修改为:  
```xml
	<aop:aspectj-autoproxy proxy-target-class="true"/>
```  
&emsp;&emsp;即可实现一直使用CGLib方式进行动态代理。完整SpringMVC-servlet.xml文件配置如下：
```xml
	<?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:context="http://www.springframework.org/schema/context"
           xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:aop="http://www.springframework.org/schema/aop"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context-4.1.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
        <aop:aspectj-autoproxy/>
        <context:component-scan base-package="com.michael.controller"></context:component-scan>
        <!-- 配置注解驱动后，Springmvc才可以正确解析@Controller, @RequestMapping等注解，完成请求转发-->
        <!--mvc:annotation-driven会启用ｍｖｃ编程相关模型，比如自动注册ｊｓｏｎ转换器 -->
        <mvc:annotation-driven/>
        <mvc:resources mapping="/pages/**" location="/WEB-INF/pages/"/>

        <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
            <property name="prefix" value="/pages/"/>
            <property name="suffix" value=".html"/>
        </bean>
    </beans>
```  
&emsp;&emsp;被拦截的Controller层代码具体如下：
```java
    @Controller
    public class UserController {
        @Resource
        private UserService userService;  

        @RequestMapping(value = "/addUser")
        public @ResponseBody Map<String, String> addUser(String userName, int userAge) {
            User user = new User();
            user.setUserName(userName);
            user.setUserAge(userAge);
            userService.addUser(user);
            Map<String, String> map = new HashMap<>();
            map.put("result", "s");
            return map;
        }
    }
```  
&emsp;&emsp;AOP切面代码如下：
```java
    @Component
    @Aspect//定义切面标识
    public class AnnotationAOP {
    	//代理com.michael.controller包下的UserController类的所有方法
        @Before("execution(* com.michael.controller.UserController.*(..))")
        public void beforeTest(JoinPoint joinPoint) {
            System.out.println("模拟日志记录...");  
        }
    }
```  

&emsp;&emsp;之后在浏览器中访问/addUser对应的链接，会发现在addUser方法被调用之前，确实调用了切面中的beforeTest方法。可知实际执行的绝不是UserController对象的方法，而是由AOP产生的代理对象的方法。  

- 注解配置拦截Service层  

&emsp;&emsp;拦截Service层的实现除以下两点外，其余与拦截Controller层大致相同。  
	&emsp;&emsp;（1）由于Service层扫描加载的配置位于Spring.xml（ApplicationContext.xml）中，所以若想对Service层进行切面相关操作，对于AOP的配置代码必须放在Spring.xml中。  
    &emsp;&emsp;（2）由于Spring实现AOP的原理是动态代理，所以必须用由Spring IOC容器控制的Service对象调用被切面增强的方法才能看到切面效果，而不能自己直接用new产生一个service层的对象去调用被切面增强的方法。
    
&emsp;&emsp;具体代码如下：

&emsp;&emsp;Spring.xml:
```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:context="http://www.springframework.org/schema/context"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
        <aop:aspectj-autoproxy/>
        <!-- 支持注解配置bean 识别@Resource @Autowired等注解-->
        <context:annotation-config></context:annotation-config>
        <context:component-scan base-package="com.michael.service"/>
        <context:component-scan base-package="com.michael.aop"/>
        <import resource="classpath:/config/mybatis-context.xml"/>
    </beans>
```

&emsp;&emsp;UserService:
```java
    @Service
    @Transactional
    public class UserService {
        @Resource
        private UserDao userDao;

        public int addUser(User user) {
            return userDao.addUser(user);
        }
    }
```
&emsp;&emsp;调用UserService的controller层代码如下：
```java
    @Controller
    public class UserController {
    	//注意userService对象必须交由IOC容器进行控制才能被切面增强
        @Resource
        private UserService userService;  

        @RequestMapping(value = "/addUser")
        public @ResponseBody Map<String, String> addUser(String userName, int userAge) {
            User user = new User();
            user.setUserName(userName);
            user.setUserAge(userAge);
            //UserService userService = new UserService(); 这样写是没有切面增强效果的！！！
            userService.addUser(user);
            Map<String, String> map = new HashMap<>();
            map.put("result", "s");
            return map;
        }
    }
```  
&emsp;&emsp;AOP切面代码如下：
```java
    @Component
    @Aspect//定义切面标识
    public class AnnotationAOP {
    	//代理com.michael.service包下的UserService类的所有方法
        @Before("execution(* com.michael.service.UserService.*(..))")
        public void beforeTest(JoinPoint joinPoint) {
            System.out.println("模拟日志记录...");  
        }
    }
```  

&emsp;&emsp;之后在浏览器中访问/addUser对应的链接，会发现在UserService类的addUser方法被调用之前，确实调用了切面中的beforeTest方法。可知实际执行的绝不是UserService对象的方法，而是由AOP产生的代理对象的方法。  

_ _ _
### **2. 使用Spring AOP过程中遇到的问题**

（1）从一Bean的被切面增强的方法中使用this调用此Bean另一个被切面增强的方法，被调用的方法无切面增强效果。

&emsp;&emsp;代码示例如下：
```java
	@Service
    @Transactional
    public class UserService implements UserServiceInterface {
        @Resource
        private UserDao userDao;

        public int addUser(User user) {
        	//只有使用AopContext.currentProxy()获得当前对象的代理对象之后再调用本类的方法才有切面效果
        	((UserService)AopContext.currentProxy()).testPropagation();
            //直接使用this调用本类方法是没有切面增强效果的
            System.out.println("this为: " + this + " 代理对象为: " + AopContext.currentProxy());
            System.out.println("this == 代理对象? " + (this == AopContext.currentProxy()));
        	//testPropagation();
            return userDao.addUser(user);
        }
        
        public void testPropagation() {
        	System.out.println("是否会被切面增强？");
        }
    }
```
&emsp;&emsp;运行此段代码后发现this.toString()方法与AopContext.currentProxy().toString()方法输出相同，但this == 代理对象?输出为false。由于java中 = = 操作符号比较的是两个对象的内存地址，所以执行addUser方法的当前对象与代理对象确实是两个对象，不通过代理对象直接调用本类中的其它方法没有切面效果是可以理解的，上面问题得到解决。  
&emsp;&emsp;但是toString()方法输出内容相同，两个属于不同类的对象均调用默认的toString()方法，理论上来说结果不可能相同，难道当前对象的toString()方法也被默认代理了，执行AopContext.currentProxy().toString()实际执行的是当前对象的toString()方法? 由于当前类采用的AOP实现方式是JDK动态代理，接下来通过分析JDK动态代理实现机制来解释这个问题。  

- 分析JDK动态代理实现机制  

&emsp;&emsp;先看一个JDK动态代理实现AOP的一个小Demo。  
```java
    public interface HelloWorld {
        void sayHello();
    }
    //被代理类必须实现一个或多个接口
    public class HelloWorldImpl implements HelloWorld {
        @Override
        public void sayHello() {
            System.out.println("hello world！");
        }
    }
    public class MyInvocationHandler implements InvocationHandler {
        //被代理的目标类对象
        private Object target;

        public MyInvocationHandler(Object target) {
            this.target = target;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            //在执行目标类的方法之前可以进行指定操作
            System.out.println("准备输出hello world");
            return method.invoke(target, args);
        }
    }
    //产生代理对象的工具类
    public class ProxyFactory {
        public static Object getProxy(Object target) {
            return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                            target.getClass().getInterfaces(),
                              new MyInvocationHandler(target));
        }
    }
    public class JDKProxyTest {
    	public static void main(String[] args) {
        	//将动态生成的代理类保存到磁盘
            system.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
            helloWorld = (HelloWorld)ProxyFactory.getProxy(new HelloWorldImpl());
            helloWorld.sayHello();
        }        
    }
```
&emsp;&emsp;输出结果：  
&emsp;&emsp;&emsp;&emsp;准备输出hello world  
&emsp;&emsp;&emsp;&emsp;hello world！  
&emsp;&emsp;实现了代理效果。反编译jdk生成的动态代理代码如下：
```java
    public final class $Proxy0 extends Proxy implements HelloWorld {
        private static Method m1;
        private static Method m3;
        private static Method m2;
        private static Method m0;

        public $Proxy0(InvocationHandler var1) throws  {
            super(var1);
        }

        public final boolean equals(Object var1) throws  {
            try {
                return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
            } catch (RuntimeException | Error var3) {
                throw var3;
            } catch (Throwable var4) {
                throw new UndeclaredThrowableException(var4);
            }
        }

        public final void sayHello() throws  {
            try {
                super.h.invoke(this, m3, (Object[])null);
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }

        public final String toString() throws  {
            try {
                return (String)super.h.invoke(this, m2, (Object[])null);
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }

        public final int hashCode() throws  {
            try {
                return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
            } catch (RuntimeException | Error var2) {
                throw var2;
            } catch (Throwable var3) {
                throw new UndeclaredThrowableException(var3);
            }
        }

        static {
            try {
                m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
                m3 = Class.forName("com.michael.DynamicProxy.HelloWorld").getMethod("sayHello", new Class[0]);
                m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
                m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
            } catch (NoSuchMethodException var2) {
                throw new NoSuchMethodError(var2.getMessage());
            } catch (ClassNotFoundException var3) {
                throw new NoClassDefFoundError(var3.getMessage());
            }
        }
    }
```
&emsp;&emsp;生成的代码十分清晰，一目了然，代理类除了代理HelloWorld接口中的sayHello()方法之外，还将自身的equals(),hashCode(),toString()方法交给被代理类来执行，也就是说我们在使用代理类执行这三个方法时，实际执行的是被代理类(本例中为HelloWorldImpl)的equals(), hashCode(), toString()方法，这也就解释了上面的为何被代理类与代理类toString()方法输出的内容完全相同的问题。  
&emsp;&emsp;学习JDK实现动态代理机制中尚未解决的问题：查资料见到有人说因为是Java中不允许多重继承，JDK生成的代理类已经继承了Proxy类，所以不能再继承要被代理的类，所以JDK动态代理要求被代理类必须实现接口，那么JDK生成的代理类为何必须要继承Proxy类呢？？？  

- 分析CGLib动态代理实现机制

```java
public class Test {

    private static final String SEPARATOR = File.separator;
    private static final String PROJECT_ROOTPATH = System.getProperty("user.dir");
    private static final String SAVE_PATH = PROJECT_ROOTPATH + SEPARATOR + "cglibClasses";

    public static void main(String[] args) {
        //代理类class文件存入本地磁盘
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, SAVE_PATH);
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(PersonService.class);
        enhancer.setCallback(new CglibProxyIntercepter());

        PersonService proxy= (PersonService) enhancer.create();

        proxy.setPerson();
        proxy.getPerson("1");
    }
}

public class CglibProxyIntercepter implements MethodInterceptor {
    /**
     * sub：cglib生成的代理对象，method：被代理对象方法，objects：方法入参，methodProxy:代理方法
     *
     * FastClass机制就是对一个类的方法建立索引，通过索引来直接调用相应的方法,为了避免通过反射调用，提高效率
     */
    @Override
    public Object intercept(Object sub, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("执行前...");
        // 通过引入MethodProxy类和动态生成的FastClass类使得可以直接调用被代理对象的方法而无需通过反射，这样更加高效
        Object object = methodProxy.invokeSuper(sub, objects);
        System.out.println("执行后...");
        return object;
    }
}

public class PersonService {
    public PersonService() {
        System.out.println("PersonService构造");
    }
    //该方法不能被子类覆盖，不会被CGLib动态代理
    final public Person getPerson(String code) {
        System.out.println("PersonService:getPerson>>"+code);
        return null;
    }

    public void setPerson() {
        System.out.println("PersonService:setPerson");
    }
}

执行结果：
PersonService构造
执行前...
PersonService:setPerson
执行后...
PersonService:getPerson>>1
```  
在CGLib动态代理执行过后，磁盘上保存了3个动态生成的类，经过debug得知一个类是动态生成的代理类(类名中不含FastClassByCGLIB)，其余两个类是在此代理类的代理方法被调用的时候生成的fastClass类，fastClass类中含有被代理类各个方法的索引，通过其可以直接调用被代理类的各个方法，这样的调用方式比JDK动态代理的反射调用方式更加高效。  

CGLIB创建代理对象时所花费的时间却比JDK多。所以对于单例的对象，因为无需频繁创建对象，用CGLIB合适，反之使用JDK方式要更为合适一些。同时由于CGLib由于是采用动态创建子类的方法，对于final修饰的方法无法进行代理。   

具体CGLib原理不再分析，参见： [Cglib动态代理实现方式](https://www.cnblogs.com/monkey0307/p/8328821.html)   

(2)一个方法被多个切面增强，此方法被调用时各个切面的执行顺序如何判断？  

&emsp;&emsp;对于在同一个切面定义的通知函数将会根据在类中的声明顺序执行。如下所示:
```xml
<?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:context="http://www.springframework.org/schema/context"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
           <context:component-scan base-package="com.AOPTest"/>
           <context:annotation-config/>
            <aop:aspectj-autoproxy  expose-proxy="true"/>
    </beans>
```

```java
    @Component
    @Aspect
    public class InOneAspect {
        /**
         * Pointcut定义切点函数
         */
        @Pointcut("execution(* com.AOPTest.MyBean.*(..))")
        private void myPointcut(){}

        @Before("myPointcut()")
        public void beforeOne(){
            System.out.println("前置通知....执行顺序1");
        }

        @Before("myPointcut()")
        public void beforeTwo(){
            System.out.println("前置通知....执行顺序2");
        }

        @AfterReturning(value = "myPointcut()")
        public void AfterReturningThree(){
            System.out.println("后置通知....执行顺序3");
        }

        @AfterReturning(value = "myPointcut()")
        public void AfterReturningFour(){
            System.out.println("后置通知....执行顺序4");
        }
    }
    @Component
    public class MyBean implements MyBeanInterface {
        @Override
        public void sayHello() {
            System.out.println("Hello!");
        }
    }
    public class Main {
        public static void main(String[] args) {
            ApplicationContext context = new ClassPathXmlApplicationContext(
                    "/applicationContext.xml", Main.class
            );
            MyBeanInterface myBean = context.getBean(MyBeanInterface.class);
            myBean.sayHello();
        }
    }
```
输出结果：  
&emsp;&emsp;前置通知....执行顺序1  
&emsp;&emsp;前置通知....执行顺序2  
&emsp;&emsp;Hello!  
&emsp;&emsp;后置通知....执行顺序4  
&emsp;&emsp;后置通知....执行顺序3  

&emsp;&emsp;如果在不同的切面中定义多个通知响应同一个切点，进入时则优先级高的切面类中的通知函数优先执行，退出时则最后执行，优先级由切面类实现的Ordered接口中getOrder方法返回值确定，返回值越小，优先级越高。如下：  
```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:context="http://www.springframework.org/schema/context"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
           xsi:schemaLocation="http://www.springframework.org/schema/beans
                http://www.springframework.org/schema/beans/spring-beans.xsd
                http://www.springframework.org/schema/context
                http://www.springframework.org/schema/context/spring-context-4.1.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
           <context:component-scan base-package="com.AOPTest"/>
           <context:annotation-config/>

           <bean id = "executionTimeLoggingSpringAop" class = "com.AOPTest.ExecutionTimeLoggingSpringAOP"/>
           <bean id = "secondAOPAspect" class = "com.AOPTest.SecondAOPAspect"/>
           <aop:config expose-proxy="true">
              <aop:pointcut id = "executionTimeLoggingPointcut" expression = "execution(public * *(..))"/>
              <aop:advisor id = "executionTimeLoggingAdvisor" advice-ref = "executionTimeLoggingSpringAop"
                            pointcut-ref="executionTimeLoggingPointcut"/>
           </aop:config>
            <aop:config expose-proxy="true">
                <aop:pointcut id="secondAOPAspectPointcut" expression="execution(public * *(..))"/>
                <aop:advisor id = "secondAOPAspectAdvisor" advice-ref = "secondAOPAspect"
                             pointcut-ref="secondAOPAspectPointcut"/>
            </aop:config>
    </beans>
```
```java
	//切面类1
    public class ExecutionTimeLoggingSpringAOP implements MethodBeforeAdvice, AfterReturningAdvice, Ordered {
        @Override
        public void before(Method method, Object[] objects, Object o) throws Throwable {
            System.out.println("ExecutionTimeLoggingSpringAOP前置通知");
        }

        @Override
        public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
            System.out.println("ExecutionTimeLoggingSpringAOP后置通知");
        }

        //返回值越小优先级越高
        @Override
        public int getOrder() {
            return 1;
        }
    }
    //切面类2
    public class SecondAOPAspect implements MethodBeforeAdvice, AfterReturningAdvice, Ordered {
        @Override
        public void afterReturning(Object o, Method method, Object[] objects, Object o1) throws Throwable {
            System.out.println("SecondAOPAspect后置通知");
        }

        @Override
        public void before(Method method, Object[] objects, Object o) throws Throwable {
            System.out.println("SecondAOPAspect前置通知");
        }

        @Override
        public int getOrder() {
            return 0;
        }
    }
    @Component
    public class MyBean implements MyBeanInterface {
        @Override
        public void sayHello() {
            System.out.println("Hello!");
        }
    }
    public class Main {
        public static void main(String[] args) {
            ApplicationContext context = new ClassPathXmlApplicationContext(
                    "/applicationContext.xml", Main.class
            );
            MyBeanInterface myBean = context.getBean(MyBeanInterface.class);
            myBean.sayHello();
        }
    }
```
输出结果：  
&emsp;&emsp;SecondAOPAspect前置通知  
&emsp;&emsp;ExecutionTimeLoggingSpringAOP前置通知  
&emsp;&emsp;Hello!  
&emsp;&emsp;ExecutionTimeLoggingSpringAOP后置通知  
&emsp;&emsp;SecondAOPAspect后置通知  

(完)  
    




