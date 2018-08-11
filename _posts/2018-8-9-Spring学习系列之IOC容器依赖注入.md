---
layout:     post
title:      Spring学习系列之IOC容器依赖注入
date:       2018-8-9
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Spring
---
>本文记录我对Spring IOC容器依赖注入相关过程及注意事项的理解(基于Spring3.2版本)。       

_ _ _
### **前言**
上篇博客记录并简单分析了Spring IOC容器的初始过程，这个初始化的过程主要是在IOC容器中建立BeanDefinition数据映射，在此过程中并没有看到IOC容器对Bean依赖关系进行注入，本篇博客就来分析一下IOC容器是怎样对Bean的依赖关系进行注入的。  

一般情况下依赖注入的过程是在用户第一次向IOC容器索要Bean时触发的:
```java
public class TestFileSystemXmlApplicationContext {
    public static void main(String[] args) {
        FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext("beans.xml");
        ((org.springframework.mytests.BlankDisc) (context.getBean("compactDisc"))).play();
    }
}
```
下面就以getBean()方法为例来分析依赖注入过程的实现。  

_ _ _
### **getBean()中究竟做了什么**
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    public Object getBean(String name) throws BeansException {
        // 这里获得的beanFactory是DefaultListableBeanFactory类的对象
		return getBeanFactory().getBean(name);
	}   
    public Object getBean(String name) throws BeansException {       
		return doGetBean(name, null, null, false);
	}    
}

