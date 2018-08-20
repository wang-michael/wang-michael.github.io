---
layout:     post
title:      Spring学习系列之AOP实现机制分析
date:       2018-8-17
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Spring
---
>本文记录我对Spring AOP实现机制的使用和其实现原理的理解(基于Spring3.2版本)。       

_ _ _
### **前言**
面向切面编程思想(AOP)虽然与设计公共子模块有几分相似，但在传统的公共子模块调用中，除了直接硬调用之外并没有其他的手段，而AOP为处理这一类问题提供了一套完整的理论和灵活多样的实现方法。  

AOP思想可以总结为基础(源程序)+切面(通知程序)+配置(编织)；  

Spring AOP的实现和其他特性的实现一样，除了可以使用Spring本身提供的AOP实现之外，还封装了业界优秀的AOP解决方案AspectJ来供应用使用。在Spring自身对于Aop的实现中，充分利用了**IOC容器Proxy代理对象以及AOP拦截器的功能特性**，通过这些对AOP基本功能的封装机制，为用户提供了AOP的实现框架。    

**由切入点表达式所匹配的连接点的概念是AOP的核心，默认情况下Spring使用AspectJ切入点表达式语言。Spring只支持方法级别的切入点(因为Spring AOP构建在动态代理基础之上)，对于构造器和字段级别的切入点不支持。**  

Spring AOP实现中对于具体的AOP代理对象的生成，根据不同的需要，分别由ProxyFactoryBean、AspectJProxyFactory和ProxyFactory来完成。对于需要使用AspectJ的AOP应用，AspectJProxyFactory起到继承Spring和AspectJ的作用；对于使用Spring AOP的应用，ProxyFactoryBean和ProxyFactory都提供了AOP功能的封装，只是使用ProxyFactoryBean，可以在IOC容器中完成声明式配置，而使用ProxyFactory，则需要编程式的使用Spring AOP的功能。  

下面先来看一个使用Spring AOP的Demo：  
```java
public class TestFileSystemXmlApplicationContext {
//     xml文件中声明方式实现AOP
    public static void main(String[] args) {
        FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext("beans.xml");
        Wait waiter = (Wait) context.getBean("waiter");
        waiter.say();
        context.close();
    }

    // 编程方式实现AOP
//    private static SayHelloBeforemale male ;
//    private static Waiter wait;
//    private static ProxyFactory xy;
//
//    public static void main(String[] args) {
//        //实例化并赋值
//        male = new SayHelloBeforemale();
//        wait = new Waiter();
//        xy = new ProxyFactory();
//        //设置目标对象
//        xy.setTarget(wait);
//        //设置增强类对象
//        xy.addAdvice(male);
//
//        Waiter w = (Waiter)xy.getProxy();
//        w.say();
//    }
}

public interface Wait {
    void say();
}

public class Waiter implements Wait {
    @Override
    public void say() {
        // TODO Auto-generated method stub
        System.out.println("先生");
    }
}

public class SayHelloBeforemale implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("hello");
    }
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">    

    <!--定义advise sayhelloadvice定义了通知包含什么-->
    <bean id="sayhelloadvice" class="mytests.aopTests.SayHelloBeforemale"/>

    <!--waiter定义了在何时(调用proxyInterfaces接口的方法时)为什么对象(target指定，此对象需要实现proxyInterfaces接口)-->
    <!--应用何种通知(interceptorNames中指定)-->
    <!-- 定义advisor-->
    <bean id="waiter" class="org.springframework.aop.framework.ProxyFactoryBean">        
        <property name="target">
            <bean class="mytests.aopTests.Waiter"/>
        </property>
        <property name="interceptorNames">
            <list>
                <value>sayhelloadvice</value>
            </list>
        </property>
    </bean>
</beans>
```
接下来我们就从ProxyFactoryBean开始分析Spring AOP的实现机制。  

_ _ _
### **Spring AOP的实现原理分析**

