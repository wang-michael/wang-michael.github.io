---
layout:     post
title:      Spring学习系列之Spring事务处理机制的实现
date:       2018-9-17
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Spring
---
>本文记录我对Spring事务处理机制的实现原理的理解(基于Spring3.2版本)。       

_ _ _
### **前言**
先从一个SSM整合的Demo开始看起，在applicationContext.xml配置文件中，先使用TransactionProxyFactoryBean为目标bean生成事务代理的配置。虽然此方式是最传统，配置文件最臃肿、难以阅读的方式，但更有助于理解Spring的事务处理机制。applicationContext.xml配置如下：  
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:jdbc="http://www.springframework.org/schema/jdbc"
       xmlns:jee="http://www.springframework.org/schema/jee"
       xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/aop
            http://www.springframework.org/schema/aop/spring-aop.xsd
            http://www.springframework.org/schema/tx
            http://www.springframework.org/schema/tx/spring-tx-3.2.xsd
            http://www.springframework.org/schema/jdbc
            http://www.springframework.org/schema/jdbc/spring-jdbc-3.2.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context-3.2.xsd
	    	http://www.springframework.org/schema/jee
	    	http://www.springframework.org/schema/jee/spring-jee-3.2.xsd">
    	<!-- 支持注解配置bean -->
		<context:annotation-config></context:annotation-config>
		<!--使用annotation 自动注册bean,并检查@Required,@Autowired的属性已被注入-->
	 	<!-- base-package为需要扫描的包（含所有子包）  -->
		<context:component-scan base-package="com.michael"/>

    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:/config/database.properties</value>
                <!--要是有多个配置文件，只需在这里继续添加即可 -->
            </list>
        </property>
    </bean>

    <bean id="poolDataSource"  abstract="true">
        <property name="maxActive" value="500"/>  <!-- 连接池的最大数据库连接数。设为0表示无限制。 -->
        <property name="initialSize" value="100"/>  <!-- 初始化连接数量 -->
        <property name="maxWait" value="50000"/>  <!-- 最大建立连接等待时间。如果超过此时间将接到异常。设为-1表示无限制。 -->
        <property name="removeAbandonedTimeout" value="60"/>  <!--自我中断时间秒 -->
        <property name="minEvictableIdleTimeMillis" value="30000"/>  <!--连接的超时时间，默认为半小时。-->
        <property name="minIdle" value="100"/>  <!-- 最小等待连接中的数量,设 0 为没有限制 -->
        <property name="timeBetweenEvictionRunsMillis" value="30000"/>  <!-- #运行判断连接超时任务的时间间隔，单位为毫秒，默认为-1，即不执行任务。 -->
        <property name="jmxEnabled" value="true"/>  <!-- 注册池JMX。的默认值是true。-->
        <property name="testWhileIdle" value="false"/>  <!--默认值是false,当连接池中的空闲连接是否有效 -->
        <property name="testOnBorrow" value="true"/> <!-- 默认值是true，当从连接池取连接时，验证这个连接是否有效-->
        <property name="validationInterval" value="30000"/>  <!--检查连接死活的时间间隔（单位：毫妙） 0以下的话不检查。默认是0。 -->
        <property name="testOnReturn" value="false"/>  <!--默认值是flase,当从把该连接放回到连接池的时，验证这个连接是 -->
        <property name="validationQuery" value="select 1"/>  <!--一条sql语句，用来验证数据库连接是否正常。这条语句必须是一个查询模式，并至少返回一条数据。可以为任何可以验证数据库连接是否正常的sql-->
        <property name="logAbandoned" value="false"/>  <!--是否记录中断事件， 默认为 false-->
        <property name="removeAbandoned" value="true"/>  <!-- 是否自动回收超时连接-->
        <!--这些拦截器将被插入到链中的一个java.sql.Connection对象的操作都是以拦截器。默认值是空的。
              预定义的拦截器：
              org.apache.tomcat.jdbc.pool.interceptor.ConnectionState - 跟踪自动提交，只读目录和事务隔离级别。
              org.apache.tomcat.jdbc.pool.interceptor.tatementFinalizer - 跟踪打开的语句，并关闭连接时返回到池中。-->
        <property name="jdbcInterceptors" value="org.apache.tomcat.jdbc.pool.interceptor.ConnectionState;org.apache.tomcat.jdbc.pool.interceptor.StatementFinalizer"/>
    </bean>

    <bean id="dataSource" class="org.apache.tomcat.jdbc.pool.DataSource" destroy-method="close" parent="poolDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatisStudy?characterEncoding=utf8&amp;allowMultiQueries=yes" />
        <property name="username" value="root" />
        <property name="password" value="mima_mima" />
    </bean>

    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>

    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!--dataSource属性指定要用到的连接池-->
        <property name="dataSource" ref="dataSource"/>
        <!--configLocation属性指定mybatis的核心配置文件-->
        <property name="configLocation" value="classpath:config/Configuration.xml" />
        <property name="mapperLocations" value="classpath*:com/michael/mapper/*.xml" />
    </bean>

    <bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
        <property name="mapperInterface" value="com.michael.mapper.UserMapper"/>
        <property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
    </bean>

    <bean id="exactUserService" class="com.michael.service.UserService">
        <property name="userMapper" ref="userMapper"/>
    </bean>
 
    // 为UserService加上事务控制的代理
    <bean id="userService" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
        <property name="transactionManager">
            <ref bean="transactionManager"/>
        </property>
        <property name="transactionAttributes">
            <props>
                <prop key="*">PROPAGATION_REQUIRED</prop>
            </props>
        </property>
        <property name="target">
            <ref local="exactUserService"/>
        </property>
    </bean>
   
    <aop:aspectj-autoproxy/>