public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
    // 这里是实际取得Bean的地方，也是触发依赖注入发生的地方
    protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// 先从缓存中取得Bean，处理那些已经被创建过的单例模式的Bean，对这种Bean的请求不需要重复的创建
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
            // 这里再调用getObjectForBeanInstance对sharedInstance进行处理，原因之一是sharedInstance可能是一个FactoryBean，
            // 调用其getObject()方法返回的才是真正的Bean，下同
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		else {
            // 非单例模式实例
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
            // 检查这个BeanFactory中是否存在与beanName对应的BeanDefinition，不存在的话就到双亲BeanFactory中进行查找
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
                // 根据beanName获取到对应的BeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
                // 获取当前Bean的所有依赖bean，这会触发getBean的依赖调用，直到获取到一个没有依赖的bean为止
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dependsOnBean : dependsOn) {
						getBean(dependsOnBean);
						registerDependentBean(dependsOnBean, beanName);
					}
				}

				// Create bean instance. 如果要创建的Bean是单例的话
				if (mbd.isSingleton()) {
                    // 调用DefaultSingletonBeanRegistry的getSingleton方法
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                        /**
                         * 这里通过调用createBean方法创建singletonBean的实例，这里有一个回调函数getObject()，会在
                         * getSingleton中调用ObjectFactory的createBean方法，由于是单例，创建之后要缓存此bean
                         */
						public Object getObject() throws BeansException {
							try {
                                // 调用AbstractAutowireCapableBeanFactory类createBean方法创建Bean，下同
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) { // 如果要创建的Bean作用域为Prototype
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
                        // 由于作用域是prototype，直接创建一个新的bean 就好了
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else { // 其它作用域的bean
					String scopeName = mbd.getScope(); // 获取beanDefinition中的bean作用域名称
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
					}
					try {
                        // 创建相应作用域的bean
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; " +
								"consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
        // 对创建的bean进行类型检查，如果没有问题，就返回这个新创建的已经包含了依赖关系的bean
		if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type [" +
							ClassUtils.getQualifiedName(requiredType) + "]", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
}

public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    
    /** Cache of singleton objects: bean name --> bean instance, singleton对象的缓存Map */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

    // 查找指定beanName的singleton bean，不存在则创建
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while the singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				beforeSingletonCreation(beanName);
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<Exception>();
				}
				try {
                    // 使用工厂方法创建单例对象，有可能获得的是属性尚未注入完成的缓存的对象
					singletonObject = singletonFactory.getObject();
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
                // 添加到singleton bean map的缓存当中去
				addSingleton(beanName, singletonObject);
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}

    ...  
}

public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    @Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		// Make sure bean class is actually resolved at this point.
        // 这里判断需要创建的bean是否可以实例化，这个类是否可以通过类装载器来载入
		resolveBeanClass(mbd, beanName);

		// Prepare method overrides.
		try {
			mbd.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
            // 如果Bean配置了PostProcessor，那么这里返回的是一个proxy，BeanPostProcessor的回调方法不是在doCreateBean方法
            // 中的initializeBean方法中被包含的吗？？？  
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbd);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}
        // 创建目标bean
		Object beanInstance = doCreateBean(beanName, mbd, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
    // 通过反射相关方法创建目标Bean
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		// Instantiate the bean. 这个BeanWrapper是用来持有创建出来的Bean对象的
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
            // 如果是singleton，先把缓存中的同名bean清除
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
        // 实际创建Bean的地方，由createBeanInstance来完成，这里由于实例化对象时可能依赖于其他的对象，可能涉及到依赖注入
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
        /**
         * 为了解决单例setter循环引用问题，通过Spring容器提前暴露刚完毕构造器注入但未完毕其它步骤（如setter注入）的Bean来完成
         */
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
            // 缓存当前构造函数已经调用过的但是属性还未注入完成的bean
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
        // 创建Bean之后继续进行bean的初始化，依赖注入往往在这里发生，这个exposedObject在初始化处理完以后会返回
        // 作为依赖注入完成后的Bean
		Object exposedObject = bean;
		try {
            // 依赖注入在populateBean中发生
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
                // 继续进行Bean的初始化，处理Bean生命周期中的各种回调
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}

    /**
     * 初始化给定的bean对象，应用factory回调、init方法以及bean post processors
     */
    protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				public Object run() {
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}

		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
	}

    // 检查Aware接口并设置相关依赖
    private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
    
    // BeanPostProcessor前置处理
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			result = beanProcessor.postProcessBeforeInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
		return result;
	}
    
    /**
     * 包含对Initializingbean的处理和自动以init-method的处理     
     */
    protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						public Object run() throws Exception {
							((InitializingBean) bean).afterPropertiesSet();
							return null;
						}
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null) {
			String initMethodName = mbd.getInitMethodName();
			if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
    
    // BeanPostProcessor后置处理
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			result = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
		return result;
	}
}
```
在上面doCreateBean方法中可以看到，与依赖注入关系特别密切的方法有createBeanInstance和populateBean,首先来分析createBeanInstance方法：  
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    /**
	 * Create a new instance for the specified bean, using an appropriate instantiation strategy:
	 * factory method, constructor autowiring, or simple instantiation.
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param args arguments to use if creating a prototype using explicit arguments to a
	 * static factory method. It is invalid to use a non-null args value in any other case.
	 * @return BeanWrapper for the new instance
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 * @see #instantiateBean
	 * 根据情况选择三种实例化策略之一：instantiateUsingFactoryMethod、autowireConstructor、instantiateBean
	 */
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
		// Make sure bean class is actually resolved at this point.
        // 确认需要创建的Bean实例的类可以实例化
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
        // 使用工厂方法对bean实例化
		if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}
        // 使用默认的构造函数进行实例化
		// Need to determine the constructor...
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// No special handling: simply use no-arg constructor.
		return instantiateBean(beanName, mbd);
	}
    // 使用默认的构造函数初始化Bean
    protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
            // 使用默认的实例化策略对Bean进行实例化，默认的实例化策略是CglibSubclassingInstantiationStrategy,也就是使用
            // CGlib来对Bean进行实例化
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
					public Object run() {
						return getInstantiationStrategy().instantiate(mbd, beanName, parent);
					}
				}, getAccessControlContext());
			}
			else {
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
}

```
使用createBeanInstance进行实例化对象之后，就是在populateBean中对这些对象的依赖关系,比如填充<bean>标签中的<property>指定的属性等，完成依赖注入过程：  
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    /**
	 * Populate the bean instance in the given BeanWrapper with the property values
	 * from the bean definition.
	 * 根据bean definition中的property值填充给定BeanWrapper中的bean
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param bw BeanWrapper with bean instance
	 */
	protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
		PropertyValues pvs = mbd.getPropertyValues();

		if (bw == null) {
			if (!pvs.isEmpty()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		boolean continueWithPropertyPopulation = true;

		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}

        /**
         * bean标签内采用的是autowire的模式自动装配类中的属性，而不是<property>与ref相结合的模式，比如：
         *   <bean id="computer3" class="ioc.test.Computer" autowire="default"></bean>
         *   public class Computer {     
         *      private Host host;
         *     private Display display;
         *      //电脑运行方法
         *      public void run(){
         *        System.out.println("你好，我是电脑，正在运行！");
         *        System.out.print("   "+host.run()+",");
         *        System.out.println(display.run());
         *      }
         *      //Geter和Seter方法，省略
         *    }
         */
        // 开始进行依赖注入的过程，先处理autowire的注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
            // 可以根据bean的名字或者类型进行autowire
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}
        // 应该是对一些钩子函数的处理？？？
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}
        // 对bean属性进行注入
		applyPropertyValues(beanName, mbd, bw, pvs);
	}

    /**
	 * Apply the given property values, resolving any runtime references
	 * to other beans in this bean factory. Must use deep copy, so we
	 * don't permanently modify this property.
	 * @param beanName the bean name passed for better exception information
	 * @param mbd the merged bean definition
	 * @param bw the BeanWrapper wrapping the target object
	 * @param pvs the new property values
	 */
    // 通过applyPropertyValues了解具体的对属性进行解析然后注入的过程
	protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs == null || pvs.isEmpty()) {
			return;
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original; // 保存初始PropertyValue的list

		if (System.getSecurityManager() != null) {
			if (bw instanceof BeanWrapperImpl) {
				((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
			}
		}

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			if (mpvs.isConverted()) {
				// Shortcut: use the pre-converted values as-is.
				try {
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			original = mpvs.getPropertyValueList();
		}
		else {
			original = Arrays.asList(pvs.getPropertyValues());
		}

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// Create a deep copy, resolving any references for values.
        /**
         * 这个list的目的是保存对origin list中PropertyValue中的value解析后的结果
         *    比如对于<property name="cdPlayer" ref="cdPlayer"/>就需要解析ref中的引用的对象，没有则触发依赖注入
         */
		List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
		boolean resolveNecessary = false;
        
        // 对每个PropertyValue进行解析
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
                // 使用BeanDefinitionValueResolver进行解析
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// Set our (possibly massaged) deep copy.
		try {
            // 将解析后的PropertyValues注入到BeanWrapper中，真正的依赖注入
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
	}
}
```
下面先来看BeanDefinitionValueResolver对PropertyValue进行解析的过程，之后再看解析后的属性注入过程bw.setPropertyValues(new MutablePropertyValues(deepCopy))：  
```java
class BeanDefinitionValueResolver {
    
    /**
	 * Given a PropertyValue, return a value, resolving any references to other
	 * beans in the factory if necessary. The value could be:
	 * <li>A BeanDefinition, which leads to the creation of a corresponding
	 * new bean instance. Singleton flags and names of such "inner beans"
	 * are always ignored: Inner beans are anonymous prototypes.
	 * <li>A RuntimeBeanReference, which must be resolved.
	 * <li>A ManagedList. This is a special collection that may contain
	 * RuntimeBeanReferences or Collections that will need to be resolved.
	 * <li>A ManagedSet. May also contain RuntimeBeanReferences or
	 * Collections that will need to be resolved.
	 * <li>A ManagedMap. In this case the value may be a RuntimeBeanReference
	 * or Collection that will need to be resolved.
	 * <li>An ordinary object or {@code null}, in which case it's left alone.
	 * @param argName the name of the argument that the value is defined for
	 * @param value the value object to resolve
	 * @return the resolved object
	 */
	public Object resolveValueIfNecessary(Object argName, Object value) {
		// We must check each value to see whether it requires a runtime reference
		// to another bean to be resolved.
        // 这里对RuntimeBeanReference进行解析，RuntimeBeanReference是在对BeanDefinition进行解析时生成的数据对象
        // 是不是可以理解为对类似<property name="cdPlayer" ref="cdPlayer"/>标签的处理？？？
		if (value instanceof RuntimeBeanReference) {
			RuntimeBeanReference ref = (RuntimeBeanReference) value;
			return resolveReference(argName, ref);
		}
		else if (value instanceof RuntimeBeanNameReference) {
			String refName = ((RuntimeBeanNameReference) value).getBeanName();
			refName = String.valueOf(evaluate(refName));
			if (!this.beanFactory.containsBean(refName)) {
				throw new BeanDefinitionStoreException(
						"Invalid bean name '" + refName + "' in bean reference for " + argName);
			}
			return refName;
		}
		else if (value instanceof BeanDefinitionHolder) {
			// Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
			BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
			return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
		}
		else if (value instanceof BeanDefinition) {
			// Resolve plain BeanDefinition, without contained name: use dummy name.
			BeanDefinition bd = (BeanDefinition) value;
			String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
					ObjectUtils.getIdentityHexString(bd);
			return resolveInnerBean(argName, innerBeanName, bd);
		}
        // 对ManagedArray进行解析
		else if (value instanceof ManagedArray) {
			// May need to resolve contained runtime references.
			ManagedArray array = (ManagedArray) value;
			Class<?> elementType = array.resolvedElementType;
			if (elementType == null) {
				String elementTypeName = array.getElementTypeName();
				if (StringUtils.hasText(elementTypeName)) {
					try {
						elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
						array.resolvedElementType = elementType;
					}
					catch (Throwable ex) {
						// Improve the message by showing the context.
						throw new BeanCreationException(
								this.beanDefinition.getResourceDescription(), this.beanName,
								"Error resolving array type for " + argName, ex);
					}
				}
				else {
					elementType = Object.class;
				}
			}
			return resolveManagedArray(argName, (List<?>) value, elementType);
		}
        // 对ManagedList进行解析
		else if (value instanceof ManagedList) {
			// May need to resolve contained runtime references.
			return resolveManagedList(argName, (List<?>) value);
		}
        // 对ManagedSet进行解析
		else if (value instanceof ManagedSet) {
			// May need to resolve contained runtime references.
			return resolveManagedSet(argName, (Set<?>) value);
		}
        // 对ManagedMap进行解析
		else if (value instanceof ManagedMap) {
			// May need to resolve contained runtime references.
			return resolveManagedMap(argName, (Map<?, ?>) value);
		}
		else if (value instanceof ManagedProperties) {
			Properties original = (Properties) value;
			Properties copy = new Properties();
			for (Map.Entry<Object, Object> propEntry : original.entrySet()) {
				Object propKey = propEntry.getKey();
				Object propValue = propEntry.getValue();
				if (propKey instanceof TypedStringValue) {
					propKey = evaluate((TypedStringValue) propKey);
				}
				if (propValue instanceof TypedStringValue) {
					propValue = evaluate((TypedStringValue) propValue);
				}
				copy.put(propKey, propValue);
			}
			return copy;
		}
		else if (value instanceof TypedStringValue) {
			// Convert value to target type here.
			TypedStringValue typedStringValue = (TypedStringValue) value;
			Object valueObject = evaluate(typedStringValue);
			try {
				Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
				if (resolvedTargetType != null) {
					return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
				}
				else {
					return valueObject;
				}
			}
			catch (Throwable ex) {
				// Improve the message by showing the context.
				throw new BeanCreationException(
						this.beanDefinition.getResourceDescription(), this.beanName,
						"Error converting typed String value for " + argName, ex);
			}
		}
		else {
			return evaluate(value);
		}
	}

    /**
	 * Resolve a reference to another bean in the factory.
	 * 解析引用指向beanfactory中的另一个bean
	 */
	private Object resolveReference(Object argName, RuntimeBeanReference ref) {
		try {
			String refName = ref.getBeanName();
			refName = String.valueOf(evaluate(refName));
            // 如果ref是在双亲IOC容器中，那就到双亲IOC容器中去获取
			if (ref.isToParent()) {
				if (this.beanFactory.getParentBeanFactory() == null) {
					throw new BeanCreationException(
							this.beanDefinition.getResourceDescription(), this.beanName,
							"Can't resolve reference to bean '" + refName +
							"' in parent factory: no parent factory available");
				}
				return this.beanFactory.getParentBeanFactory().getBean(refName);
			}
			else {
                // 在当前IOC容器中取得Bean，这里会触发一个getBean的过程，如果依赖注入没有发生，这里会触发相应的依赖注入的发生
				Object bean = this.beanFactory.getBean(refName);
				this.beanFactory.registerDependentBean(refName, this.beanName);
				return bean;
			}
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					this.beanDefinition.getResourceDescription(), this.beanName,
					"Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
		}
	}
}
```
介绍完对PropertyValue进行解析的过程，来看将解析后的属性进行注入的过程：  
```java
public class BeanWrapperImpl extends AbstractPropertyAccessor implements BeanWrapper {

    private void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
		String propertyName = tokens.canonicalName;
		String actualName = tokens.actualName;

		if (tokens.keys != null) { // 对集合类的域进行注入
			// Apply indexes and map keys: fetch value for all keys but the last one.
			PropertyTokenHolder getterTokens = new PropertyTokenHolder();
			getterTokens.canonicalName = tokens.canonicalName;
			getterTokens.actualName = tokens.actualName;
			getterTokens.keys = new String[tokens.keys.length - 1];
			System.arraycopy(tokens.keys, 0, getterTokens.keys, 0, tokens.keys.length - 1);
			Object propValue;
			try { // getPropertyValue取得Bean中对注入对象的引用，比如Array、List、Map、Set等
				propValue = getPropertyValue(getterTokens);
			}
			catch (NotReadablePropertyException ex) {
				throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
						"Cannot access indexed value in property referenced " +
						"in indexed property path '" + propertyName + "'", ex);
			}
			// Set value for last key.
			String key = tokens.keys[tokens.keys.length - 1];
			if (propValue == null) {
				// null map value case
				if (this.autoGrowNestedPaths) {
					// TODO: cleanup, this is pretty hacky
					int lastKeyIndex = tokens.canonicalName.lastIndexOf('[');
					getterTokens.canonicalName = tokens.canonicalName.substring(0, lastKeyIndex);
					propValue = setDefaultValue(getterTokens);
				}
				else {
					throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + propertyName,
							"Cannot access indexed value in property referenced " +
							"in indexed property path '" + propertyName + "': returned null");
				}
			}
			if (propValue.getClass().isArray()) { // 对Array进行注入
				PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
				Class<?> requiredType = propValue.getClass().getComponentType();
				int arrayIndex = Integer.parseInt(key);
				Object oldValue = null;
				try {
					if (isExtractOldValueForEditor() && arrayIndex < Array.getLength(propValue)) {
						oldValue = Array.get(propValue, arrayIndex);
					}
					Object convertedValue = convertIfNecessary(propertyName, oldValue, pv.getValue(),
							requiredType, TypeDescriptor.nested(property(pd), tokens.keys.length));
					Array.set(propValue, arrayIndex, convertedValue);
				}
				catch (IndexOutOfBoundsException ex) {
					throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
							"Invalid array index in property path '" + propertyName + "'", ex);
				}
			}
			else if (propValue instanceof List) { // 对List进行注入
				PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
				Class<?> requiredType = GenericCollectionTypeResolver.getCollectionReturnType(
						pd.getReadMethod(), tokens.keys.length);
				List<Object> list = (List<Object>) propValue;
				int index = Integer.parseInt(key);
				Object oldValue = null;
				if (isExtractOldValueForEditor() && index < list.size()) {
					oldValue = list.get(index);
				}
				Object convertedValue = convertIfNecessary(propertyName, oldValue, pv.getValue(),
						requiredType, TypeDescriptor.nested(property(pd), tokens.keys.length));
				int size = list.size();
				if (index >= size && index < this.autoGrowCollectionLimit) {
					for (int i = size; i < index; i++) {
						try {
							list.add(null);
						}
						catch (NullPointerException ex) {
							throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
									"Cannot set element with index " + index + " in List of size " +
									size + ", accessed using property path '" + propertyName +
									"': List does not support filling up gaps with null elements");
						}
					}
					list.add(convertedValue);
				}
				else {
					try {
						list.set(index, convertedValue);
					}
					catch (IndexOutOfBoundsException ex) {
						throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
								"Invalid list index in property path '" + propertyName + "'", ex);
					}
				}
			}
			else if (propValue instanceof Map) { // 对Map进行注入
				PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
				Class<?> mapKeyType = GenericCollectionTypeResolver.getMapKeyReturnType(
						pd.getReadMethod(), tokens.keys.length);
				Class<?> mapValueType = GenericCollectionTypeResolver.getMapValueReturnType(
						pd.getReadMethod(), tokens.keys.length);
				Map<Object, Object> map = (Map<Object, Object>) propValue;
				// IMPORTANT: Do not pass full property name in here - property editors
				// must not kick in for map keys but rather only for map values.
				TypeDescriptor typeDescriptor = (mapKeyType != null ?
						TypeDescriptor.valueOf(mapKeyType) : TypeDescriptor.valueOf(Object.class));
				Object convertedMapKey = convertIfNecessary(null, null, key, mapKeyType, typeDescriptor);
				Object oldValue = null;
				if (isExtractOldValueForEditor()) {
					oldValue = map.get(convertedMapKey);
				}
				// Pass full property name and old value in here, since we want full
				// conversion ability for map values.
				Object convertedMapValue = convertIfNecessary(propertyName, oldValue, pv.getValue(),
						mapValueType, TypeDescriptor.nested(property(pd), tokens.keys.length));
				map.put(convertedMapKey, convertedMapValue);
			}
			else {
				throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
						"Property referenced in indexed property path '" + propertyName +
						"' is neither an array nor a List nor a Map; returned value was [" + propValue + "]");
			}
		}

		else { // 对非集合类的域进行注入
			PropertyDescriptor pd = pv.resolvedDescriptor;
			if (pd == null || !pd.getWriteMethod().getDeclaringClass().isInstance(this.object)) {
				pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
				if (pd == null || pd.getWriteMethod() == null) {
					if (pv.isOptional()) {
						logger.debug("Ignoring optional value for property '" + actualName +
								"' - property not found on bean class [" + getRootClass().getName() + "]");
						return;
					}
					else {
						PropertyMatches matches = PropertyMatches.forProperty(propertyName, getRootClass());
						throw new NotWritablePropertyException(
								getRootClass(), this.nestedPath + propertyName,
								matches.buildErrorMessage(), matches.getPossibleMatches());
					}
				}
				pv.getOriginalPropertyValue().resolvedDescriptor = pd;
			}

			Object oldValue = null;
			try {
				Object originalValue = pv.getValue();
				Object valueToApply = originalValue;
				if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
					if (pv.isConverted()) {
						valueToApply = pv.getConvertedValue();
					}
					else {
						if (isExtractOldValueForEditor() && pd.getReadMethod() != null) {
							final Method readMethod = pd.getReadMethod();
							if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers()) &&
									!readMethod.isAccessible()) {
								if (System.getSecurityManager()!= null) {
									AccessController.doPrivileged(new PrivilegedAction<Object>() {
										public Object run() {
											readMethod.setAccessible(true);
											return null;
										}
									});
								}
								else {
									readMethod.setAccessible(true);
								}
							}
							try {
								if (System.getSecurityManager() != null) {
									oldValue = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
										public Object run() throws Exception {
											return readMethod.invoke(object);
										}
									}, acc);
								}
								else {
									oldValue = readMethod.invoke(object);
								}
							}
							catch (Exception ex) {
								if (ex instanceof PrivilegedActionException) {
									ex = ((PrivilegedActionException) ex).getException();
								}
								if (logger.isDebugEnabled()) {
									logger.debug("Could not read previous value of property '" +
											this.nestedPath + propertyName + "'", ex);
								}
							}
						}
						valueToApply = convertForProperty(
								propertyName, oldValue, originalValue, new TypeDescriptor(property(pd)));
					}
					pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
				}
                // 这里取得注入属性的set方法，通过反射机制，把对象注入进去
				final Method writeMethod = (pd instanceof GenericTypeAwarePropertyDescriptor ?
						((GenericTypeAwarePropertyDescriptor) pd).getWriteMethodForActualAccess() :
						pd.getWriteMethod());
				if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers()) && !writeMethod.isAccessible()) {
					if (System.getSecurityManager()!= null) {
						AccessController.doPrivileged(new PrivilegedAction<Object>() {
							public Object run() {
								writeMethod.setAccessible(true);
								return null;
							}
						});
					}
					else {
						writeMethod.setAccessible(true);
					}
				}
				final Object value = valueToApply;
				if (System.getSecurityManager() != null) {
					try {
						AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
							public Object run() throws Exception {
								writeMethod.invoke(object, value);
								return null;
							}
						}, acc);
					}
					catch (PrivilegedActionException ex) {
						throw ex.getException();
					}
				}
				else {
					writeMethod.invoke(this.object, value);
				}
			}
			catch (TypeMismatchException ex) {
				throw ex;
			}
			catch (InvocationTargetException ex) {
				PropertyChangeEvent propertyChangeEvent =
						new PropertyChangeEvent(this.rootObject, this.nestedPath + propertyName, oldValue, pv.getValue());
				if (ex.getTargetException() instanceof ClassCastException) {
					throw new TypeMismatchException(propertyChangeEvent, pd.getPropertyType(), ex.getTargetException());
				}
				else {
					throw new MethodInvocationException(propertyChangeEvent, ex.getTargetException());
				}
			}
			catch (Exception ex) {
				PropertyChangeEvent pce =
						new PropertyChangeEvent(this.rootObject, this.nestedPath + propertyName, oldValue, pv.getValue());
				throw new MethodInvocationException(pce, ex);
			}
		}
	}
}
```

_ _ _
### **Bean生命周期总结**
在上面分析完了Bean的依赖注入过程之后，对Bean的生命周期做一个总结：  

<img src="/img/2018-8-9/SpringBeanLifeCycle.png" width="500" height="500" alt="SpringBeanLifeCycle图示" />
<center>图1：Spring Bean生命周期图</center>    

接下来就来看看生命周期图中的各个阶段在代码中哪里体现的：

1) 实例化Bean对象，即调用其构造方法
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    // 通过反射相关方法创建目标Bean
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		// Instantiate the bean. 这个BeanWrapper是用来持有创建出来的Bean对象的
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
            // 如果是singleton，先把缓存中的同名bean清除
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
        // 实际创建Bean的地方，由createBeanInstance来完成，这里由于实例化对象时可能依赖于其他的对象，可能涉及到依赖注入
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		...
	}
}
```