#### **初始化通知器链**
ProxyFactoryBean是一个FactoryBean，我们获取的代理对象就是从ProxyFactoryBean.getObject()方法返回的：  
```java
public class ProxyFactoryBean extends ProxyCreatorSupport
		implements FactoryBean<Object>, BeanClassLoaderAware, BeanFactoryAware {

    /**
	 * Return a proxy. Invoked when clients obtain beans from this factory bean.
	 * Create an instance of the AOP proxy to be returned by this factory.
	 * The instance will be cached for a singleton, and create on each call to
	 * {@code getObject()} for a proxy.
	 * @return a fresh AOP proxy reflecting the current state of this factory
	 * 
	 */
	public Object getObject() throws BeansException {
        // 如果尚未初始化过通知器链则进行初始化
		initializeAdvisorChain();
        // 这里对singleton和prototype的类型进行区分，生成对应的proxy
		if (isSingleton()) {
			return getSingletonInstance();
		}
		else {
			if (this.targetName == null) {
				logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
						"Enable prototype proxies by setting the 'targetName' property.");
			}
			return newPrototypeInstance();
		}
	}

    /**
	 * Create the advisor (interceptor) chain. Advisors that are sourced
	 * from a BeanFactory will be refreshed each time a new prototype instance
	 * is added. Interceptors added programmatically through the factory API
	 * are unaffected by such changes.
	 */
	private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
		if (this.advisorChainInitialized) {
			return;
		}

		if (!ObjectUtils.isEmpty(this.interceptorNames)) {
			if (this.beanFactory == null) {
				throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) " +
						"- cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
			}

			// Globals can't be last unless we specified a targetSource using the property...
			if (this.interceptorNames[this.interceptorNames.length - 1].endsWith(GLOBAL_SUFFIX) &&
					this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
				throw new AopConfigException("Target required after globals");
			}

			// Materialize interceptor chain from bean names.
            // 添加Advisor链的调用，是通过beans.xml中interceptorNames属性来配置的
			for (String name : this.interceptorNames) {
				if (logger.isTraceEnabled()) {
					logger.trace("Configuring advisor or advice '" + name + "'");
				}
                // 全局Advisor
				if (name.endsWith(GLOBAL_SUFFIX)) {
					if (!(this.beanFactory instanceof ListableBeanFactory)) {
						throw new AopConfigException(
								"Can only use global advisors or interceptors with a ListableBeanFactory");
					}
					addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
							name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
				}
                // beans.xml中手动配置的Advisor
				else {
					// If we get here, we need to add a named interceptor.
					// We must check if it's a singleton or prototype.
                    /**
                     * 程序执行到这里说明需要加入指定名称的拦截器Advice，并且需要检查这个bean是singleton还是
                     * prototype类型
                     */
					Object advice;
					if (this.singleton || this.beanFactory.isSingleton(name)) {
						// Add the real Advisor/Advice to the chain.
						advice = this.beanFactory.getBean(name);
					}
					else {
						// It's a prototype Advice or Advisor: replace with a prototype.
						// Avoid unnecessary creation of prototype bean just for advisor chain initialization.
						advice = new PrototypePlaceholderAdvisor(name);
					}
					addAdvisorOnChainCreation(advice, name);
				}
			}
		}
        // 初始化完成标志
		this.advisorChainInitialized = true;
	}

    /**
	 * Add all global interceptors and pointcuts.
	 * 在所有的global interceptors and pointcut中筛选出名字符合传入的prefix的，加入到通知器链中
	 */
	private void addGlobalAdvisor(ListableBeanFactory beanFactory, String prefix) {
		String[] globalAdvisorNames =
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, Advisor.class);
		String[] globalInterceptorNames =
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(beanFactory, Interceptor.class);
		List<Object> beans = new ArrayList<Object>(globalAdvisorNames.length + globalInterceptorNames.length);
		Map<Object, String> names = new HashMap<Object, String>(beans.size());
		for (String name : globalAdvisorNames) {
			Object bean = beanFactory.getBean(name);
			beans.add(bean);
			names.put(bean, name);
		}
		for (String name : globalInterceptorNames) {
			Object bean = beanFactory.getBean(name);
			beans.add(bean);
			names.put(bean, name);
		}
		OrderComparator.sort(beans);
		for (Object bean : beans) {
			String name = names.get(bean);
			if (name.startsWith(prefix)) {
				addAdvisorOnChainCreation(bean, name);
			}
		}
	}

    /**
	 * Invoked when advice chain is created.
	 * <p>Add the given advice, advisor or object to the interceptor list.
	 * Because of these three possibilities, we can't type the signature
	 * more strongly.
	 * @param next advice, advisor or target object
	 * @param name bean name from which we obtained this object in our owning
	 * bean factory
	 * next：要加入到通知器链中的Advice
	 * name：beans.xml中为Advice设置的名称
	 */
	private void addAdvisorOnChainCreation(Object next, String name) {
		// We need to convert to an Advisor if necessary so that our source reference
		// matches what we find from superclass interceptors.
        // 将一个advice对象通过适配器适配为advisor对象
		Advisor advisor = namedBeanToAdvisor(next);
		if (logger.isTraceEnabled()) {
			logger.trace("Adding advisor with name '" + name + "'");
		}
        // 调用AdvisedSupport.addAdvisor
		addAdvisor(advisor);
	}
    
    /**
	 * Convert the following object sourced from calling getBean() on a name in the
	 * interceptorNames array to an Advisor or TargetSource.
	 */
	private Advisor namedBeanToAdvisor(Object next) {
		try {
            // 调用DefaultAdvisorAdapterRegistry.wrap方法将interceptorNames对应的Advice封装为一个Advisor
			return this.advisorAdapterRegistry.wrap(next);
		}
		catch (UnknownAdviceTypeException ex) {
			// We expected this to be an Advisor or Advice,
			// but it wasn't. This is a configuration error.
			throw new AopConfigException("Unknown advisor type " + next.getClass() +
					"; Can only include Advisor or Advice type beans in interceptorNames chain except for last entry," +
					"which may also be target or TargetSource", ex);
		}
	}       
}

public class AdvisedSupport extends ProxyConfig implements Advised {

    /**
	 * List of Advisors. If an Advice is added, it will be wrapped
	 * in an Advisor before being added to this List.
	 * 保存ProxyFactoryBean中的Advisors
	 */
	private List<Advisor> advisors = new LinkedList<Advisor>();

    public void addAdvisor(Advisor advisor) {
		int pos = this.advisors.size();
		addAdvisor(pos, advisor);
	}

    public void addAdvisor(int pos, Advisor advisor) throws AopConfigException {
		if (advisor instanceof IntroductionAdvisor) {
			validateIntroductionAdvisor((IntroductionAdvisor) advisor);
		}
		addAdvisorInternal(pos, advisor);
	}
   
    // 真正的addAdvisor方法
    private void addAdvisorInternal(int pos, Advisor advisor) throws AopConfigException {
		Assert.notNull(advisor, "Advisor must not be null");
		if (isFrozen()) {
			throw new AopConfigException("Cannot add advisor: Configuration is frozen.");
		}
		if (pos > this.advisors.size()) {
			throw new IllegalArgumentException(
					"Illegal position " + pos + " in advisor list with size " + this.advisors.size());
		}
		this.advisors.add(pos, advisor);
		updateAdvisorArray();
		adviceChanged();
	}
}
```

