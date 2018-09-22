---
layout:     post
title:      Spring学习系列之IOC容器初始化
date:       2018-8-8
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Spring
---
>本文记录我对Spring IOC容器初始化相关过程及注意事项的理解(基于Spring3.2版本)。     

_ _ _
### **前言**
Spring IOC容器是IOC思想的其中一种实现，是用来管理对象的依赖关系的。Spring IOC容器中有两个主要的容器系列，一个是实现BeanFactory接口的简单容器系列，这系列容器只实现了容器的最基本功能；另一个是ApplicationContext应用上下文，它是BeanFactory的子接口，其具体实现中封装了BeanFactory并添加了一些其它功能，作为容器的高级形态而存在。继承类图如下：  
<img src="/img/2018-8-8/applicationContextExtendsBeanFactory.png" width="700" height="700" alt="ApplicationContext的继承图" />
<center>图1：ApplicationContext的继承图</center>   


下面来分别看下使用BeanFactory和ApplicationContext获取Bean的方式：  
```java
public class BlankDisc {
    private String title;
    private String artist;
    private List<String> tracks;

    public void setTitle(String title) {
        this.title = title;
    }

    public void setArtist(String artist) {
        this.artist = artist;
    }

    public void setTracks(List<String> tracks) {
        this.tracks = tracks;
    }

    public void play() {
        System.out.println("playing " + title + " by " + artist);
        for (String track : tracks) {
            System.out.println("-Track: " + track);
        }
    }
}

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context-4.1.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
    <bean id = "compactDisc" class="org.springframework.mytests.BlankDisc">
        <property name="title" value="my title"/>
        <property name="artist" value="my artist"/>
        <property name="tracks">
            <list>
                <value>tracks1</value>
                <value>tracks2</value>
                <value>tracks3</value>
                <value>tracks4</value>
            </list>
        </property>
    </bean>
</beans>
```
使用BeanFactory方式：  
```java
public class TestDefaultListableBeanFactory {
    public static void main(String[] args) {
        ClassPathResource res = new ClassPathResource("beans.xml");
        DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
        reader.loadBeanDefinitions(res);
        
        ((BlankDisc) (factory.getBean("compactDisc"))).play();
    }
}
```
在上面使用BeanFactory获取Bean的过程中我们可以看到，使用IOC容器时需要以下几个步骤：
1. 创建IOC配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息。  
2. 创建一个BeanFactory，这里使用DefaultListableBeanFactory。  
3. 创建一个读取器，并通过回调方式配置给BeanFactory。  
4. 从定义好的位置读入配置信息，具体的解析过程由XmlBeanDefinitionReader来完成。完成整个载入和注册Bean定义之后，需要的IOC容器就建立起来了，这个时候就可以直接使用IOC容器了。   

使用ApplicationContext方式：
```java
public class TestFileSystemXmlApplicationContext {
    public static void main(String[] args) {
        FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext("beans.xml");
        ((org.springframework.mytests.BlankDisc) (context.getBean("compactDisc"))).play();
    }
}
```
可见使用FileSystemXmlApplicationContext方式获取Bean相比于BeanFactory无疑更加优雅，但是在其内部其实也是封装了上述BeanFactory获取Bean的步骤。    

Spring框架将IOC容器的初始化过程分为了3部分，即Resource的定位、BeanDefinition的载入和注册三个过程。下面就以使用FileSystemXmlApplicationContext获取Bean的示例对这三个过程进行分析。  

_ _ _
### **Resource定位**
在使用FileSystemXmlApplicationContext获取Bean时，Spring IOC容器初始化是在new FileSystemXmlApplicationContext("beans.xml")过程中完成的。FileSystemXmlApplicationContext的继承图如下：  

<img src="/img/2018-8-8/FileSystemXmlApplicationContextExtends.png" width="700" height="700" alt="FileSystemXmlApplicationContext的继承图" />
<center>图2：FileSystemXmlApplicationContext的继承图</center>    

