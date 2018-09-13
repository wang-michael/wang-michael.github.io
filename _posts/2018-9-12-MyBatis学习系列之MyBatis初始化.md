---
layout:     post
title:      MyBatis学习系列之MyBatis初始化
date:       2018-9-12
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - MyBatis
---
>本文记录我对MyBatis初始化相关过程及注意事项的理解(基于MyBatis 3.4版本)。     

_ _ _
## **前言**
MyBatis中的配置文件中主要包含两种，分别是mybatis-config.xml配置文件和映射配置文件(mapper.xml)。  

在MyBatis初始化的过程中，除了会读取mybatis-config.xml配置文件以及映射配置文件，还会加载配置文件中指定的类，处理类中的注解，创建一些配置对象，最终完成框架中各个模块的初始化。接下来就以下面这个Demo为例介绍MyBatis的初始化过程。  
```java
mybatis-config.xml:

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <settings>
        <!--设置全局自动映射级别-->
        <!-- PARTIAL与FULL的区别在于是否映射嵌套关系 -->
        <setting name="autoMappingBehavior" value="PARTIAL"/>
        <!--<setting name="autoMappingBehavior" value="FULL"/>-->
        <!--<setting name="autoMappingBehavior" value="NONE"/>-->
    </settings>

    <typeAliases>
        <!--<typeAlias type="com.michael.mybatisStudy.model.User" alias="User"/>-->
        <package name="com.michael.mybatisStudy.model"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"> <!-- 声明使用那种事务管理机制 JDBC/MANAGED -->
                <property name="autoCommit" value="false"/>
            </transactionManager>
            <!-- 配置数据库连接信息 -->
            <!--POOLED代表使用MyBatis提供的org.apache.ibatis.datasource.pooled.PooledDataSource实现-->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/mybatisStudy?characterEncoding=utf8&amp;allowMultiQueries=yes" />
                <property name="username" value="" />
                <property name="password" value="" />
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="com\michael\mybatisStudy\mapper\UserMapper.xml"/>
    </mappers>

</configuration>

public class MyBatisUtil {

    public static SqlSessionFactory getSqlSessionFactory() {
        return sqlSessionFactory;
    }

    private static SqlSessionFactory sqlSessionFactory = null;

    static {
        String resource = "configuration.xml";
        try {
            // 通过SqlSessionFactoryBuilder来获取SqlSessionFactory
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(
                    Resources.getResourceAsStream(resource)
            );
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
MyBatis的初始化过程就是从sqlSessionFactory的创建开始的，我们就从这里开始分析。  

_ _ _
## **初始化过程分析**

### **mybatis-config.xml文件解析**
SqlSessionFactory对象通常在MyBatis应用的整个生命周期中只需要一个即可，其中含有一个成员变量是Configuration类对象，用来总揽Mybatis框架中所有配置。这个Configuration对象中的内容就是在解析配置文件的过程中填充的，下面就先来看用来解析配置文件的XMLConfigBuilder对象创建：   
```java
public class SqlSessionFactoryBuilder {

  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      // 解析mybatis-config.xml配置文件的入口
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }

  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
}

// 建造者模式，具体的建造者，BaseBuilder抽象类扮演着建造者接口的角色
// 一个sqlSessionFactory对应一个XMLConfigBuilder对象
public class XMLConfigBuilder extends BaseBuilder {
  
  // 标识是否解析过mybatis-config.xml文件
  private boolean parsed;
  // 用于解析mybatis-config.xml配置文件的XPathParser对象
  private final XPathParser parser;
  // 标识<environment>配置的名称，默认读取<environment>标签的default属性
  private String environment;
  // ReflectorFactory用来创建和缓存Reflector对象
  private final ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();

  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
}

public abstract class BaseBuilder {