#### **获取代理对象**
通过上述代码分析我们可以看到在第一次调用ProxyFactoryBean.getObject()方法的时候会为其初始化一个Advisor链，接下来才是获取代理对象的过程：  
```java
public class ProxyFactoryBean extends ProxyCreatorSupport
		implements FactoryBean<Object>, BeanClassLoaderAware, BeanFactoryAware {

    /**
	 * Return a proxy. Invoked when clients obtain beans from this factory bean.
	 * Create an instance of the AOP proxy to be returned by this factory.
	 * The instance will be cached for a singleton, and create on each call to
	 * {@code getObject()} for a proxy.
	 * @return a fresh AOP proxy reflecting the current state of this factory
	 * 
	 */
	public Object getObject() throws BeansException {
        // 如果尚未初始化过通知器链则进行初始化
		initializeAdvisorChain();
        // 这里对singleton和prototype的类型进行区分，生成对应的proxy
		if (isSingleton()) {
			return getSingletonInstance();
		}
		else {
			if (this.targetName == null) {
				logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
						"Enable prototype proxies by setting the 'targetName' property.");
			}
			return newPrototypeInstance();
		}
	}

    /**
	 * Return the singleton instance of this class's proxy object,
	 * lazily creating it if it hasn't been created already.
	 * @return the shared singleton proxy
	 */
	private synchronized Object getSingletonInstance() {
		if (this.singletonInstance == null) {
			this.targetSource = freshTargetSource();
			if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
				// Rely on AOP infrastructure to tell us what interfaces to proxy.
                // 根据AOP框架来判断需要代理的接口
				Class<?> targetClass = getTargetClass();
				if (targetClass == null) {
					throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
				}
                // 设置代理对象要实现的接口
				setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
			}
			// Initialize the shared singleton instance.
			super.setFrozen(this.freezeProxy);
            // 注意这里的方法会使用ProxyFactory来生成需要的Proxy
            // 每次调用都会创建一个新的AopProxy类对象
			this.singletonInstance = getProxy(createAopProxy());
		}
		return this.singletonInstance;
	}
    
    // 通过createAopProxy返回的AopProxy来生成代理对象
    // 这里传入的AopProxy类对象可能是一个JdkDynamicAopProxy类的对象，其内部持有ProxyFactoryBean的引用
    // 每个JdkDynamicAopProxy每次调用getProxy获取的Proxy对象都是新的
    protected Object getProxy(AopProxy aopProxy) {
		return aopProxy.getProxy(this.proxyClassLoader);
	}
}

public class ProxyCreatorSupport extends AdvisedSupport {

    /**
	 * Subclasses should call this to get a new AOP proxy. They should <b>not</b>
	 * create an AOP proxy with {@code this} as an argument.
	 */
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this);
	}

    /**
	 * Return the AopProxyFactory that this ProxyConfig uses.
	 */
	public AopProxyFactory getAopProxyFactory() {
		return this.aopProxyFactory;
	}

    /**
	 * Create a new ProxyCreatorSupport instance.
	 * 默认的aopProxyFactory
	 */
	public ProxyCreatorSupport() {
		this.aopProxyFactory = new DefaultAopProxyFactory();
	}
}

public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
            // 如果targetClass是接口类，使用JDK动态代理
			if (targetClass.isInterface()) {
				return new JdkDynamicAopProxy(config);
			}
            // 非接口类使用CGLIB动态代理
			return CglibProxyFactory.createCglibProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}
```
接下来以JdkDynamicAopProxy为例来介绍其代理对象的生成过程：  
```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

    /** 
     * Config used to configure this proxy
     * 可能是一个 ProxyFactoryBean 类的实例
     */
	private final AdvisedSupport advised;

    public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}

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
总结下上面代码分析的获取代理对象的过程，首先是通过ProxyFactoryBean.getObject()获取，获取的时候先选择获取的对象是单例还是Prototype类型，上面分析的是获取的单例对象的方法getSingletonInstance，返回给我们的是一个代理对象，但是每次调用  getSingletonInstance返回给我们的代理对象都不是单例的，那单例体现在哪里呢？  

原来每个返回的代理对象内部持有的被代理的目标对象都是单例的，可以通过以下这个图更清晰的看一下：    
<img src="/img/2018-8-17/SpringJDKAOPDynamicImpl.png" width="700" height="700" alt="SpringJDKAOPDynamicImpl" />
<center>图1：Spring AOP JDK动态代理单例实现</center>

#### **对目标对象进行拦截增强**
我们调用返回的代理对象的方法，实际都会被代理到此代理对象对应的JdkDynamicAopProxy中的invoke方法，就是在这个方法中完成了一个完整的拦截器链对目标对象的拦截过程，比如获得拦截器链并对拦截器链中的拦截器进行配置，逐个运行拦截器链里的拦截增强，直到最后到目标方法的运行等。下面就来分析这个invoke方法的内部实现：  
```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

    /** 
     * Config used to configure this proxy
     * 可能是一个 ProxyFactoryBean 类的实例
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
            // 如果是一个interceptor，直接调用这个interceptor对应的方法就好
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
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
                // 切点增强的类是否是被代理的类
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {   
                    // 一个Advisor封装的Advice可能既是一个BeforeAdvice，又是一个AfterAdvice，所以可能对应多个MethodInterceptor
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                    // 使用MethodMatcher的matches方法进行匹配
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
		return interceptorList;
	}
}
```  
拦截器是在Advisor与method匹配成功之后通过DefaultAdvisorAdapterRegistry.getInterceptors()中获得的，接下来看下它的实现：  
```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

    private final List<AdvisorAdapter> adapters = new ArrayList<AdvisorAdapter>(3);

	/**
	 * Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
	 * 是我们熟悉的几种Advice的适配器实现，其目标是将Advice对象适配成一个MethodInterceptor对象并提供一些其它功能
	 */
	public DefaultAdvisorAdapterRegistry() {
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}

    // 将一个Advice对象封装为一个Advisor
    public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
		if (adviceObject instanceof Advisor) {
			return (Advisor) adviceObject;
		}
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}
		Advice advice = (Advice) adviceObject;
		if (advice instanceof MethodInterceptor) {
			// So well-known it doesn't even need an adapter.
			return new DefaultPointcutAdvisor(advice);
		}
		for (AdvisorAdapter adapter : this.adapters) {
			// Check that it is supported.
			if (adapter.supportsAdvice(advice)) {
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}
    
    // 一个advisor可能对应多个MethodInterceptor
    public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<MethodInterceptor>(3);
		Advice advice = advisor.getAdvice();
		if (advice instanceof MethodInterceptor) {
			interceptors.add((MethodInterceptor) advice);
		}
        // 遍历在DefaultAdvisorAdapterRegistry中的AdvisorAdapter，
		for (AdvisorAdapter adapter : this.adapters) {
			if (adapter.supportsAdvice(advice)) {
                // 调用适配器中的getInterceptor方法将Advisor转化为MethodInterceptor
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
		return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
	}
}
```
在ReflectiveMethodInvocation.proceed方法中就是调用相应MethodInterceptor.invoke方法来实现的对目标对象的增强。  

**MethodBeforeAdviceInterceptor增强：**
```java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof MethodBeforeAdvice);
	}

	public MethodInterceptor getInterceptor(Advisor advisor) {
		MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
		return new MethodBeforeAdviceInterceptor(advice);
	}
    
}