下面我们就来看看这个过程究竟做了什么：  
```java
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {

    // configLocation中包含的是BeanDefinition所在的文件路径
    public FileSystemXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
    // 这个构造函数在允许指定多个包含BeanDefinition的文件路径的同时，还允许指定自己的双亲IOC容器
    public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		super(parent); // 父类实例化，设置ResourcePatternResolver为PathMatchingResourcePatternResolver
		setConfigLocations(configLocations);
		if (refresh) {  
			refresh(); // 通过此方法启动IOC容器的初始化
		}
	}    
}

public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// 这里是在子类中启动refreshBeanFactory的地方，正是在refreshBeanFactory方法中完成了资源的定位
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 到这一步 资源定位、bean装载注册已经完成
			// Prepare the bean factory for use in this context. 为容器启动做一些准备工作
			prepareBeanFactory(beanFactory);

			try {
				// 设置BeanFactory的后置处理器
				postProcessBeanFactory(beanFactory);

				// 调用BeanFactory的后置处理，这些后置处理器都是在Bean定义中向容器注册的？？？怎么注册的？？？
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册Bean的后处理器，在Bean创建过程中调用
				registerBeanPostProcessors(beanFactory);

				// 对上下文中的消息源进行初始化
				initMessageSource();

				// 初始化上下文中的事件机制
				initApplicationEventMulticaster();

				// 初始化其它的特殊Bean
				onRefresh();

				// 检查监听Bean并且将这些Bean向容器注册
				registerListeners();

				// 实例化所有的(non-lazy-init)单例对象
				finishBeanFactoryInitialization(beanFactory);

		        // 发布容器事件，结束refresh过程
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// 为防止Bean资源占用，在异常处理中，销毁已经在前面过程中生成的单例Bean
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
		}
	}
}

```
refresh()方法执行过程图示如下：
<img src="/img/2018-8-8/refreshStep.jpg" width="700" height="700" alt="refresh()方法执行过程图" />
<center>图3：refresh()方法执行过程图</center>

接下来看下ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();中的obtainFreshBeanFactory()方法：
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}

    protected BeanFactory getInternalParentBeanFactory() {
		return (getParent() instanceof ConfigurableApplicationContext) ?
				((ConfigurableApplicationContext) getParent()).getBeanFactory() : getParent();
	}
}

public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
    
    @Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
            // 装载BeanDefinition
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}

    protected DefaultListableBeanFactory createBeanFactory() {
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}
}
```
接下来看定位资源并装载的方法loadBeanDefinitions(DefaultListableBeanFactory beanFactory)：
```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {

    @Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 创建一个 XmlBeanDefinitionReader 并通过回调方式配置给指定的 BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this); // 当前对象本身是ResourceLoader子类
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations(); // 获取传入的配置文件路径，完成资源定位
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
}

public abstract class AbstractBeanDefinitionReader implements EnvironmentCapable, BeanDefinitionReader {

    public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int counter = 0;
		for (Resource resource : resources) {
			counter += loadBeanDefinitions(resource);
		}
		return counter;
	}

    public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int counter = 0;
		for (String location : locations) {
			counter += loadBeanDefinitions(location);
		}
		return counter;
	}

    public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(location, null);
	}

    public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader(); // 这里的ResourceLoader其实就是FileSystemXmlApplicationContext对象
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) { // FileSystemXmlApplicationContext对象是ResourcePatternResolver类的实例
			// Resource pattern matching available.
			try {
                // 调用AbstractApplicationContext中getResources(String locationPattern)调用完成资源定位
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location); 

                // 调用XmlBeanDefinitionReader类的loadBeanDefinitions方法装载BeanDefinitions
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
            // 调用XmlBeanDefinitionReader类的loadBeanDefinitions方法
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
}
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    public Resource[] getResources(String locationPattern) throws IOException {
        // 调用PathMatchingResourcePatternResolver中locationPattern定位资源
		return this.resourcePatternResolver.getResources(locationPattern);
	}
}

public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {

    public Resource[] getResources(String locationPattern) throws IOException {
		Assert.notNull(locationPattern, "Location pattern must not be null");
		if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
			// a class path resource (multiple resources for same name possible)
			if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
				// a class path resource pattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// all class path resources with the given name
				return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
			}
		}
		else {
			// Only look for a pattern after a prefix here
			// (to not get fooled by a pattern symbol in a strange prefix).
			int prefixEnd = locationPattern.indexOf(":") + 1;
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
				// a file pattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// a single resource with the given name
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}

    public Resource getResource(String location) {
		return getResourceLoader().getResource(location); 
	}
}