  // configuration是mybatis初始化过程中的核心对象，mybatis中几乎所有的配置信息会保存到Configuration对象中
  protected final Configuration configuration;
  // 在mybatis-config.xml中可以使用<typeAliases>标签定义别名，这些定义的别名都会记录在该对象中
  protected final TypeAliasRegistry typeAliasRegistry;
  // 在mybatis-config.xml配置文件中可以使用<typeHandler>标签添加自定义TypeHandler器，这些TypeHandler都会记录在
  // typeHandlerRegistry中
  protected final TypeHandlerRegistry typeHandlerRegistry;

  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
}
```
接下来我们继续看XMLConfigBuilder.parse()方法来分析下整个解析过程：
```java
public class XMLConfigBuilder extends BaseBuilder {

  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    // 返回配置好的configuration对象
    return configuration;
  }

  // 从根节点<configuration>开始对每种子节点进行解析
  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
      // 解析<properties节点>
      propertiesElement(root.evalNode("properties"));
      // 解析<settings>节点
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings); // 将settings值设置到Configuration中
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      // 解析mapper节点
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }

  private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      Properties defaults = context.getChildrenAsProperties();
      String resource = context.getStringAttribute("resource");
      String url = context.getStringAttribute("url");
      if (resource != null && url != null) {
        throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
      }
      if (resource != null) {
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      Properties vars = configuration.getVariables();
      if (vars != null) {
        defaults.putAll(vars);
      }
      parser.setVariables(defaults);
      // 将解析好的属性填充到configuration当中去
      configuration.setVariables(defaults);
    }
  }

  // 解析<mapper>节点，每个mapper.xml文件对应一个XMLMapperBuilder对象，每个XMLMapperBuilder对应一个MapperBuilderAssistant对象
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {

        /** 1.第一种配置映射文件的方式
         *  <package name="映射文件所在包名">
         *  注意：这种方式必须保证接口名（例如IUserDao）和xml名（IUserDao.xml）相同，还必须在同一个包中。
         */
        if ("package".equals(child.getName())) {
          // 扫描指定的包，并向MapperRegistry注册Mapper接口
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        }
        /** 第二种方式：<mapper resource="">
         *  这种方式不用保证同接口同包同名。例如：<mapper resource="cn/sdut/pojo/PersonMapper.xml"/>
         */    
        else {
          // 获取<mapper>节点的resource、url、class属性，这三个属性互斥
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          // 如果mapper节点指定了resource或是url属性，则创建XMLMapperBuilder对象，并通过该对象解析
          // resource或是url属性指定的mapper配置文件
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);

            // 创建XMLMapperBuilder对象，解析映射配置文件
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();

          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);

            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();

          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
}
```

### mapper.xml配置文件解析

在上面的代码中在XMLConfigBuilder.parseConfiguration方法中依次对mybatis-config.xml中各个子节点进行了解析，对每种子节点的解析过程封装成了一个方法，下面来详细分析下在mapperElement(XNode parent)方法中对于mapper.xml映射文件的解析过程：  
```java
// 每个mapper.xml文件使用一个相应的XMLMapperBuilder对象进行解析，每个XMLMapperBuilder对应一个MapperBuilderAssistant对象
public class XMLMapperBuilder extends BaseBuilder {