</beans>
```
从上面的配置中可以看出，Spring事务处理模块是通过TransactionProxyFactoryBean来实现声明式事务处理的，比如事务属性的配置和读取、事务对象的抽象等。TransactionProxyFactoryBean是Spring AOP思想的体现之一，通过这个TransactionProxyFactoryBean可以生成Proxy代理对象，在这个代理对象中，通过TransactionInterceptor来完成对代理方法的拦截，正是这些AOP的拦截功能，将事务处理的功能编织进来。  

在Spring事务处理中，对于具体的事务处理实现，比如事务的生成、提交、回滚、挂起等，由于不同的底层数据库有不同的支持方式，因此，在Spring事务处理中，对主要的事务实现做了一个抽象和适配。适配的具体事务处理器包括：对DataSource数据源的事务处理支持，对Hibernate数据源的事务处理支持，对JDO数据源的事务处理支持，对JPA和JTA等数据源的事务处理支持等。这一系列的事务处理支持，都是通过设计PlatformTransactionManager、AbstractPlatformTransactionManager以及一系列事务处理器(比如Demo中的DataSourceTransactionManager）来实现的，而TransactionInterceptor又持有PlatformTransactionManager对象，通过这样一个成员变量设计，就把这一系列的事务处理的实现与前面提到的TransactionProxyFactoryBean结合起来，从而形成了一个Spring声明式事务处理的设计体系。  

<img src="/img/2018-9-17/SpringTransactionStructure.jpg" width="700" height="700" alt="SpringTransactionStructure" />
<center>图1：Spring事务处理设计体系</center>

下面就开始分析TransactionProxyFactoryBean生成userService代理对象的过程。    

_ _ _
### **TransactionProxyFactoryBean代理对象生成**
```java
<bean id="userService" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <property name="transactionManager">
        <ref bean="transactionManager"/>
    </property>
    <property name="transactionAttributes">
        <props>
            <prop key="*">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
    <property name="target">
        <ref local="exactUserService"/>
    </property>
</bean>

public class ProxyConfig implements Serializable {
}

public abstract class AbstractSingletonProxyFactoryBean extends ProxyConfig
		implements FactoryBean<Object>, BeanClassLoaderAware, InitializingBean {

    private Object target; // 保存被代理的对象
    private transient ClassLoader proxyClassLoader; 

    /**
	 * Set the target object, that is, the bean to be wrapped with a transactional proxy.
	 * <p>The target may be any object, in which case a SingletonTargetSource will
	 * be created. If it is a TargetSource, no wrapper TargetSource is created:
	 * This enables the use of a pooling or prototype TargetSource etc.
	 * @see org.springframework.aop.TargetSource
	 * @see org.springframework.aop.target.SingletonTargetSource
	 * @see org.springframework.aop.target.LazyInitTargetSource
	 * @see org.springframework.aop.target.PrototypeTargetSource
	 * @see org.springframework.aop.target.CommonsPoolTargetSource
	 */
	public void setTarget(Object target) {
		this.target = target;
	}
 
    // BeanClassLoaderAware回调
    public void setBeanClassLoader(ClassLoader classLoader) {
		if (this.proxyClassLoader == null) {
			this.proxyClassLoader = classLoader;
		}
	}
}

public class TransactionProxyFactoryBean extends AbstractSingletonProxyFactoryBean
		implements BeanFactoryAware {

    private final TransactionInterceptor transactionInterceptor = new TransactionInterceptor();

    /**
	 * Set the transaction manager. This will perform actual
	 * transaction management: This class is just a way of invoking it.
	 * @see TransactionInterceptor#setTransactionManager
	 */
	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionInterceptor.setTransactionManager(transactionManager);
	}

    /**
	 * Set properties with method names as keys and transaction attribute
	 * descriptors (parsed via TransactionAttributeEditor) as values:
	 * e.g. key = "myMethod", value = "PROPAGATION_REQUIRED,readOnly".
	 * <p>Note: Method names are always applied to the target class,
	 * no matter if defined in an interface or the class itself.
	 * <p>Internally, a NameMatchTransactionAttributeSource will be
	 * created from the given properties.
	 * @see #setTransactionAttributeSource
	 * @see TransactionInterceptor#setTransactionAttributes
	 * @see TransactionAttributeEditor
	 * @see NameMatchTransactionAttributeSource
	 * transactionAttributes中key值代表要对target中哪些方法设置参数，比如"myMethod"
	 * value值代表要设置的参数，比如"PROPAGATION_REQUIRED,readOnly"
	 */
	public void setTransactionAttributes(Properties transactionAttributes) {
		this.transactionInterceptor.setTransactionAttributes(transactionAttributes);
	}

    // BeanFactoryAware接口回调
    public void setBeanFactory(BeanFactory beanFactory) {
		this.transactionInterceptor.setBeanFactory(beanFactory);
	}
}
```
可以看到transactionManager与transactionAttributes依赖注入时都被设置到了TransactionProxyFactoryBean.transactionInterceptor对象中，被代理的目标对象target保存在TransactionProxyFactoryBean中。  

在TransactionProxyFactoryBean属性都被设置好之后，接下来进行的就是其对应的InitializingBean的afterPropertiesSet方法调用：  
```java
public abstract class AbstractSingletonProxyFactoryBean extends ProxyConfig
		implements FactoryBean<Object>, BeanClassLoaderAware, InitializingBean {
    // 每个被代理的Service层对象有自己的transactionInterceptor
    private final TransactionInterceptor transactionInterceptor = new TransactionInterceptor();
    private Object proxy; // 每个TransactionProxyFactoryBean对象对应唯一一个proxy(被代理后的对象)

    public void afterPropertiesSet() {
		if (this.target == null) {
			throw new IllegalArgumentException("Property 'target' is required");
		}
		if (this.target instanceof String) {
			throw new IllegalArgumentException("'target' needs to be a bean reference, not a bean name as value");
		}
		if (this.proxyClassLoader == null) {
			this.proxyClassLoader = ClassUtils.getDefaultClassLoader();
		}
        // 准备使用ProxyFactory创建AOP代理对象，proxyFactory与ProxyFactoryBean同级别，只不过是编程式的使用AOP的一种方式
		ProxyFactory proxyFactory = new ProxyFactory();

		if (this.preInterceptors != null) {
			for (Object interceptor : this.preInterceptors) {
				proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(interceptor));
			}
		}

		// Add the main interceptor (typically an Advisor).
        // 调用的是子类TransactionProxyFactoryBean的createMainInterceptor方法，返回的是TransactionAttributeSourceAdvisor
		proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(createMainInterceptor()));

		if (this.postInterceptors != null) {
			for (Object interceptor : this.postInterceptors) {
				proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(interceptor));
			}
		}

		proxyFactory.copyFrom(this);

		TargetSource targetSource = createTargetSource(this.target);
		proxyFactory.setTargetSource(targetSource);

		if (this.proxyInterfaces != null) {
			proxyFactory.setInterfaces(this.proxyInterfaces);
		}
		else if (!isProxyTargetClass()) {
			// Rely on AOP infrastructure to tell us what interfaces to proxy.
			proxyFactory.setInterfaces(
					ClassUtils.getAllInterfacesForClass(targetSource.getTargetClass(), this.proxyClassLoader));
		}
        // 使用JDK动态代理或者CGLib的方式产生代理对象
		this.proxy = proxyFactory.getProxy(this.proxyClassLoader);
	}

    public Object getObject() {
		if (this.proxy == null) {
			throw new FactoryBeanNotInitializedException();
		}
		return this.proxy; // 返回代理后的对象
	}
}