public class DefaultResourceLoader implements ResourceLoader {

    public Resource getResource(String location) {
		Assert.notNull(location, "Location must not be null");
		if (location.startsWith(CLASSPATH_URL_PREFIX)) {
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
		}
		else {
			try {
				// Try to parse the location as a URL...
				URL url = new URL(location);
				return new UrlResource(url);
			}
			catch (MalformedURLException ex) {
				// No URL -> resolve as resource path.
                // 回调到FileSystemXmlApplicationContext中的getResourceByPath方法
				return getResourceByPath(location);
			}
		}
	}
}

public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
    
    @Override
    protected Resource getResourceByPath(String path) {
		if (path != null && path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}
}
```

至此资源的定位完成，具体定位资源方法调用流程如下：  
<img src="/img/2018-8-8/resourceDefine.png" width="700" height="700" alt="资源定位方法调用流程图" />
<center>图4：资源定位方法调用流程图</center>

_ _ _
### **Bean载入**
接下来看Bean的装载过程：
```java
public abstract class AbstractBeanDefinitionReader implements EnvironmentCapable, BeanDefinitionReader {
    public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader(); // 这里的ResourceLoader其实就是FileSystemXmlApplicationContext对象
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) { // FileSystemXmlApplicationContext对象是ResourcePatternResolver类的实例
			// Resource pattern matching available.
			try {
                // 调用AbstractApplicationContext中getResources(String locationPattern)调用完成资源定位
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location); 

                // 调用XmlBeanDefinitionReader类的loadBeanDefinitions方法装载BeanDefinitions
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
            // 调用XmlBeanDefinitionReader类的loadBeanDefinitions方法
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
}

public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {

    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}

    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<EncodedResource>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                // 在doLoadBeanDefinitions方法中进行具体的读取过程
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}

    // 从特定的Xml文件中实际载入BeanDefinition
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			int validationMode = getValidationModeForResource(resource);
			Document doc = this.documentLoader.loadDocument(
					inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
            // registerBeanDefinitions启动的是对BeanDefinition解析的详细过程，其中会涉及到Spring的Bean配置规则
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
    // 返回的是从资源文件中读取的Bean的个数
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        // 默认得到一个DefaultBeanDefinitionDocumentReader类的对象，为具体的Spring Bean的解析准备好了数据
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		documentReader.setEnvironment(getEnvironment());
		int countBefore = getRegistry().getBeanDefinitionCount();

        // 具体解析过程在这个DefaultBeanDefinitionDocumentReader.registerBeanDefinitions中完成
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
}

public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
    
    public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		Element root = doc.getDocumentElement();
		doRegisterBeanDefinitions(root);
	}

    protected void doRegisterBeanDefinitions(Element root) {
		String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
		if (StringUtils.hasText(profileSpec)) {
			String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
					profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			if (!getEnvironment().acceptsProfiles(specifiedProfiles)) {
				return;
			}
		}

		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(this.readerContext, root, parent);

		preProcessXml(root);
        // 从xml文件根标签开始解析
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}

    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
            // 从根标签开始一个个进行解析
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}

	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            // 处理文件中的BeanDefinition信息
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}

    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        // parseBeanDefinitionElement()里面包含了对各种Spring Bean定义规则的处理，比如熟悉的BeanDefinition中的id，name等属性元素是
        // 如何处理的,解析完成之后会将解析结果放到BeanDefinition对象中并设置到BeanDefinitionHolder中去
        // BeanDefinitionHolder持有的BeanDefinition对象仅包含Bean定义的一系列信息，并不包含实际创建的Bean对象
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
}

public class BeanDefinitionParserDelegate {