2) 设置对象属性，即调用其set方法
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    // 通过反射相关方法创建目标Bean
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		
        ...

		// Initialize the bean instance.
        // 创建Bean之后继续进行bean的初始化，依赖注入往往在这里发生，这个exposedObject在初始化处理完以后会返回
        // 作为依赖注入完成后的Bean
		Object exposedObject = bean;
		try {
            // 依赖注入在populateBean中发生,在populateBean注入bean的相关属性
			populateBean(beanName, mbd, instanceWrapper);
			...
	}
}
```

3) 检查Aware接口并设置相关依赖
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    // 通过反射相关方法创建目标Bean
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		
        ...

		// Initialize the bean instance.
        // 创建Bean之后继续进行bean的初始化，依赖注入往往在这里发生，这个exposedObject在初始化处理完以后会返回
        // 作为依赖注入完成后的Bean
		Object exposedObject = bean;
		try {
            // 依赖注入在populateBean中发生,在populateBean注入bean的相关属性
			populateBean(beanName, mbd, instanceWrapper);
            if (exposedObject != null) {
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		...
	}

    protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
        // 在invokeAwareMethods方法中检查Aware接口并设置相关依赖
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				public Object run() {
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		...
	}
    
    private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
}
```
在这里可以看到这里检查的Aware类的子接口有三种，但据Spring官方文档中描述，Aware接口实现至少有以下几种：  