public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {

	private MethodBeforeAdvice advice;


	/**
	 * Create a new MethodBeforeAdviceInterceptor for the given advice.
	 * @param advice the MethodBeforeAdvice to wrap
	 */
	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	public Object invoke(MethodInvocation mi) throws Throwable {
        // 前置增强，增强之后继续执行ReflectiveMethodInvocation.proceed()进行之后的增强
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
		return mi.proceed();
	}

}
```
**AfterReturningAdviceInterceptor增强(被代理对象方法执行之后，返回值返回之前)：**
```java
class AfterReturningAdviceAdapter implements AdvisorAdapter, Serializable {

	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof AfterReturningAdvice);
	}

	public MethodInterceptor getInterceptor(Advisor advisor) {
		AfterReturningAdvice advice = (AfterReturningAdvice) advisor.getAdvice();
		return new AfterReturningAdviceInterceptor(advice);
	}
}

public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {

	private final AfterReturningAdvice advice;


	/**
	 * Create a new AfterReturningAdviceInterceptor for the given advice.
	 * @param advice the AfterReturningAdvice to wrap
	 */
	public AfterReturningAdviceInterceptor(AfterReturningAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	public Object invoke(MethodInvocation mi) throws Throwable {
        // 先让原方法执行
		Object retVal = mi.proceed();
        // 原方法执行之后，返回值返回之前增强
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}

}
```
**ThrowsAdviceInterceptor增强((被代理对象方法抛出异常之后执行)：**
```java
@SuppressWarnings("serial")
class ThrowsAdviceAdapter implements AdvisorAdapter, Serializable {

	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof ThrowsAdvice);
	}

	public MethodInterceptor getInterceptor(Advisor advisor) {
		return new ThrowsAdviceInterceptor(advisor.getAdvice());
	}

}