    public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}
    // 解析<bean> 标签，完成之后会将解析结果放到BeanDefinition对象中并设置到BeanDefinitionHolder中去
    public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
		String id = ele.getAttribute(ID_ATTRIBUTE); // 这里可以看到对我们熟悉的id、name、alias等属性的处理
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		List<String> aliases = new ArrayList<String>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}

        // 这个方法会引发对Bean元素的详细解析
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
}
```
经过上述对XML文件的逐层解析，我们在XML文件中定义的BeanDefinition就被整个载入到了IOC容器中，并在容器中建立了数据映射。在IOC容器中建立了对应的数据结构，或者说可以看成是POJO对象在IOC容器中的抽象，这些数据结构可以以AbstractBeanDefinition为入口，让IOC容器执行索引，查询和操作。  

经过上述Resource定位和Bean装载的过程，IOC容器大致完成了管理Bean对象的数据准备工作，但是，重要的依赖注入过程实际上在这个时候还没有发生，现在，在IOC容器的BeanDefinition中存在的还只是一些静态的配置信息。要完全发挥容器的作用，还需要下面介绍的数据向容器中进行注册的过程。  

_ _ _
### **Bean注册**
对Bean的注册过程发生在对Bean解析之后：
```java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
    
    ... 
    
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        // parseBeanDefinitionElement()里面包含了对各种Spring Bean定义规则的处理，比如熟悉的BeanDefinition中的id，name等属性元素是
        // 如何处理的,解析完成之后会将解析结果放到BeanDefinition对象中并设置到BeanDefinitionHolder中去
        // BeanDefinitionHolder持有的BeanDefinition对象仅包含Bean定义的一系列信息，并不包含实际创建的Bean对象
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance. 注册最后装饰好的实例对象
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
}

public class BeanDefinitionReaderUtils {

    public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
        // 调用DefaultListableBeanFactory的registerBeanDefinition方法
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String aliase : aliases) {
				registry.registerAlias(beanName, aliase);
			}
		}
	}
}