  public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
        configuration, resource, sqlFragments);
  }

  private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = parser;
    this.sqlFragments = sqlFragments;
    this.resource = resource;
  }

  public void parse() {
    // 判断是否已经加载过此mapper.xml文件
    if (!configuration.isResourceLoaded(resource)) {
      
      configurationElement(parser.evalNode("/mapper")); // 处理<mapper>节点及下面各个子节点
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }

  /**
   * mapper.xml中可能出现的节点共有以下几种情况：<cache>、<cache-ref>、<parameterMap>、<resultMap>、<sql>、
   * <select>、<update>、<delete>、<insert>
   * 
   * XMLMapperBuilder与XMLConfigBuilder思想类似，也是将每个节点的解析过程封装成了一个方法，而这些方法由
   * XMLMapperBuilder.configurationElement()方法调用
   */
  private void configurationElement(XNode context) {
    try {
      // 获取<mapper>节点的namespace属性
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      // 通过MapperBuilderAssistant的currentNamespace字段，记录当前命名空间(即mapper.xml中<mapper>标签的nameSpace)
      builderAssistant.setCurrentNamespace(namespace);
      // 解析cache-ref节点
      cacheRefElement(context.evalNode("cache-ref"));
      // 解析cache节点
      cacheElement(context.evalNode("cache"));
      // 解析parameterMap节点(该节点已经废弃，不再推荐使用)
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // 解析resultMap节点
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // 解析sql节点
      sqlElement(context.evalNodes("/mapper/sql"));
      // 解析<select>、<insert>、<update>、<delete>等sql节点
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }  

}
```
接下来就分别来看对mapper.xml中各个节点的解析过程：  

#### mapper.xml中<cache>节点解析：
```java
public class XMLMapperBuilder extends BaseBuilder {

  // mybaits的二级缓存是mapper范围级别，除了在SqlMapConfig.xml设置二级缓存的总开关，还要在具体的mapper.xml中开启二级缓存。
  // 负责解析<cache>节点
  private void cacheElement(XNode context) throws Exception {
    if (context != null) {
      // 默认的缓存实现类是PerpetualCache，使用HashMap实现的缓存
      // 还可以在mapper.xml中通过type属性指定实现encache缓存，比如<cache type="org.mybatis.caches.ehcache.EhcacheCache">
      String type = context.getStringAttribute("type", "PERPETUAL");
      
      // 获取<cache>节点内部的一些属性
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      // 获取缓存清理原则，默认是LRU
      String eviction = context.getStringAttribute("eviction", "LRU");
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      Long flushInterval = context.getLongAttribute("flushInterval");
      Integer size = context.getIntAttribute("size");
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      boolean blocking = context.getBooleanAttribute("blocking", false);

      /**
       * 获取<cache>节点下的子节点，将用于初始化
       * 
       * 比如：  
       * <cache type="org.mybatis.caches.ehcache.EhcacheCache" >
       *   <property name="timeToIdleSeconds" value="3600"/>
       *   <property name="timeToLiveSeconds" value="3600"/>
       *   <!-- 同ehcache参数maxElementsInMemory -->
       *   <property name="maxEntriesLocalHeap" value="1000"/>
       *   <!-- 同ehcache参数maxElementsOnDisk -->
       *   <property name="maxEntriesLocalDisk" value="10000000"/>
       *   <property name="memoryStoreEvictionPolicy" value="LRU"/>
       * </cache>
       */
      Properties props = context.getChildrenAsProperties();
      // 通过MapperBuilderAssistant创建cache对象，并添加到Configuration.caches集合中保存
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }
}
```
MapperBuilderAssistant是一个辅助类，其useNewCache方法负责创建cache对象，并将其添加到Configuration.caches集合中保存。Configuration中的caches字段是一个map，其key值是cache的id(默认是映射文件的namespace)，value是与此namespace对应的二级缓存对象。下面就来看看Cache对象的具体创建过程：  
```java
public class MapperBuilderAssistant extends BaseBuilder {
  public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();

    // 添加建立好的cache到Configuration中
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }
}

public class CacheBuilder {
  private final String id; // Cache对象标识，一般对应mapper.xml文件中的namespace
  private Class<? extends Cache> implementation; // Cache接口的真正实现类，默认对应前面介绍的PerpetualCache
  private final List<Class<? extends Cache>> decorators; // 装饰器集合，默认只包含LruCache.class
  private Integer size; // Cache大小
  private Long clearInterval; // Cache清理时间周期
  private boolean readWrite; // 是否可读写
  private Properties properties; // 其它配置信息
  private boolean blocking; // 是否阻塞

  public CacheBuilder(String id) {
    this.id = id;
    this.decorators = new ArrayList<Class<? extends Cache>>();
  } 

  // 参数设置好之后创建真正的Cache对象
  public Cache build() {
    setDefaultImplementations();
    // 根据implementation指定的类型，通过反射获取参数为String类型的构造方法，并通过该构造方法构建cache对象
    Cache cache = newBaseCacheInstance(implementation, id);
    setCacheProperties(cache);
    // issue #352, do not apply decorators to custom caches
    // 检查cache对象的类型，如果是PerpetualCache类型，则为其添加decorators集合中的装饰器；
    // 如果是自定义类型的Cache接口实现，则不添加decorators集合中的装饰器(比如整合的外部encache缓存)
    if (PerpetualCache.class.equals(cache.getClass())) {     
      for (Class<? extends Cache> decorator : decorators) {
        // 反射创建cache对象并添加装饰器
        cache = newCacheDecoratorInstance(decorator, cache);
        setCacheProperties(cache);
      }
      // 添加MyBatis中提供的标准装饰器
      cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
      cache = new LoggingCache(cache);
    }
    return cache;
  }

  private void setDefaultImplementations() {
    if (implementation == null) {
      implementation = PerpetualCache.class;
      if (decorators.isEmpty()) {
        decorators.add(LruCache.class);
      }
    }
  }

  // 添加MyBatis中提供的标准装饰器
  private Cache setStandardDecorators(Cache cache) {
    try {
      MetaObject metaCache = SystemMetaObject.forObject(cache);
      if (size != null && metaCache.hasSetter("size")) {
        metaCache.setValue("size", size);
      }
      if (clearInterval != null) {
        cache = new ScheduledCache(cache);
        ((ScheduledCache) cache).setClearInterval(clearInterval);
      }
      if (readWrite) {
        cache = new SerializedCache(cache);
      }
      cache = new LoggingCache(cache);
      // 使用SynchronizedCache进行装饰，保证二级缓存的线程安全性
      cache = new SynchronizedCache(cache);
      if (blocking) {
        cache = new BlockingCache(cache);
      }
      return cache;
    } catch (Exception e) {
      throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
    }
  }
  
}
```
经过上面的分析过程我们知道二级缓存对象与mapper.xml文件中的namespace一一对应，其对应关系保存在Configuration.caches集合中：  
```java
public class Configuration {
  protected final Map<String, Cache> caches = new StrictMap<Cache>("Caches collection");

  public void addCache(Cache cache) {
    // cache的id即为mapper.xml文件中的namespace
    caches.put(cache.getId(), cache);
  }
}
```

#### mapper.xml中<cache-ref>节点解析
cache-ref使用示例：<cache-ref namespace=”com.someone.application.data.SomeMapper”/> 作用是与另一个nameSpace共用一个cache
```java
public class XMLMapperBuilder extends BaseBuilder {

  private void cacheRefElement(XNode context) {
    if (context != null) {
      configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
      CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
      try {
        cacheRefResolver.resolveCacheRef();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteCacheRef(cacheRefResolver);
      }
    }
  }
}

public class CacheRefResolver {
  private final MapperBuilderAssistant assistant;
  private final String cacheRefNamespace;

  public CacheRefResolver(MapperBuilderAssistant assistant, String cacheRefNamespace) {
    this.assistant = assistant;
    this.cacheRefNamespace = cacheRefNamespace;
  }

  public Cache resolveCacheRef() {
    return assistant.useCacheRef(cacheRefNamespace);
  }
}

public class MapperBuilderAssistant extends BaseBuilder {

  public Cache useCacheRef(String namespace) {
    if (namespace == null) {
      throw new BuilderException("cache-ref element requires a namespace attribute.");
    }
    try {
      unresolvedCacheRef = true;
      // 确保当前cache-ref对应的namespace对应的cache确实存在
      Cache cache = configuration.getCache(namespace);
      if (cache == null) {
        throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.");
      }
      currentCache = cache;
      unresolvedCacheRef = false;
      return cache;
    } catch (IllegalArgumentException e) {
      throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.", e);
    }
  }
}
```

#### mapper.xml中select、update、delete、insert节点解析
```java
public class XMLMapperBuilder extends BaseBuilder {
  
  // 参数是<select>、<update>、<delete>、<insert>等节点构成的集合
  private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
  }

  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      // 对于每个XNode由一个XMLStatementBuilder来解析
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        // 实际节点解析过程
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
}