public class ThrowsAdviceInterceptor implements MethodInterceptor, AfterAdvice {

	private static final String AFTER_THROWING = "afterThrowing";

	private static final Log logger = LogFactory.getLog(ThrowsAdviceInterceptor.class);


	private final Object throwsAdvice;

	/** Methods on throws advice, keyed by exception class */
	private final Map<Class, Method> exceptionHandlerMap = new HashMap<Class, Method>();


	/**
	 * Create a new ThrowsAdviceInterceptor for the given ThrowsAdvice.
	 * @param throwsAdvice the advice object that defines the exception
	 * handler methods (usually a {@link org.springframework.aop.ThrowsAdvice}
	 * implementation)
	 */
	public ThrowsAdviceInterceptor(Object throwsAdvice) {
		Assert.notNull(throwsAdvice, "Advice must not be null");
		this.throwsAdvice = throwsAdvice;

		Method[] methods = throwsAdvice.getClass().getMethods();
		for (Method method : methods) {
			if (method.getName().equals(AFTER_THROWING) &&
					(method.getParameterTypes().length == 1 || method.getParameterTypes().length == 4) &&
					Throwable.class.isAssignableFrom(method.getParameterTypes()[method.getParameterTypes().length - 1])
				) {
				// Have an exception handler
				this.exceptionHandlerMap.put(method.getParameterTypes()[method.getParameterTypes().length - 1], method);
				if (logger.isDebugEnabled()) {
					logger.debug("Found exception handler method: " + method);
				}
			}
		}

		if (this.exceptionHandlerMap.isEmpty()) {
			throw new IllegalArgumentException(
					"At least one handler method must be found in class [" + throwsAdvice.getClass() + "]");
		}
	}