public class TransactionProxyFactoryBean extends AbstractSingletonProxyFactoryBean
		implements BeanFactoryAware {

    /**
	 * Creates an advisor for this FactoryBean's TransactionInterceptor.
	 */
	@Override
	protected Object createMainInterceptor() {
		this.transactionInterceptor.afterPropertiesSet();
        // 在我们的配置文件中并没有设置pointcut属性，所以返回的是TransactionAttributeSourceAdvisor
		if (this.pointcut != null) { 
			return new DefaultPointcutAdvisor(this.pointcut, this.transactionInterceptor);
		}
		else {
			// Rely on default pointcut.
			return new TransactionAttributeSourceAdvisor(this.transactionInterceptor);
		}
	}
}

public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
}

public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {

    // 在配置文件中为TransactionProxyFactoryBean中设置的三个属性
    private PlatformTransactionManager transactionManager;

	private TransactionAttributeSource transactionAttributeSource;

	private BeanFactory beanFactory;

    /**
	 * Check that required properties were set. 完成的工作仅是检测三个属性不为空
	 */
	public void afterPropertiesSet() {
		if (this.transactionManager == null && this.beanFactory == null) {
			throw new IllegalStateException(
					"Setting the property 'transactionManager' or running in a ListableBeanFactory is required");
		}
		if (this.transactionAttributeSource == null) {
			throw new IllegalStateException(
					"Either 'transactionAttributeSource' or 'transactionAttributes' is required: " +
					"If there are no transactional methods, then don't use a transaction aspect.");
		}
	}
}
```
接下来以JDK动态代理方式为例重点分析AbstractSingletonProxyFactoryBean.afterPropertiesSet中JDK动态代理对象的生成过程：  
```java
public abstract class AbstractSingletonProxyFactoryBean extends ProxyConfig
		implements FactoryBean<Object>, BeanClassLoaderAware, InitializingBean {

    private final TransactionInterceptor transactionInterceptor = new TransactionInterceptor();
    private Object proxy; // 每个TransactionProxyFactoryBean对象对应唯一一个proxy(被代理后的对象)

    public void afterPropertiesSet() {
		...
        // 使用JDK动态代理或者CGLib的方式产生代理对象
		this.proxy = proxyFactory.getProxy(this.proxyClassLoader);
	}
}

public class ProxyFactory extends ProxyCreatorSupport {

    public Object getProxy(ClassLoader classLoader) {
        // createAopProxy返回值在这里先选用JdkDynamicAopProxy，之后调用JdkDynamicAopProxy的getProxy方法
		return createAopProxy().getProxy(classLoader);
	}
}

public class ProxyCreatorSupport extends AdvisedSupport {

    private AopProxyFactory aopProxyFactory;

    protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
        // this在这里是ProxyFactory的实例对象
		return getAopProxyFactory().createAopProxy(this);
	}

    public ProxyCreatorSupport() {
		this.aopProxyFactory = new DefaultAopProxyFactory();
	}

    public AopProxyFactory getAopProxyFactory() {
		return this.aopProxyFactory;
	}
}

public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

    // 参数config在这里是ProxyFactory的实例对象
    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface()) {
				return new JdkDynamicAopProxy(config);
			}
			return CglibProxyFactory.createCglibProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}

final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

    /** Config used to configure this proxy */
	private final AdvisedSupport advised;  // advised在这里是ProxyFactory的实例对象

    public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}
    
    // 返回AbstractSingletonProxyFactoryBean中使用的实际代理对象
    public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
}
```
代理对象生成之后，Spring为实现事务处理而设计的TransactionInterceptor已经设置到ProxyFactory生成的AOP代理对象中去了，这个TransactionInterceptor对象被注入到TransactionAttributeSourceAdvisor对象中，被代理对象持有，之后会作为AOP Advice的拦截器来实现它的功能。  

TransactionProxyFactoryBean生成的代理对象相关UML图如下：  
<img src="/img/2018-9-17/TransactionProxyFactoryBeanProxyUML.png" width="700" height="700" alt="TransactionProxyFactoryBeanProxyUML" />
<center>图2：TransactionProxyFactoryBean代理对象相关UML图</center>

在AbstractSingletonProxyFactoryBean.afterPropertiesSet方法执行完之后,可以看到在Spring声明式事务处理实现中的一些重要的类已经悄然登场，比如TransactionAttributeSourceAdvisor和TransactionInterceptor，正是这些类通过AOP封装了Spring对事务处理的基本实现，接下来先来分析TransactionAttributeSourceAdvisor，来了解具体的事务配置属性是如何读入的。  

_ _ _
### **TransactionAttributeSourceAdvisor事务配置属性读入**
TransactionAttributeSourceAdvisor既然是一个Advisor，其中必然就含有Advice和相应的PointCut：
```java
public class TransactionAttributeSourceAdvisor extends AbstractPointcutAdvisor {

    private TransactionInterceptor transactionInterceptor; // advice实现

    private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
		@Override
		protected TransactionAttributeSource getTransactionAttributeSource() {
			return (transactionInterceptor != null ? transactionInterceptor.getTransactionAttributeSource() : null);
		}
	}; // pointCut实现    

    public Pointcut getPointcut() {
		return this.pointcut;
	}

    public TransactionAttributeSourceAdvisor(TransactionInterceptor interceptor) {
		setTransactionInterceptor(interceptor);
	}

    public void setTransactionInterceptor(TransactionInterceptor interceptor) {
		this.transactionInterceptor = interceptor;
	}
}

abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {

	public boolean matches(Method method, Class targetClass) {
		TransactionAttributeSource tas = getTransactionAttributeSource();
        // 配置中没有设置transactionAttributes或者当前方法符合我们配置的开启事务的条件，都要开启事务控制
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}

	@Override
	public boolean equals(Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof TransactionAttributeSourcePointcut)) {
			return false;
		}
		TransactionAttributeSourcePointcut otherPc = (TransactionAttributeSourcePointcut) other;
		return ObjectUtils.nullSafeEquals(getTransactionAttributeSource(), otherPc.getTransactionAttributeSource());
	}

	@Override
	public int hashCode() {
		return TransactionAttributeSourcePointcut.class.hashCode();
	}

	@Override
	public String toString() {
		return getClass().getName() + ": " + getTransactionAttributeSource();
	}


	/**
	 * Obtain the underlying TransactionAttributeSource (may be {@code null}).
	 * To be implemented by subclasses.
	 */
	protected abstract TransactionAttributeSource getTransactionAttributeSource();

}
```

#### **TransactionAttributeSourceAdvisor中PointCut介绍**
在TransactionAttributeSourcePointcut的matches判断过程中，会用到transactionAttributeSource对象，根据其getTransactionAttribute方法与切点是否匹配，来决定是否开启事务拦截控制。这个对象是在对TransactionInterceptor进行依赖注入时就配置好的：    
```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
}