<img src="/img/2018-8-9/AwareImplemention.png" width="700" height="700" alt="重要的Aware接口" />
<center>图2：重要的Aware接口</center>

除了上面检查的三个子接口，其余的是在Bean生命周期中什么时候被检查的呢？  

原来Spring中还在容器start时预先注册了一些BeanPostProcessor，比如用来处理ApplicationContextAware回调的ApplicationContextAwareProcessor，对继承自ApplicationContextAware的bean进行处理，调用其setApplicationContext。

4) BeanPostProcessors前置处理     
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
    
    protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {		
        ...
		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}
        ...
    }

    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			result = beanProcessor.postProcessBeforeInitialization(result, beanName);
            // 某个BeanPostProcessor.postProcessBeforeInitialization返回null，会阻止其他BeanPostProcessor继续处理
			if (result == null) { 
				return result;
			}
		}
        // 返回的result可能是被BeanPostProcessor包装后的  
		return result;
	}
}
```

Spring框架中注册的BeanPostProcessor对于所有Bean的生命周期都会拦截，使用要点：  
>BeanPostProcessors作用在一个bean(或者对象)的实例上;也就是说，Spring IoC实例化一个bean实例之后， BeanPostProcessors，才开始进行处理。 BeanPostProcessors作用范围是每一个容器。这仅仅和你正在使用容器有关。如果你在一个容器中定义了一个BeanPostProcessor ，它将仅仅后置处理那个容器中的beans。换言之，一个容器中的beans不会被另一个容器中的BeanPostProcessor处理，即使这两个容器，具有相同的父类。  

关于其具体使用方式，可以参考：   
[Spring BeanPostProcessor接口使用](https://www.jianshu.com/p/e1c3c6e90e8a)

5) 检查是否是InitializingBean以决定是否调用afterPropertiesSet方法 以及 检查是否有自定义的init-method
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

    protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {		
        ...
		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		...
	}

    protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						public Object run() throws Exception {
							((InitializingBean) bean).afterPropertiesSet();
							return null;
						}
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
                // 检查是否是InitializingBean以决定是否调用afterPropertiesSet方法
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null) {
            // 检查是否有自定义的init-method并反射调用
			String initMethodName = mbd.getInitMethodName();
			if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
}
```