	public int getHandlerMethodCount() {
		return this.exceptionHandlerMap.size();
	}

	/**
	 * Determine the exception handle method. Can return null if not found.
	 * @param exception the exception thrown
	 * @return a handler for the given exception type
	 */
	private Method getExceptionHandler(Throwable exception) {
		Class exceptionClass = exception.getClass();
		if (logger.isTraceEnabled()) {
			logger.trace("Trying to find handler for exception of type [" + exceptionClass.getName() + "]");
		}
		Method handler = this.exceptionHandlerMap.get(exceptionClass);
		while (handler == null && !exceptionClass.equals(Throwable.class)) {
			exceptionClass = exceptionClass.getSuperclass();
			handler = this.exceptionHandlerMap.get(exceptionClass);
		}
		if (handler != null && logger.isDebugEnabled()) {
			logger.debug("Found handler for exception of type [" + exceptionClass.getName() + "]: " + handler);
		}
		return handler;
	}

	public Object invoke(MethodInvocation mi) throws Throwable {
        // try-catch包装，proceed方法之后执行过程中如果抛出了异常，先调用ThrowsAdvice中的回调方法，之后将此异常向上抛出
		try {
			return mi.proceed();
		}
		catch (Throwable ex) {
			Method handlerMethod = getExceptionHandler(ex);
			if (handlerMethod != null) {
				invokeHandlerMethod(mi, ex, handlerMethod);
			}
			throw ex;
		}
	}