public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {

    private TransactionAttributeSource transactionAttributeSource;

    // 设置我们配置的事务属性
    public void setTransactionAttributes(Properties transactionAttributes) {
		NameMatchTransactionAttributeSource tas = new NameMatchTransactionAttributeSource();
		tas.setProperties(transactionAttributes);
		this.transactionAttributeSource = tas;
	}
}

public class NameMatchTransactionAttributeSource implements TransactionAttributeSource, Serializable {

    // key值为方法名称，可能含有通配符，value值为对应的属性
    private Map<String, TransactionAttribute> nameMap = new HashMap<String, TransactionAttribute>();

    public void setProperties(Properties transactionAttributes) {
		TransactionAttributeEditor tae = new TransactionAttributeEditor();
		Enumeration propNames = transactionAttributes.propertyNames();
		while (propNames.hasMoreElements()) {
			String methodName = (String) propNames.nextElement();
			String value = transactionAttributes.getProperty(methodName);
			tae.setAsText(value);
			TransactionAttribute attr = (TransactionAttribute) tae.getValue();
			addTransactionalMethod(methodName, attr);
		}
	}

    public void addTransactionalMethod(String methodName, TransactionAttribute attr) {
		if (logger.isDebugEnabled()) {
			logger.debug("Adding transactional method [" + methodName + "] with attribute [" + attr + "]");
		}
        // 将实际事务属性都放置在nameMap中
		this.nameMap.put(methodName, attr);
	}
}
```
在以上的代码实现中，可以看到NameMatchTransactionAttributeSource作为TransactionAttributeSource的具体实现，是实际完成事务处理属性读入和匹配的地方，下面就来看下NameMatchTransactionAttributeSource是怎样实现事务处理属性的读入和匹配的：  
```java
public class NameMatchTransactionAttributeSource implements TransactionAttributeSource, Serializable {

    // 对调用的方法进行判断，判断它是否是事务方法，如果是事务方法，那么取出相应的事务配置属性
    public TransactionAttribute getTransactionAttribute(Method method, Class<?> targetClass) {
		// look for direct name match
		String methodName = method.getName();
		TransactionAttribute attr = this.nameMap.get(methodName);
        
        // 这里还要再匹配一次的原因是放入nameMap中的方法名字中可能含有通配符
		if (attr == null) {
			// Look for most specific name match.
			String bestNameMatch = null;
			for (String mappedName : this.nameMap.keySet()) {
				if (isMatch(methodName, mappedName) &&
						(bestNameMatch == null || bestNameMatch.length() <= mappedName.length())) {
					attr = this.nameMap.get(mappedName);
					bestNameMatch = mappedName;
				}
			}
		}

		return attr;
	}

    protected boolean isMatch(String methodName, String mappedName) {
		return PatternMatchUtils.simpleMatch(mappedName, methodName);
	}
}
```
可以看到实现逻辑很简单，首先使用方法名字在nameMap中查找，找不到的话再使用PatternMatchUtils.simpleMatch进行正则匹配，如果匹配成功的话，就将相应的事务处理属性从nameMap中取出，从而触发事务处理器的拦截。  

#### **TransactionAttributeSourceAdvisor中Advice介绍**
TransactionAttributeSourceAdvisor中Advice的实现实际就是TransactionInterceptor。对于TransactionInterceptor如何进行事务控制的分析，我们从TransactionProxyFactoryBean.getObject()方法开始分析。  

TransactionProxyFactoryBean是一个FactoryBean，通过IOC容器获取的实际对象是通过getObject返回的：  
```java
public class TransactionProxyFactoryBean extends AbstractSingletonProxyFactoryBean
		implements BeanFactoryAware {    
}

public abstract class AbstractSingletonProxyFactoryBean extends ProxyConfig
		implements FactoryBean<Object>, BeanClassLoaderAware, InitializingBean {

    private Object proxy;

    public Object getObject() {
		if (this.proxy == null) {
			throw new FactoryBeanNotInitializedException();
		}
		return this.proxy;
	}
}
```
其中代理对象是在AbstractSingletonProxyFactoryBean.afterPropertiesSet方法中生成的，之前已经介绍过。proxy中持有的invocationHandler是JdkDynamicAopProxy类对象，所以对代理对象方法的调用都会被代理到JdkDynamicAopProxy.invoke()方法上：  
```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

    /** 
     * Config used to configure this proxy
     * 这里是一个 ProxyFactory 类的实例
     */
	private final AdvisedSupport advised;

    /**
	 * Implementation of {@code InvocationHandler.invoke}.
	 * <p>Callers will see exactly the exception thrown by the target,
	 * unless a hook method throws an exception.
	 * 动态代理生成的类会调用这个类的invoke方法对实际对象的方法进行拦截，之后再调用实际对象的方法 
	 */
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
                // 如果目标对象没有实现Object类的基本方法equals
				return equals(args[0]);
			}
			if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
                // 如果目标对象没有实现Object类的基本方法hashCode
				return hashCode();
			}
			if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
                // 根据代理对象的配置来调用服务 何时会出现这种情况？？？  
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// May be null. Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
            // 获取被代理的目标对象
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}

			// Get the interception chain for this method.
            // 获得定义好的拦截器链
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// Check whether we have any advice. If we don't, we can fallback on direct
			// reflective invocation of the target, and avoid creating a MethodInvocation.
            // 如果没有设定拦截器，那么就直接调用target的对应方法
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
			}
			else {
				// We need to create a method invocation...
                // 如果有拦截器的设定，那么需要调用拦截器之后才调用目标对象的相应方法
                // 通过构造一个ReflectiveMethodInvocation来实现，下面会介绍
                // 这个ReflectiveMethodInvocation类的具体实现
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				
                // Proceed to the joinpoint through the interceptor chain.
                // 沿着拦截器链继续前进
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
    
}