6) BeanPostProcessors后置处理
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
    
    protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {		
        ...
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
        ...
    }

    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
			result = beanProcessor.postProcessAfterInitialization(result, beanName);
			if (result == null) {
				return result;
			}
		}
		return result;
	}
}
``` 
7) 注册必要的Destruction相关回调接口
```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
    
    // 通过反射相关方法创建目标Bean
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {		
        ...
		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
}
```
到这一步之后，Bean就已经注入完成，可以使用了。  

**最后补充介绍一下BeanFactoryPostProcessor，与BeanPostProcessor的不同之处在于，BeanFactoryPostProcessor是在Bean生命周期开始之前(即Bean实例化之前)就起作用了，BeanPostProcessor在Bean生命周期之中起作用。**  

也就是说，Spring的IoC容器允许 BeanFactoryPostProcessor来读取配置元数据并在容器实例化任何bean(除了BeanFactoryPostProcessor)之前可以修改它。  

可以通过设置order属性来控制这些BeanFactoryPostProcessor执行的顺序。可以设置这个属性仅当BeanFactoryPostProcessor实现了Ordered接口。如果你编写自己的BeanFactoryPostProcessor你也应该考虑实现Ordered接口。  

Spring包含了一些预定义的bean工厂后置处理器， 比如PropertyOverrideConfigurer和PropertyPlaceholderConfigurer。下面是使用PropertyPlaceholderConfigurer的一个例子：  
```java
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations" value="classpath:com/foo/jdbc.properties"/>
</bean>