	private void invokeHandlerMethod(MethodInvocation mi, Throwable ex, Method method) throws Throwable {
		Object[] handlerArgs;
		if (method.getParameterTypes().length == 1) {
			handlerArgs = new Object[] { ex };
		}
		else {
			handlerArgs = new Object[] {mi.getMethod(), mi.getArguments(), mi.getThis(), ex};
		}
		try {
            // 反射调用ThrowsAdvice中的回调方法
			method.invoke(this.throwsAdvice, handlerArgs);
		}
		catch (InvocationTargetException targetEx) {
			throw targetEx.getTargetException();
		}
	}

}
```
ThrowsAdvice与BeforeAdvice和AfterAdvice接口的不同之处在于其接口中没有任何抽象方法，是一个标记接口，Spirng内部是用反射调用指定名称的默认方法来实现方法匹配的，需要实现下列接口中的其中1个：  
<img src="/img/2018-8-17/ThrowsAdviceCallBack.png" width="700" height="700" alt="ThrowsAdvice接口默认回调" />
<center>图2：ThrowsAdvice接口默认回调</center>

通过上面的代码分析过程，我们可以得知对于下面这种配置方式，其执行顺序应当是：第一个sayhelloadvice的beforeAdvice -> 第二个sayhelloadvice的beforeAdvice -> 第二个sayhelloadvice的afterAdvice -> 第一个sayhelloadvice的afterAdvice。  
```java
<!--定义advise sayhelloadvice定义了通知包含什么-->
<bean id="sayhelloadvice" class="mytests.aopTests.SayHelloBeforemale"/>

<!--waiter定义了在何时(调用proxyInterfaces接口的方法时)为什么对象(target指定，此对象需要实现proxyInterfaces接口)-->
<!--应用何种通知(interceptorNames中指定)-->
<!-- 定义advisor-->
<bean id="waiter" class="org.springframework.aop.framework.ProxyFactoryBean">   
    <property name="target">
        <bean class="mytests.aopTests.Waiter"/>
    </property>
    <property name="interceptorNames">
        <list>
            <value>sayhelloadvice</value>
            <value>sayhelloadvice</value>
        </list>
    </property>
</bean>
``` 

经过上面的分析过程，我们对Spring AOP的实现原理有了一个大致的了解，现在我们进行一下总结：  
1. 初始化ProxyFactoryBean：通过读取xml配置初始化ProxyFactoryBean对象，生成相应的Advisor将Advice和PointCut结合起来。回顾通过ProxyFactoryBean实现AOP的整个过程，可以看到，在它的实现中，首先要对目标对象以及拦截器进行正确配置，以便AopProxy代理对象顺利产生；这些配置既可以通过配置ProxyFactoryBean的属性来完成，也可以通过编程式的使用ProxyFactory来实现。这两种AOP的实现只是在表面配置的方式上不同，对于内在的AOP实现原理他们是一样的。
2. 获取代理对象：在生成AopProxy代理对象的时候，Spring AOP专门设计了AopProxyFactory作为AOP代理对象的生成工厂，由它来负责产生相应的AopProxy代理对象，默认使用的工厂是DefaultAopProxyFactory。在DefaultAopProxyFactory对象中定义了AopProxy代理对象的生成策略，从而决定使用哪种AopProxy代理独享的生成技术(使用JDK动态代理还是CGLIB)来完成生产任务。而最终的AopProxy代理对象的产生则是交给JDKDynamicAopProxy和Cglib2AopProxy这两个具体的工厂来完成，他们使用了不同的生产技术，前者是JDK动态代理(使用InvocationHandler对象的invoke完成回调)，后者是CGLIB。  
3. 对被代理对象进行增强：在我们执行得到的代理对象的方法时，前面定义的Proxy机制就起作用了。当Proxy对象暴露的方法被调用时，并不是直接运行目标对象的调用方法，而是根据Proxy的定义，改变原有的目标对象方法调用的运行轨迹。这种改变体现在，首先会触发对这些方法调用进行拦截，这些拦截为对目标方法调用的功能增强提供了工作空间，拦截过程发生在JDK的Proxy代理对象中，是通过invoke方法来完成的，这个invoke方法是虚拟机触发的一个回调，AOP切面增强就是在这个方法中进行的。    

_ _ _
### **Spring AOP与BeanPostProcessor结合**
在实际使用Spring的AOP功能时，我们往往不会直接使用到ProxyFactoryBean对象，而是通过这种配置方式(当然也可以通过注解)：
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd">
    
    <bean id="sayhelloadvice" class="mytests.aopTests.SayHelloBeforemale"/>
    <bean id="waiter" class="mytests.aopTests.Waiter"/>
    
    <aop:config>
        <aop:aspect ref = "sayhelloadvice">
            <aop:before pointcut="execution(* mytests.aopTests.Waiter.say*(..))" method="before"></aop:before>
            <aop:after-returning pointcut="execution(* mytests.aopTests.Waiter.say*(..))" method="afterReturning"></aop:after-returning>
        </aop:aspect>
    </aop:config>
</beans>
```
经过这种配置方式之后，我们获取的Waiter对象也是被代理之后的，那么代理对象是怎么生成的呢？  