public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {

    protected final Object proxy;

	protected final Object target;

	protected final Method method;

	protected Object[] arguments;

	private final Class targetClass;

    /**
	 * Lazily initialized map of user-specific attributes for this invocation.
	 */
	private Map<String, Object> userAttributes;

    /**
	 * List of MethodInterceptor and InterceptorAndDynamicMethodMatcher
	 * that need dynamic checks.
	 */
	protected final List interceptorsAndDynamicMethodMatchers;

    protected ReflectiveMethodInvocation(
			Object proxy, Object target, Method method, Object[] arguments,
			Class targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {

		this.proxy = proxy;
		this.target = target;
		this.targetClass = targetClass;
		this.method = BridgeMethodResolver.findBridgedMethod(method);
		this.arguments = arguments;
		this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
	}

    public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
        // 从索引为-1的拦截器开始调用，并按序递增
        // 如果拦截器链中的拦截器迭代调用完成，这里开始调用target的函数，这个函数是通过反射机制完成的
        // 具体实现在AopUtils.invokeJoinpointUsingReflection中
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

        // 这里沿着定义好的interceptorOrInterceptionAdvice链进行处理
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
            // 这里对拦截器进行动态匹配的判断，这里就是触发对前面分析的pointCut进行匹配的地方，如果和定义的pointCut匹配
            // 那么这个advice将会得到执行
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
                // this引用的传入使得在interceptor.invoke方法中可以继续调用proceed方法
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
                // 如果不匹配，那么proceed会被递归调用，直到所有的拦截器都被运行过为止
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
            // 如果是一个interceptor，直接调用这个interceptor对应的方法就好,TransactionInterceptor就属于这种情况
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
    
    // target是被代理的实际对象，这里调用的实际对象的方法，在拦截器执行完成之后调用
    protected Object invokeJoinpoint() throws Throwable {
		return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
	}
}
```  
上面代码分析了拦截器链的调用过程，拦截器的获得是在JdkDynamicAopProxy.invoke方法中这句代码得来的：
```java
// 获得定义好的拦截器链
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
```
下面就来看下这句代码究竟做了什么：
```java
public class AdvisedSupport extends ProxyConfig implements Advised {

    /**
	 * Determine a list of {@link org.aopalliance.intercept.MethodInterceptor} objects
	 * for the given method, based on this configuration.
	 * 在当前ProxyFactoryBean中获取被代理对象指定方法名称的拦截器链
	 * @param method the proxied method
	 * @param targetClass the target class
	 * @return List of MethodInterceptors (may also include InterceptorAndDynamicMethodMatchers)
	 */
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class targetClass) {
		MethodCacheKey cacheKey = new MethodCacheKey(method);
        // 先从缓存中找
		List<Object> cached = this.methodCache.get(cacheKey);
        // 没有找到
		if (cached == null) {
            // 根据method参数获取与其对应的拦截器链的工作是由advisorChainFactory来完成的
            // 这里使用的是DefaultAdvisorChainFactory
            // 这里的this可以是一个ProxyFactoryBean类的实现
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}
}

public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {

    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, Class targetClass) {

		// This is somewhat tricky... we have to process introductions first,
		// but we need to preserve order in the ultimate list.
        // advisor链已经在config中持有了，这里可以直接使用
		List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
		boolean hasIntroductions = hasMatchingIntroductions(config, targetClass);
        // 拦截器链是通过AdvisorAdapterRegistry来加入的，单例模式，默认是DefaultAdvisorAdapterRegistry
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

        // 从当前ProxyFactoryBean中获取到的所有Advisor中进行选择
		for (Advisor advisor : config.getAdvisors()) {
            // 根据advisor的不同类型进行区分
			if (advisor instanceof PointcutAdvisor) {  // TransactionAttributeSourceAdvisor就属于PointcutAdvisor
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
                // 切点增强的类是否是被代理的类
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {   
                    // 一个Advisor封装的Advice可能既是一个BeforeAdvice，又是一个AfterAdvice，所以可能对应多个MethodInterceptor
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);

                    // 这里的MethodMatcher对象实质上就是TransactionAttributeSourcePointcut的对象
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                    // 使用MethodMatcher的matches方法进行匹配，实际这里就是从我们设置的事务属性中进行筛选匹配
					if (MethodMatchers.matches(mm, method, targetClass, hasIntroductions)) {
						if (mm.isRuntime()) { // 运行时匹配
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(targetClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
        // 这里返回的实际就是TransactionInterceptor
		return interceptorList;
	}
}
```  
通过上面的过程分析可知我们获取的拦截目标方法的拦截器其实就是TransactionInterceptor，下面来看看其是如何进行拦截的：  

_ _ _
### **TransactionInterceptor具体拦截过程**
```java
// 由于TransactionInterceptor属于MethodInterceptor实现，ReflectiveMethodInvocation.proceed中会使用其invoke方法
// 对目标对象进行拦截
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {

    // 这里的传入的MethodInvocation参数是ReflectiveMethodInvocation类实现
    public Object invoke(final MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
        // 被代理对象的类对象
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
        // 调用父类invokeWithinTransaction方法
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
		});
	}
}

public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {

    /**
	 * General delegate for around-advice-based subclasses, delegating to several other template
	 * methods on this class. Able to handle {@link CallbackPreferringPlatformTransactionManager}
	 * as well as regular {@link PlatformTransactionManager} implementations.
	 * @param method the Method being invoked
	 * @param targetClass the target class that we're invoking the method on
	 * @param invocation the callback to use for proceeding with the target invocation
	 * @return the return value of the method, if any
	 * @throws Throwable propagated from the target invocation
	 */
	protected Object invokeWithinTransaction(Method method, Class targetClass, final InvocationCallback invocation)
			throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
        // 根据配置的事务属性决定使用哪个TransactionManager，这里使用的是DataSourceTransactionManager
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass);

        // DataSourceTransactionManager不属于CallbackPreferringPlatformTransactionManager，不需要通过回调的方式来使用
		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
            // 创建事务，并将创建事务的过程中得到的信息放到TransactionInfo中去
            // TransactionInfo是保存当前事务状态的对象
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
                // 其实是环绕通知的一种实现，使处理器继续沿着拦截器链调用，直到调用最后的目标方法，
                // 之后再根据执行结果提交或回滚事务
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
                // 方法调用过程中出现了异常，需要依据具体情况进行回滚或提交操作
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
                // 这里把与线程绑定的TransactionInfo设置为oldTransactionInfo
				cleanupTransactionInfo(txInfo);
			}
            // 通过事务处理器提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
        // 对CallbackPreferringPlatformTransactionManager来说，需要通过回调函数的方式实现事务的创建和提交
		else {
			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
						new TransactionCallback<Object>() {
							public Object doInTransaction(TransactionStatus status) {
								TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
								try {
									return invocation.proceedWithInvocation();
								}
								catch (Throwable ex) {
									if (txAttr.rollbackOn(ex)) {
										// A RuntimeException: will lead to a rollback.
										if (ex instanceof RuntimeException) {
											throw (RuntimeException) ex;
										}
										else {
											throw new ThrowableHolderException(ex);
										}
									}
									else {
										// A normal return value: will lead to a commit.
										return new ThrowableHolder(ex);
									}
								}
								finally {
									cleanupTransactionInfo(txInfo);
								}
							}
						});

				// Check result: It might indicate a Throwable to rethrow.
				if (result instanceof ThrowableHolder) {
					throw ((ThrowableHolder) result).getThrowable();
				}
				else {
					return result;
				}
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
		}
	}
}
```
由上面的分析过程可知，**Spring事务控制的核心就是TransactionAspectSupport.invokeWithinTransaction中使用的环绕式拦截过程；**DataSourceTransactionManager作为PlatformTransactionManager实现时，通过createTransactionIfNecessary来开启事务，之后调用实际方法，最后commitTransactionAfterReturning提交事务，**下面就以对TransactionAspectSupport.invokeWithinTransaction方法分析作为主线**，先来分析createTransactionIfNecessary()方法。  

#### **事务开启——createTransactionIfNecessary(...)方法**
**createTransactionIfNecessary(...)方法实现分析**
```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
    
    /**
	 * Create a transaction if necessary based on the given TransactionAttribute.
	 * <p>Allows callers to perform custom TransactionAttribute lookups through
	 * the TransactionAttributeSource.
	 * @param txAttr the TransactionAttribute (may be {@code null})
	 * @param joinpointIdentification the fully qualified method name
	 * (used for monitoring and logging purposes)
	 * @return a TransactionInfo object, whether or not a transaction was created.
	 * The {@code hasTransaction()} method on TransactionInfo can be used to
	 * tell if there was a transaction created.
	 * @see #getTransactionAttributeSource()
	 */
	@SuppressWarnings("serial")
	protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
        // 如果没有指定名字，使用方法特征来作为事务名
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		...
	}
}