public class XMLStatementBuilder extends BaseBuilder {

  private final MapperBuilderAssistant builderAssistant;
  private final XNode context;
  private final String requiredDatabaseId;

  // 参数中的XNode是<select>、<update>、<delete>、<insert>其中之一
  public XMLStatementBuilder(Configuration configuration, MapperBuilderAssistant builderAssistant, XNode context, String databaseId) {
    super(configuration);
    this.builderAssistant = builderAssistant;
    this.context = context;
    this.requiredDatabaseId = databaseId;
  }

  public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    // 获取节点的各个属性
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    Class<?> resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    // statementType默认使用PREPARED
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);
    
    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    // MyBatis使用SqlSource接口标识映射文件或者注解中定义的SQL语句，可以使用其getBoundSql方法获取可执行的sql
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
}

public class MapperBuilderAssistant extends BaseBuilder {

  public MappedStatement addMappedStatement(
      String id,
      SqlSource sqlSource,
      StatementType statementType,
      SqlCommandType sqlCommandType,
      Integer fetchSize,
      Integer timeout,
      String parameterMap,
      Class<?> parameterType,
      String resultMap,
      Class<?> resultType,
      ResultSetType resultSetType,
      boolean flushCache,
      boolean useCache,
      boolean resultOrdered,
      KeyGenerator keyGenerator,
      String keyProperty,
      String keyColumn,
      String databaseId,
      LanguageDriver lang,
      String resultSets) {

    if (unresolvedCacheRef) {
      throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        .cache(currentCache);

    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
      statementBuilder.parameterMap(statementParameterMap);
    }
    // 把sql(增删改查)节点封装成相应的MappedStatement
    MappedStatement statement = statementBuilder.build();
    // 将MappedStatement保存到全局configuration对象中
    configuration.addMappedStatement(statement);
    return statement;
  }
}