原来Spring IOC容器初始化的时候在AbstractApplicationContext.refresh()方法中对registerBeanPostProcessors的调用会默认注册一些BeanPostProcessors，对于上面这种配置，就会产生一个AspectJAwareAdvisorAutoProxyCreator对象，它是一个BeanPostProcessor，会被注册到Spring IOC容器中。之后我们从IOC容器中获取的对象在实例化时都会经过此BeanPostProcessor的后置处理：  
```java
public abstract class AbstractAutoProxyCreator extends ProxyConfig
		implements SmartInstantiationAwareBeanPostProcessor, BeanClassLoaderAware, BeanFactoryAware,
		Ordered, AopInfrastructureBean {

    /**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 * 对bean进行后置处理
	 */
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.containsKey(cacheKey)) {
                // 判断是否有必要对bean进行代理
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

    /**
	 * Wrap the given bean if necessary, i.e. if it is eligible for being proxied.
	 * @param bean the raw bean instance
	 * @param beanName the name of the bean
	 * @param cacheKey the cache key for metadata access
	 * @return a proxy wrapping the bean, or the raw bean instance as-is
	 */
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (beanName != null && this.targetSourcedBeans.containsKey(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // 实际产生代理对象的方法
			Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

    /**
	 * Create an AOP proxy for the given bean.
	 * @param beanClass the class of the bean
	 * @param beanName the name of the bean
	 * @param specificInterceptors the set of interceptors that is
	 * specific to this bean (may be empty, but not null)
	 * @param targetSource the TargetSource for the proxy,
	 * already pre-configured to access the bean
	 * @return the AOP proxy for the bean
	 * @see #buildAdvisors	 
	 */
	protected Object createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

        // 可见实际上还是使用了ProxyFactory来产生的代理对象
		ProxyFactory proxyFactory = new ProxyFactory();
		// Copy our properties (proxyTargetClass etc) inherited from ProxyConfig.
		proxyFactory.copyFrom(this);

		if (!shouldProxyTargetClass(beanClass, beanName)) {
			// Must allow for introductions; can't just set interfaces to
			// the target's interfaces only.
			Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, this.proxyClassLoader);
			for (Class<?> targetInterface : targetInterfaces) {
				proxyFactory.addInterface(targetInterface);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		for (Advisor advisor : advisors) {
			proxyFactory.addAdvisor(advisor);
		}

		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(this.proxyClassLoader);
	}
}
```
可见正是通过上面的过程判断是否需要对目标对象进行代理(如果需要代理就使用ProxyFactory方式生成)，从而实现AOP功能的。

(完)  

参考文章：  
《Spring技术内幕》  
[BeanPostProcessor —— 连接Spring IOC和AOP的桥梁](http://www.cnblogs.com/yuxiang1/archive/2018/06/19/9199730.html)
  



待完成：SpringAOP整个设计类图意图分析，SpringAOP官方文档学习  