// new DelegatingTransactionAttribute(txAttr)的过程
public abstract class DelegatingTransactionAttribute extends DelegatingTransactionDefinition
		implements TransactionAttribute, Serializable {

    private final TransactionAttribute targetAttribute;

    public DelegatingTransactionAttribute(TransactionAttribute targetAttribute) {
		super(targetAttribute);
		this.targetAttribute = targetAttribute;
	}
}

public abstract class DelegatingTransactionDefinition implements TransactionDefinition, Serializable {

    private final TransactionDefinition targetDefinition;

    public DelegatingTransactionDefinition(TransactionDefinition targetDefinition) {
		Assert.notNull(targetDefinition, "Target definition must not be null");
		this.targetDefinition = targetDefinition;
	}
}

public interface TransactionAttribute extends TransactionDefinition {}
```
上面分析了new DelegatingTransactionAttribute(txAttr)的实现过程，接下来分析createTransactionIfNecessary方法中tm.getTransaction实现:  
```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
    
    /**
	 * Create a transaction if necessary based on the given TransactionAttribute.
	 * <p>Allows callers to perform custom TransactionAttribute lookups through
	 * the TransactionAttributeSource.
	 * @param txAttr the TransactionAttribute (may be {@code null})
	 * @param joinpointIdentification the fully qualified method name
	 * (used for monitoring and logging purposes)
	 * @return a TransactionInfo object, whether or not a transaction was created.
	 * The {@code hasTransaction()} method on TransactionInfo can be used to
	 * tell if there was a transaction created.
	 * @see #getTransactionAttributeSource()
	 * 一个线程+一个Service单例对象——>唯一对应一个TransactionInfo对象——>唯一对应一个TransactionStatus对象
	 * 这样设置TransactionStatus的对应关系的原因可能是每个Service对应的事务属性配置可能是不同的，比如ServiceA配置事务传播级别
	 * 为REQUIRED，而ServiceB配置为REQUIRED_NEW，所以每个Service都需要一个TransactionStatus对象来记录事务的属性
	 */
	@SuppressWarnings("serial")
	protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
        // 如果没有指定名字，使用方法特征来作为事务名
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
                // 这里使用了定义好的事务方法的配置信息，事务创建由事务处理器来完成
                // 同时返回TransactionStatus来记录当前的事务状态，比如当前事务是否是新创建的事务等等
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
        // TransactionInfo中将PlatformTransactionManager，TransactionAttribute，TransactionStatus
        // 组合到了一起，为其提供了相应的get、set方法
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
}

// 分析DataSourceTransactionManager.getTransaction实现
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {

    public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
        // 取得Transaction对象，由具体的事务处理器实现，比如DataSourceTransactionManager，
        // 可以是DataSourceTransactionManager.DataSourceTransactionObject类的对象，
        // 其主要作用是通过它可以获得与当前线程绑定的ConnectionHolder，其中含有底层数据库连接Connection
		Object transaction = doGetTransaction();

		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			// Use defaults if no transaction definition given.
            // 如果并没有手动配置事务属性，则使用默认的，比如默认事务传播行为是PROPAGATION_REQUIRED
			definition = new DefaultTransactionDefinition();
		}

        // 检查当前线程中是否已经存在事务，如果存在的话，那么需要根据在事务属性中定义的事务传播属性配置来处理事务的产生
        // DataSourceTransactionManager中isExistingTransaction方法的实现是检测transaction中ConnectionHolder是否为空
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
            // 这里对当前线程中已经有事务存在的情况进行处理，结果封装在TransactionStatus中
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}
        // 检查对于要开启的新事务的事务配置是否合理
		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

        // 当前没有事务存在，这时需要根据事务属性设置来创建事务
        // 这里可以看到对事务传播属性设置的处理，比如mandatory、required、required_new、nested等
		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
                // 默认情况下getTransactionSynchronization() = SYNCHRONIZATION_ALWAYS，所以newSynchronization为true
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
                
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
                // 创建事务，为其获取底层数据库连接，在TransactionSynchronizationManager中绑定dataSource与connectionHolder之间关系；
                // 关闭事务自动提交，也就是开启事务，并设置事务相关属性：比如事务隔离级别；
				doBegin(transaction, definition);
                // 如果是新事务的话在TransactionSynchronizationManager中绑定事务隔离级别等属性
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException ex) {
				resume(null, suspendedResources);
				throw ex;
			}
			catch (Error err) {
				resume(null, suspendedResources);
				throw err;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}

    protected DefaultTransactionStatus newTransactionStatus(
			TransactionDefinition definition, Object transaction, boolean newTransaction,
			boolean newSynchronization, boolean debug, Object suspendedResources) {

		boolean actualNewSynchronization = newSynchronization &&
				!TransactionSynchronizationManager.isSynchronizationActive();
		return new DefaultTransactionStatus(
				transaction, newTransaction, actualNewSynchronization,
				definition.isReadOnly(), debug, suspendedResources);
	}

    protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
        // 判断是不是新事务，是的话需要把事务属性存放到当前线程中
		if (status.isNewSynchronization()) {
			TransactionSynchronizationManager.setActualTransactionActive(status.hasTransaction());
			TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
					(definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) ?
							definition.getIsolationLevel() : null);
			TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
			TransactionSynchronizationManager.setCurrentTransactionName(definition.getName());
			TransactionSynchronizationManager.initSynchronization();
		}
	}
}