public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
 
    /** Map of bean definition objects, keyed by bean name 实际通过一个HashMap来持有我们载入的BeanDefinition*/
	private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(64);

    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		synchronized (this.beanDefinitionMap) {
			oldBeanDefinition = this.beanDefinitionMap.get(beanName);
			if (oldBeanDefinition != null) { // 如果此Bean名称之前被定义过
				if (!this.allowBeanDefinitionOverriding) { // 不允许覆盖之前重名的定义的话则抛出异常
					throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
							"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
							"': There is already [" + oldBeanDefinition + "] bound.");
				}
				else {
					if (this.logger.isInfoEnabled()) {
						this.logger.info("Overriding bean definition for bean '" + beanName +
								"': replacing [" + oldBeanDefinition + "] with [" + beanDefinition + "]");
					}
				}
			}
			else {
				this.beanDefinitionNames.add(beanName);
				this.frozenBeanDefinitionNames = null;
			}
            // beanName作为map的key，BeanDefinition作为value存入map中
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
        // 之前与此bean同名的bean是单例的话，需要进行一些清理关系
		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}

    /**
	 * Reset all bean definition caches for the given bean,
	 * including the caches of beans that are derived from it.
	 * @param beanName the name of the bean to reset
	 */
	protected void resetBeanDefinition(String beanName) {
		// Remove the merged bean definition for the given bean, if already created.
		clearMergedBeanDefinition(beanName);

		// Remove corresponding bean from singleton cache, if any. Shouldn't usually
		// be necessary, rather just meant for overriding a context's default beans
		// (e.g. the default StaticMessageSource in a StaticApplicationContext).
		destroySingleton(beanName);
                
		// Reset all bean definitions that have the given bean as parent (recursively).
		for (String bdName : this.beanDefinitionNames) {
			if (!beanName.equals(bdName)) {
				BeanDefinition bd = this.beanDefinitionMap.get(bdName);
				if (beanName.equals(bd.getParentName())) {
					resetBeanDefinition(bdName);
				}
			}
		}
	}
}
```
在完成了上述BeanDefinition的注册过程之后，就完成了IOC容器的初始化过程，此时在ApplicationContext包装的实际的IOC容器DefaultListableBeanFactory中已经建立了整个Bean的配置信息，而且这些BeanDefinition已经可以被容器使用了，它们都在beanDefinitionMap中被检索和使用（Spring IOC容器初始化过程通常并不创建Bean对象，只是将Bean信息处理为BeanDefinition）。容器的作用就是对这些信息进行处理和维护。这些信息是容器建立依赖反转的基础，有了这些基础数据，才有接下来的依赖注入过程。    

关于Spring容器依赖注入过程，参见下一篇博客。  

_ _ _
### **refresh()中其它准备工作**
再来看下BeanDefinition注册完成之后进行的对容器的其它准备工作：  
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// 这里是在子类中启动refreshBeanFactory的地方，正是在refreshBeanFactory方法中完成了资源的定位
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // 到这一步 资源定位、bean装载注册已经完成
			// Prepare the bean factory for use in this context. 为容器启动做一些准备工作
			prepareBeanFactory(beanFactory);

			try {
				// 设置BeanFactory的后置处理器
				postProcessBeanFactory(beanFactory);

				// 调用BeanFactory的后置处理，这些后置处理器都是在Bean定义中向容器注册的？？？怎么注册的？？？
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册Bean的后处理器，在Bean创建过程中调用
				registerBeanPostProcessors(beanFactory);

				// 对上下文中的消息源进行初始化
				initMessageSource();

				// 初始化上下文中的事件机制
				initApplicationEventMulticaster();

				// 初始化其它的特殊Bean
				onRefresh();

				// 检查监听Bean并且将这些Bean向容器注册
				registerListeners();

				// 实例化所有的(non-lazy-init)单例对象
				finishBeanFactoryInitialization(beanFactory);

		        // 发布容器事件，结束refresh过程
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// 为防止Bean资源占用，在异常处理中，销毁已经在前面过程中生成的单例Bean
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}
		}
	}
}
```
#### **prepareBeanFactory**
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    /**
	 * Configure the factory's standard context characteristics,
	 * such as the context's ClassLoader and post-processors.
	 * @param beanFactory the BeanFactory to configure
	 * 对BeanFactory做一些准备工作
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver());
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
        // 注册ApplicationContextAwareProcessor，对继承自ApplicationContextAware的bean进行处理
        // 取消ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware、EnvironmentAware这5
        // 个接口的自动注入。因为ApplicationContextAwareProcessor把这5个接口的实现工作做了
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
}
```

#### **postProcessBeanFactory**
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {
    
    /**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for registering special
	 * BeanPostProcessors etc in certain ApplicationContext implementations.
	 * @param beanFactory the bean factory used by the application context
	 * 此方法可以用来在Application context标准初始化过程完成之后更改其内部bean factory的状态。
	 * 到目前为止，所有bean definitions都已经被加载，但是还没有bean被实例化。
	 * 可以在ApplicationContext子类实现中覆盖这个方法用来进行注册特殊的BeanPostProcessors等操作。
	 */
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	}
}
```
AnnotationConfigEmbeddedWebApplicationContext对应的postProcessBeanFactory方法：
```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  // 调用父类EmbeddedWebApplicationContext的实现
  super.postProcessBeanFactory(beanFactory);
  // 查看basePackages属性，如果设置了会使用ClassPathBeanDefinitionScanner去扫描basePackages包下的bean并注册
  if (this.basePackages != null && this.basePackages.length > 0) {
    this.scanner.scan(this.basePackages);
  }
  // 查看annotatedClasses属性，如果设置了会使用AnnotatedBeanDefinitionReader去注册这些bean
  if (this.annotatedClasses != null && this.annotatedClasses.length > 0) {
    this.reader.register(this.annotatedClasses);
  }
}
```
父类EmbeddedWebApplicationContext的实现：
```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  beanFactory.addBeanPostProcessor(
      new WebApplicationContextServletContextAwareProcessor(this));
  beanFactory.ignoreDependencyInterface(ServletContextAware.class);
}
```

#### **invokeBeanFactoryPostProcessors**
调用在context中注册为bean的factory processors，我的理解：应用中的BeanFactoryPostProcessors来源可以有两种:  
1. 容器初始化时注册的一些BeanFactoryPostProcessor，比如基于web程序的Spring容器AnnotationConfigEmbeddedWebApplicationContext构造的时候，会初始化内部属性AnnotatedBeanDefinitionReader reader，这个reader构造的时候会在BeanFactory中注册一些post processor，包括BeanPostProcessor和BeanFactoryPostProcessor；(比如ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor)
2. 我们自己在Bean定义文件中自定义的BeanFactoryPostProcessor。  

invokeBeanFactoryPostProcessors方法中首先对容器注册的BeanFactoryPostProcessor进行处理，比如ConfigurationClassPostProcessor(这个processor是优先级最高的被执行的processor)；ConfigurationClassPostProcessor会去BeanFactory中找出所有有@Configuration注解的bean，然后使用ConfigurationClassParser去解析这个类。解析过程中如果发现程序中有自定义的BeanFactoryPostProcessor，那么这个PostProcessor就会通过ConfigurationClassPostProcessor被解析出来，然后在第二步中被Spring容器找到并执行。  
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    /**
	 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before singleton instantiation.
	 * 
	 * 实例化所有注册的并且BeanFactoryPostProcessor beans，并调用其postProcessBeanFactory方法。  
	 * 必须在singleton对象实例化之前被调用。  
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<String>();
        /**
         * 先找容器初始化时注册的一些BeanFactoryPostProcessor，分为两种：
         *   1. BeanFactoryPostProcessor：用来修改Spring容器中已经存在的bean的定义，使用ConfigurableListableBeanFactory对bean进行处理
         *   2. BeanDefinitionRegistryPostProcessor：继承BeanFactoryPostProcessor，作用跟BeanFactoryPostProcessor一样，只不过是使用
         *      BeanDefinitionRegistry对bean进行处理
         */
		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
			List<BeanDefinitionRegistryPostProcessor> registryPostProcessors =
					new LinkedList<BeanDefinitionRegistryPostProcessor>();
			for (BeanFactoryPostProcessor postProcessor : getBeanFactoryPostProcessors()) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryPostProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryPostProcessor.postProcessBeanDefinitionRegistry(registry);
					registryPostProcessors.add(registryPostProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}
			Map<String, BeanDefinitionRegistryPostProcessor> beanMap =
					beanFactory.getBeansOfType(BeanDefinitionRegistryPostProcessor.class, true, false);
			List<BeanDefinitionRegistryPostProcessor> registryPostProcessorBeans =
					new ArrayList<BeanDefinitionRegistryPostProcessor>(beanMap.values());
			OrderComparator.sort(registryPostProcessorBeans);
			for (BeanDefinitionRegistryPostProcessor postProcessor : registryPostProcessorBeans) {
				postProcessor.postProcessBeanDefinitionRegistry(registry);
			}
			invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(registryPostProcessorBeans, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
			processedBeans.addAll(beanMap.keySet());
		}
		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(getBeanFactoryPostProcessors(), beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
        // 找出所有在程序中自定义的BeanFactoryPostProcessor的实现
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
        // 把BeanFactoryPostProcessor按照优先级分为三个list分别进行处理(实现priorityOrdered接口的、实现Ordered接口的、没有实现Ordered接口的)
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (isTypeMatch(ppName, PriorityOrdered.class)) {
                // 首先对实现priorityOrdered接口的BeanFactoryPostProcessor实例化
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		OrderComparator.sort(priorityOrderedPostProcessors);
        // 首先处理实现priorityOrdered接口的BeanFactoryPostProcessors
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : orderedPostProcessorNames) {
            // 对实现Ordered接口的BeanFactoryPostProcessor实例化
			orderedPostProcessors.add(getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		OrderComparator.sort(orderedPostProcessors);
        // 处理实现Ordered接口的BeanFactoryPostProcessors
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
        // 对未实现ordered接口的BeanFactoryPostProcessor实例化
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
        // 处理未实现Ordered接口的BeanFactoryPostProcessors
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
	}

    /**
	 * Invoke the given BeanFactoryPostProcessor beans.
	 * 调用BeanFactoryPostProcessor.postProcessBeanFactory进行处理
	 */
	private void invokeBeanFactoryPostProcessors(
			Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

		for (BeanFactoryPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessBeanFactory(beanFactory);
		}
	}
}
```
分析上述代码，可以确定的BeanFactoryPostProcessor回调方法执行顺序是：  
1. 执行系统预定义的BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry方法。  
2. 执行我们自定义的BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry方法。  
3. 执行系统预定义的BeanDefinitionRegistryPostProcessor.postProcessBeanFactory方法。
4. 执行我们自定义的BeanDefinitionRegistryPostProcessor.postProcessBeanFactory方法。
5. 执行系统预定义的BeanFactoryPostProcessor.postProcessBeanFactory方法。
6. 执行我们自定义的BeanFactoryPostProcessor.postProcessBeanFactory方法(根据实现的Ordered接口和配置文件中的顺序决定执行顺序)。  