public final class MappedStatement {
  
  private String resource; // 节点中的id属性(命名空间前缀 + <select> 等标签中的id属性)
  private SqlSource sqlSource; // SqlSource对象，其中绑定了一条SQL语句
  private SqlCommandType sqlCommandType; // sql的类型，Insert、update、delete、selecet或者flush
  ...
  
  public static class Builder {
    
    public MappedStatement build() {
      assert mappedStatement.configuration != null;
      assert mappedStatement.id != null;
      assert mappedStatement.sqlSource != null;
      assert mappedStatement.lang != null;
      mappedStatement.resultMaps = Collections.unmodifiableList(mappedStatement.resultMaps);
      return mappedStatement;
    }
  }
}
```
可见处理之后是将各个select、update、delete、insert节点封装成了MappedStatement保存到configuration对象中：
```java
public class Configuration {

  // key值为mapper.xml文件中的namespace + <select>、<update>、<delete>、<insert>中的id属性
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
  
  public void addMappedStatement(MappedStatement ms) {
    mappedStatements.put(ms.getId(), ms);
  }
}
```

### 绑定此mapper.xml配置文件与对应接口之间的关系
将mapper.xml中各个子节点注册到configuration对象中之后，就要绑定此mapper.xml配置文件与对应接口之间的关系：  
```java
public class XMLMapperBuilder extends BaseBuilder {

  public XMLMapperBuilder(InputStream inputStream, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    this(new XPathParser(inputStream, true, configuration.getVariables(), new XMLMapperEntityResolver()),
        configuration, resource, sqlFragments);
  }

  private XMLMapperBuilder(XPathParser parser, Configuration configuration, String resource, Map<String, XNode> sqlFragments) {
    super(configuration);
    this.builderAssistant = new MapperBuilderAssistant(configuration, resource);
    this.parser = parser;
    this.sqlFragments = sqlFragments;
    this.resource = resource;
  }

  public void parse() {
    // 判断是否已经加载过此mapper.xml文件
    if (!configuration.isResourceLoaded(resource)) {

      configurationElement(parser.evalNode("/mapper")); // 处理<mapper>节点及下面各个子节点

      // 将resource添加到Configuration.loadResources集合中保存，它是HashSet<String>
      // 类型的集合，其中记录了已经加载过的映射文件
      configuration.addLoadedResource(resource);
      // 向Configuration.MapperRegistry中注册mapper接口与MapperProxyFactory之间的对应关系
      bindMapperForNamespace();
    }
    
    // 处理mapper.xml中解析失败的节点，解析失败的原因可能是在解析一个节点时，会引用定义在该节点之后的、还未解析的节点

    // 处理configurationElement中解析失败的<resultMap>节点
    parsePendingResultMaps();
    // 处理configurationElement中解析失败的<cache-ref>节点
    parsePendingCacheRefs();
    // 处理configurationElement中解析失败的SQL语句节点
    parsePendingStatements();
  }

  private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        //ignore, bound type is not required
      }
      if (boundType != null) {
        if (!configuration.hasMapper(boundType)) {
          // Spring may not know the real resource name so we set a flag
          // to prevent loading again this resource from the mapper interface
          // look at MapperAnnotationBuilder#loadXmlResource
          configuration.addLoadedResource("namespace:" + namespace);
          // 向configuration对象中注册mapper.xml与对应的接口间的关系
          // 还有一个作用就是解析Mapper接口中的注解信息
          configuration.addMapper(boundType);
        }
      }
    }
  }

}
```
可见mapper.xml与对应接口之间的关系也是保存在Configuration对象之中的：  
```java
public class Configuration {

  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);

  public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
  }
}

public class MapperRegistry {

  // key值为mapper.xml中namespace对应的接口，value为与其对应的MapperProxyFactory对象
  // 之后使用MapperProxyFactory为此接口产生动态代理对象实现
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();

  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
}
```
到此为止，MyBatis的初始化过程就全部介绍完了，主要分析了对于mybatis-config.xml配置文件的解析过程、映射配置文件的解析过程。在这个分析过程中我们可以发现几乎所有解析后的配置都保存在Configuration对象中，所以之后我们使用SqlSessionFactory创建SqlSession对象时只需要向其中传入一个Configuration对象即可。  

(完)  

参考文章：《MyBatis技术内幕》  