public class DataSourceTransactionManager extends AbstractPlatformTransactionManager
		implements ResourceTransactionManager, InitializingBean {

    @Override
	protected Object doGetTransaction() {
        // 一个线程+一个service对象对应一个DataSourceTransactionObject对象
		DataSourceTransactionObject txObject = new DataSourceTransactionObject();
		txObject.setSavepointAllowed(isNestedTransactionAllowed());
        // 此时TransactionSynchronizationManager.getResource返回值为null，
        // 因为是初次开启事务，还没有为线程绑定对应的Connection
		ConnectionHolder conHolder =
				(ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
		txObject.setConnectionHolder(conHolder, false);
		return txObject;
	}

    /**
	 * DataSource transaction object, representing a ConnectionHolder.
	 * Used as transaction object by DataSourceTransactionManager.
	 */
	private static class DataSourceTransactionObject extends JdbcTransactionObjectSupport {

		private boolean newConnectionHolder;

		private boolean mustRestoreAutoCommit;

		public void setConnectionHolder(ConnectionHolder connectionHolder, boolean newConnectionHolder) {
			super.setConnectionHolder(connectionHolder);
			this.newConnectionHolder = newConnectionHolder;
		}

		public boolean isNewConnectionHolder() {
			return this.newConnectionHolder;
		}

		public void setMustRestoreAutoCommit(boolean mustRestoreAutoCommit) {
			this.mustRestoreAutoCommit = mustRestoreAutoCommit;
		}

		public boolean isMustRestoreAutoCommit() {
			return this.mustRestoreAutoCommit;
		}

		public void setRollbackOnly() {
			getConnectionHolder().setRollbackOnly();
		}

		public boolean isRollbackOnly() {
			return getConnectionHolder().isRollbackOnly();
		}
	}

    /**
	 * This implementation sets the isolation level but ignores the timeout.
	 * 通过dataSource获取数据库连接，开启事务，设置事务相关属性
	 */
	@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;

		try {
			if (txObject.getConnectionHolder() == null ||
					txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
				Connection newCon = this.dataSource.getConnection();
				if (logger.isDebugEnabled()) {
					logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
				}
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}

			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();

			Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
			txObject.setPreviousIsolationLevel(previousIsolationLevel);

			// Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
			// so we don't want to do it unnecessarily (for example if we've explicitly
			// configured the connection pool to set it already).
			if (con.getAutoCommit()) {
				txObject.setMustRestoreAutoCommit(true);
				if (logger.isDebugEnabled()) {
					logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
				}
                // 开启事务
				con.setAutoCommit(false);
			}
			txObject.getConnectionHolder().setTransactionActive(true);

			int timeout = determineTimeout(definition);
			if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
				txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
			}

			// Bind the session holder to the thread.
			if (txObject.isNewConnectionHolder()) {
                // 在TransactionSynchronizationManager中绑定线程对应的ConnectionHolder
				TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
			}
		}

		catch (Throwable ex) {
			if (txObject.isNewConnectionHolder()) {
				DataSourceUtils.releaseConnection(con, this.dataSource);
				txObject.setConnectionHolder(null, false);
			}
			throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
		}
	}
}

/**
 * TransactionSynchronizationManager维护一系列的ThreadLocal变量来保持事务属性
 * 比如事务隔离级别，是否有活跃的事务等
 */
public abstract class TransactionSynchronizationManager {

    // ThreadLocal中key值为使用的数据源，value值是当前线程绑定的ConnectionHolder对象
    // 关系绑定是在DataSourceTransactionManager.doBegin方法中完成的
    private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<Map<Object, Object>>("Transactional resources");

    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<Set<TransactionSynchronization>>("Transaction synchronizations");

	private static final ThreadLocal<String> currentTransactionName =
			new NamedThreadLocal<String>("Current transaction name");

	private static final ThreadLocal<Boolean> currentTransactionReadOnly =
			new NamedThreadLocal<Boolean>("Current transaction read-only status");

	private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
			new NamedThreadLocal<Integer>("Current transaction isolation level");

	private static final ThreadLocal<Boolean> actualTransactionActive =
			new NamedThreadLocal<Boolean>("Actual transaction active");