#### **registerBeanPostProcessors**
注册拦截bean创建过程的BeanPostProcessors：  
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    /**
	 * Instantiate and invoke all registered BeanPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before any instantiation of application beans.
	 * 
	 * 完成的功能其实就是实例化并且注册所有的BeanPostProcessor
	 * 
	 * BeanPostProcessor可以分为容器启动时内部需要注册的和程序中自定义注册的
	 */
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (isTypeMatch(ppName, PriorityOrdered.class)) {
                // 最先实例化优先级最高的BeanPostProcessors
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		OrderComparator.sort(priorityOrderedPostProcessors);
        // 注册实现PriorityOrdered的BeanPostProcessors，其余以此类推
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		OrderComparator.sort(orderedPostProcessors);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		OrderComparator.sort(internalPostProcessors);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector());
	}

    /**
	 * Register the given BeanPostProcessor beans.
	 */
	private void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

		for (BeanPostProcessor postProcessor : postProcessors) {
			beanFactory.addBeanPostProcessor(postProcessor);
		}
	}

}
```

#### **initMessageSource**
在Spring容器中初始化一些国际化相关的属性。  

#### **initApplicationEventMulticaster** 
在Spring容器中初始化事件广播器，事件广播器用于事件的发布。  

#### **onRefresh**  
在不同的子类实现中初始化不同的bean，是一个模板方法，不同的Spring容器做不同的事情。  

比如web程序的容器AnnotationConfigEmbeddedWebApplicationContext中会调用createEmbeddedServletContainer方法去创建内置的Servlet容器(Tomcat、Jetty)。  

#### **registerListeners**  
添加作为监听器使用的ApplicationListener：  
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    /**
	 * Add beans that implement ApplicationListener as listeners.
	 * Doesn't affect other listeners, which can be added without being beans.
	 */
	protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}
	}
}
```