<bean id="dataSource" destroy-method="close"
        class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>
```
上面这个示例展示了从Properties文件中配置属性的方法。在运行时，PropertyPlaceholderConfigurer就会用于元数据并为数据源替换一些属性。指定替换的值作为${property-name}形式中的占位符。  

而真正的值是来自于标准的 Java Properties 格式的文件：
>jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root

PropertyPlaceholderConfigurerk类继承自PropertyResourceConfigurer，实现了BeanFactoryPostProcessor接口：  
```java
public abstract class PropertyResourceConfigurer extends PropertiesLoaderSupport
		implements BeanFactoryPostProcessor, PriorityOrdered {
    
    ...

    /**
	 * {@linkplain #mergeProperties Merge}, {@linkplain #convertProperties convert} and
	 * {@linkplain #processProperties process} properties against the given bean factory.
	 * @throws BeanInitializationException if any properties cannot be loaded
	 */
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		try {
			Properties mergedProps = mergeProperties();

			// Convert the merged properties, if necessary.
			convertProperties(mergedProps);

			// Let the subclass process the properties.
			processProperties(beanFactory, mergedProps);
		}
		catch (IOException ex) {
			throw new BeanInitializationException("Could not load properties", ex);
		}
	}

    ...
}
```

_ _ _
### **总结**
到这里为止，Bean依赖注入的过程大体上就介绍完成了，依赖注入时会根据当前Bean的依赖Bean递归的触发对依赖Bean的创建和注入，这样根据依赖关系，一层一层的完成Bean的创建和注入，直到最后完成当前Bean的创建。  

Bean创建的过程中可能遇到循环引用的问题，对于非单例对象的创建以及单例对象但属于构造函数循环引用，Spring在创建bean的过程中会抛出异常，对于单例对象setter循环引用Spring通过暴露调用过构造函数但未注入相应属性值的对象来解决循环引用问题，具体参见AbstractAutowireCapableBeanFactory.doCreateBean方法。  

(完)

参考：  
《Spring技术内幕》  
[Spring源代码解析 ---- 循环依赖](https://www.cnblogs.com/bhlsheji/p/5208076.html)