    // 这时key值为org.apache.tomcat.jdbc.pool.DataSource
    public static Object getResource(Object key) {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        // 初次调用时，由于没有与线程绑定的ConnectionHolder，所以doGetResource返回值为null
		Object value = doGetResource(actualKey);
		if (value != null && logger.isTraceEnabled()) {
			logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
					Thread.currentThread().getName() + "]");
		}
		return value;
	}

    private static Object doGetResource(Object actualKey) {
		Map<Object, Object> map = resources.get();
		if (map == null) {
			return null;
		}
		Object value = map.get(actualKey);
		// Transparently remove ResourceHolder that was marked as void...
		if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
			map.remove(actualKey);
			// Remove entire ThreadLocal if empty...
			if (map.isEmpty()) {
				resources.remove();
			}
			value = null;
		}
		return value;
	}
}
```

#### **事务提交——commitTransactionAfterReturning(...)方法**
```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {

    /**
	 * General delegate for around-advice-based subclasses, delegating to several other template
	 * methods on this class. Able to handle {@link CallbackPreferringPlatformTransactionManager}
	 * as well as regular {@link PlatformTransactionManager} implementations.
	 * @param method the Method being invoked
	 * @param targetClass the target class that we're invoking the method on
	 * @param invocation the callback to use for proceeding with the target invocation
	 * @return the return value of the method, if any
	 * @throws Throwable propagated from the target invocation
	 */
	protected Object invokeWithinTransaction(Method method, Class targetClass, final InvocationCallback invocation)
			throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
        // 根据配置的事务属性决定使用哪个TransactionManager，这里使用的是DataSourceTransactionManager
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass);

        // DataSourceTransactionManager不属于CallbackPreferringPlatformTransactionManager，不需要通过回调的方式来使用
		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
            // 创建事务，并将创建事务的过程中得到的信息放到TransactionInfo中去
            // TransactionInfo是保存当前事务状态的对象
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
                // 其实是环绕通知的一种实现，使处理器继续沿着拦截器链调用，直到调用最后的目标方法，
                // 之后再根据执行结果提交或回滚事务
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
                // 方法调用过程中出现了异常，需要依据具体情况进行回滚或提交操作
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
                // 这里把与线程绑定的TransactionInfo设置为oldTransactionInfo
				cleanupTransactionInfo(txInfo);
			}
            // 通过事务处理器提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
        ...
	}

    /**
	 * Execute after successful completion of call, but not after an exception was handled.
	 * Do nothing if we didn't create a transaction.
	 * @param txInfo information about the current transaction
	 */
	protected void commitTransactionAfterReturning(TransactionInfo txInfo) {
		if (txInfo != null && txInfo.hasTransaction()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
			}
            // 以DataSourceTransactionManager为例进行分析
			txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
		}
	}
}

public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {

    /**
	 * This implementation of commit handles participating in existing
	 * transactions and programmatic rollback requests.
	 * Delegates to {@code isRollbackOnly}, {@code doCommit}
	 * and {@code rollback}.
	 * @see org.springframework.transaction.TransactionStatus#isRollbackOnly()
	 * @see #doCommit
	 * @see #rollback
	 */
	public final void commit(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		if (defStatus.isLocalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Transactional code has requested rollback");
			}
			processRollback(defStatus);
			return;
		}
		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
			}
            // 处理回滚的情况
			processRollback(defStatus);
			// Throw UnexpectedRollbackException only at outermost transaction boundary
			// or if explicitly asked to.
			if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
			return;
		}
        // 处理提交
		processCommit(defStatus);
	}

    /**
	 * Process an actual commit.
	 * Rollback-only flags have already been checked and applied.
	 * @param status object representing the transaction
	 * @throws TransactionException in case of commit failure
	 */
	private void processCommit(DefaultTransactionStatus status) throws TransactionException {
		try {
			boolean beforeCompletionInvoked = false;
			try {
                // 事务提交的准备工作由具体的事务处理器来完成
				prepareForCommit(status);
				triggerBeforeCommit(status);
				triggerBeforeCompletion(status);
				beforeCompletionInvoked = true;
				boolean globalRollbackOnly = false;
				if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
					globalRollbackOnly = status.isGlobalRollbackOnly();
				}
				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Releasing transaction savepoint");
					}
					status.releaseHeldSavepoint();
				}
                // 根据当前线程中所保存的事务状态进行处理，如果当前的事务是一个新事务，使用具体事务处理器完成提交；
                // 如果不是新事务，则不提交
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction commit");
					}
                    // 当前就是当前status对应的Service的方法中开启的事务，就要使用这个status中的connection关闭事务
					doCommit(status);
				}
				// Throw UnexpectedRollbackException if we have a global rollback-only
				// marker but still didn't get a corresponding exception from commit.
				if (globalRollbackOnly) {
					throw new UnexpectedRollbackException(
							"Transaction silently rolled back because it has been marked as rollback-only");
				}
			}
			catch (UnexpectedRollbackException ex) {
				// can only be caused by doCommit
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
				throw ex;
			}
			catch (TransactionException ex) {
				// can only be caused by doCommit
				if (isRollbackOnCommitFailure()) {
					doRollbackOnCommitException(status, ex);
				}
				else {
					triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				}
				throw ex;
			}
			catch (RuntimeException ex) {
				if (!beforeCompletionInvoked) {
					triggerBeforeCompletion(status);
				}
				doRollbackOnCommitException(status, ex);
				throw ex;
			}
			catch (Error err) {
				if (!beforeCompletionInvoked) {
					triggerBeforeCompletion(status);
				}
				doRollbackOnCommitException(status, err);
				throw err;
			}

			// Trigger afterCommit callbacks, with an exception thrown there
			// propagated to callers but the transaction still considered as committed.
			try {
				triggerAfterCommit(status);
			}
			finally {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
			}

		}
		finally {
			cleanupAfterCompletion(status);
		}
	}
}

public class DataSourceTransactionManager extends AbstractPlatformTransactionManager
		implements ResourceTransactionManager, InitializingBean {

    // 进行具体的事务提交过程
    @Override
	protected void doCommit(DefaultTransactionStatus status) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
        // 获取数据库连接
		Connection con = txObject.getConnectionHolder().getConnection();
		if (status.isDebug()) {
			logger.debug("Committing JDBC transaction on Connection [" + con + "]");
		}
		try {
            // 事务提交！！！
			con.commit();
		}
		catch (SQLException ex) {
			throw new TransactionSystemException("Could not commit JDBC transaction", ex);
		}
	}
}
```
到这里为止，Spring事务控制系统实现原理就大致分析完成了，在上面的分析过程中我们可以看到Spring事务控制系统的实现核心是AOP思想，通过TransactionProxyFactoryBean中获取我们使用的Service层的对象，这里获取的对象实际是使用ProxyFactory对象生成的代理对象，我们调用代理对象的方法时就会先使用TransactionInterceptor拦截器进行拦截，通过这个拦截器中的环绕式拦截实现事务的开启与提交。在执行事务的过程中，使用TransactionInfo和TransactionStatus对象保存事务的相关信息，比如记录事务的执行情况，与线程的绑定情况等，对这两个对象的使用贯穿整个事务处理的全过程。        

在分析的时候我们使用的是Spring最原始的配置方式，这有助于更容易的理解Spring事务控制系统实现原理。现在的注解方式声明式事务要比这篇博客中使用的配置方式简洁的多，但是事务控制系统的核心实现原理与这里是相同的，只不过是增加了相应的注解处理过程使得事务控制方式更简洁易用而已。    

(完)  


  