#### **finishBeanFactoryInitialization**
初始化所有存在的单例且lazy-init为false的bean：  
```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {

    /**
	 * Finish the initialization of this context's bean factory,
	 * initializing all remaining singleton beans.
	 * 结束这个context对应的bean factory的实例化过程，初始化所有存在的单例且lazy-init为false的bean
	 * 在实例化这些bean的时候，各种BeanPostProcessor开始起作用。
	 */
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
        // 初始化所有存在的单例且lazy-init为false的bean
		beanFactory.preInstantiateSingletons();
	}
}

public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {

    public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isInfoEnabled()) {
			this.logger.info("Pre-instantiating singletons in " + this);
		}

		List<String> beanNames;
		synchronized (this.beanDefinitionMap) {
			// Iterate over a copy to allow for init methods which in turn register new bean definitions.
			// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
			beanNames = new ArrayList<String>(this.beanDefinitionNames);
		}

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
	}
}
```

#### **finishRefresh**
处理一些refresh做完之后需要做的其他事情。   

**通过上面对refresh()方法的分析过程可以看到，有几种类型bean的依赖注入发生在IOC容器初始化的过程中，而不像一般的依赖注入一样发生在IOC容器初始化完成以后，第一次执行getBean时。这几种bean类型包含：BeanFactoryPostProcessor实现类、BeanPostProcessor实现类、作用域为singleton且lazy-init属性为false的普通bean。**    

  
(完)

参考：
《Spring技术内幕》  
[【Spring】IOC核心源码学习（二）：容器初始化过程](http://singleant.iteye.com/blog/1177358)
[SpringBoot源码分析之Spring容器的refresh过程](https://fangjian0423.github.io/2017/05/10/springboot-context-refresh/)  
