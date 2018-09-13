MyBatis学习笔记
### Hibernate差在哪
Hibernate最致命的问题是性能，它屏蔽了SQL，意味着只能全表映射，但一张表可能有几十到上百个字段，而我们感兴趣的只有两个，这是Hibernate无法适应的。尤其在大型网站系统，对传输数据有严格规定，不能浪费带宽的场景下就更为明显了。

总结一下Hibernate的缺点：
* 全表映射带来的不便，比如更新一个字段，也需要发送所有的字段
* 无法根据不同的条件组装不同的SQL
* 对多表关联和复杂SQL查询支持较差，需要自己写SQL，返回后需要自己将数据组装为POJO
* 不能有效支持存储过程
* 虽然有HQL，但是性能较差。大型互联网公司系统往往需要优化SQL，而Hibernate做不到

在当今大型互联网公司中，灵活、SQL优化，减少数据的传输是最基本的优化方法，显然Hibernate无法满足我们的要求。这时Mybatis框架诞生了，它提供了更灵活、更方便的方法，弥补了Hibernate的这些缺陷。  

### MyBatis学习
特点：
* 半自动映射：需要手工匹配POJO、SQL和映射关系，而全表映射的Hibernate只需要提供POJO和映射关系即可

MyBatis具有高度灵活、可优化、易维护等特点，所以它目前是大型移动互联网项目的首选框架。  

### 核心组件
* SqlSessionFactoryBuilder(构造器)：它会根据配置信息或者代码来生成SqlSessionFactory
* SqlSessionFactory：依靠工厂来生成SqlSession
* SqlSession：是一个既可以发送SQL去执行并返回结果，也可以获取Mapper的接口，类似于一个JDBC中的Connection接口对象
* SQL Mapper：它是MyBatis新设计的组件，是由一个Java接口和XML文件(或注解)构成的，需要给出对应的SQL和映射规则。它负责发送SQL去执行，并返回结果。  

MyBatis使用流程：
1. 读取MyBatis-configuration.xml文件创建Configuration对象，注入到SqlSessionFactoryBuilder对象当中
2. 通过SqlSessionFactoryBuilder对象创建SqlSessionFactory对象(默认实现类为DefaultSqlSessionFactory)，正确的做法是每个数据库只对应一个SqlSessionFactory(单例模式)，管理好数据库资源的分配，避免过多的Connection被消耗(每次创建SqlSessionFactory会打开更多的数据库连接)
3. 通过SqlSessionFactory创建SqlSession

#### SqlSession具体介绍
通过SqlSessionFactory获取的SqlSession类似于一个JDBC中的Connection接口对象，每次用完要注意关闭它。(finally块中关闭)
SqlSession的生命周期应该是在请求数据库处理事务的过程中，它是一个线程不安全的对象，使用后需要及时关闭。  

它存活于一个应用的请求和操作，可以执行多条SQL，保证事务的一致性。可以说应该让一个SqlSession工作于Service层，一个SqlSession实例对象调用的多个Mapper接口的方法被包裹在同一个事务当中。    


映射器是由Java接口和XML文件共同组成的，它的作用如下：
* 定义参数类型
* 描述缓存
* 描述SQL语句
* 定义查询结果和POJO的映射关系

#### cache
默认开启一级缓存，二级缓存是不开启的。  

一级缓存：仅在同一个sqlSession中共享缓存数据
二级缓存：在sqlSessionFactory层面创建的缓存，不同sqlSession之间也可以共享(只有在sqlSession commit之后其查询的数据才会被缓存吗？？？)

开启二级缓存时，MyBatis要求返回的POJO必须是可序列化的，也就是说要求实现serializable接口，配置的方法很简单。只需要在Configuration.xml中增加<cache/>标签就ok了。  

<cache/>标签中有很多设置是默认的，比如：
* 映射语句文件中的所有select语句将会被缓存
* 映射语句文件中的所有insert、update和delete语句会刷新缓存
* 缓存会使用默认的LRU算法来回收
* ...

当然也可以修改<cache/>标签中的属性来达到修改缓存属性的目的，比如：
<cache eviction = "LRU" flushInterval = "100000" size = "1024" readOnly = "true"/>

#### 其它零碎知识
1、MyBatis传递多个参数的常用方式：
* 使用Map传递参数 parameterType = "map"
* 使用注解方式(@Param)传递参数
* 使用JavaBean传递参数 parameterType = "JavaBean具体路径"

2、MyBatis中使用resultMap进行级联
* association,代表一对一关系，比如中国公民和身份证是一对一的关系
* collection：代表一对多关系，比如班级和学生
* discriminator：是鉴别器，它可以根据实际选择采用哪个类作为实例，允许你根据特定的条件去关联不同的结果集

使用方式：
```java
<resultMap>
    <association...   />
</resultMap>
```

3、MyBatis的settings标签中的lazyLoadingEnabled(延迟加载)和aggressiveLazyLoading(是否采用层级策略延迟加载)；
* lazyLoadingEnabled：默认为false。如果为false，则所有相关联的对象属性都会被初始化加载。
* aggressiveLazyLoading：默认为true。当设置为true时，当一个懒加载属性被加载时，与其同层次的懒加载属性也会被加载(可能还没有用到)。

比如学生对象中有两个属性引用了两个其它的对象(课程成绩对象、学生证对象)均设置为懒加载，如果课程成绩对象被读取到了，那么才会对课程成绩对象进行加载；如果aggressiveLazyLoading设置为true，学生证对象同时也会被加载，尽管还没有被用到。(注意只是加载同级别的对象，对于下一个级别的懒加载对象会等到用到了的时候才加载(比如课程成绩对象中同时还有一个懒加载属性为课程对象))

实际使用时推荐lazyLoadingEnabled为true，aggressiveLazyLoading为false。

**使用延迟加载可能导致N+1问题：**
```java
“N+1问题”: 这个问题，无论是association元素还是 collection元素都会遇到，本文以更为典型的collection元素为例。在本系列所使用的示例场景下，当需要查询教师及其所指导的学生（一个 教师可指导多个学生）信息时，我们会这么做：先用一条SQL语句（“N+1问题”中的1）查询教师的信息，即

select * from teacher

此时可查询出多条（记为N）教师记录。为了进一步查询出教师指导的学生的信息，需要针对每一条教师记录，生成一条SQL语句，即

select * from student where supervisor_id=?

以 上SQL语句中的“?”就代表了每个教师的id。显而易见，这样的语句被生成了N条（“N+1问题”中的N）。这样在整个过程中，就总共执行了N+1条 SQL语句，即N+1次数据库查询。而数据库查询通常是应用程序性能的瓶颈，一般应尽量减少数据库查询的次数，那么这种方式就会大大降低系统的性能。

```
MyBatis延迟加载的意义就在于避免一次查询出的数据过多，减少了数据库IO，加快了反应速度；但与此同时会导致n+1问题，关于这个问题的讨论，具体参见：[ibatis:延迟加载和N+1问题](https://blog.csdn.net/zztp01/article/details/6981544)

#### MyBatis初始化
Configuration类总揽所有配置(包括Configuration.xml和所有mapper.xml中的节点配置)：  
```
public class Configuration {
  
  // 保存Configuration.xml中配置的Properties  
  protected Properties variables = new Properties();

  // key值为mapper.xml中namespace，value为与其对应的缓存对象(开启二级缓存之后)
  protected final Map<String, Cache> caches = new StrictMap<Cache>("Caches collection");
  // 多个mapper.xml可能引用同一个cache，通过cacheRefMap存储这种引用关系
  protected final Map<String, String> cacheRefMap = new HashMap<String, String>();
  // 存储各mapper.xml中所有ResultMap
  protected final Map<String, ResultMap> resultMaps = new StrictMap<ResultMap>("Result Maps collection");
  // 存储mapper.xml中公用的sql片段
  /**
   * 建立sql片段
   * <sql id="query_user_where">
   * </sql>
   * //引用其它mapper.xml的sql片段:<include refid="namespace.sql片段"/>
   * key 值为<sql>中的id标签   
   */
  protected final Map<String, XNode> sqlFragments = new StrictMap<XNode>("XML fragments parsed from previous mappers");
  // 存储mapper.xml中<insert>、<update>、<select>、<delete>等节点封装成的MappedStatement对象,key值为nameSpace + sql语句中的id
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
}
```


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
接下来看MyBatis对映射配置文件的解析过程。通过上面对XMLConfigBuilder.mapperElement()方法的介绍我们知道，XMLMapperBuilder负责解析映射配置文件，它也继承了BaseBuilder抽象类，也是具体建造者的角色。XMLMapperBuilder.parse()方法是解析mapper映射文件的入口，具体代码如下：    
```java
// 每个mapper.xml文件就对应一个XMLMapperBuilder对象，每个XMLMapperBuilder对应一个MapperBuilderAssistant对象
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
      ...
    }
    ...
  }

  /**
   * mapper.xml中可能出现的节点共有以下几种情况：<cache>、<cache-ref>、<parameterMap>、<resultMap>、<sql>、
   * <select>、<update>、<delete>、<insert>
   * 
   * XMLMapperBuilder也是将每个节点的解析过程封装成了一个方法，而这些方法由XMLMapperBuilder.configurationElement()
   * 方法调用
   */
  private void configurationElement(XNode context) {
    try {
      // 获取<mapper>节点的namespace属性
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      // 通过MapperBuilderAssistant的currentNamespace字段，记录当前命名空间
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
接下来来看对mapper.xml中各个节点的解析过程：  

#### mapper.xml中<cache>节点解析
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
       * 
       * ，比如：  
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

  public CacheBuilder implementation(Class<? extends Cache> implementation) {
    this.implementation = implementation;
    return this;
  }

  public CacheBuilder addDecorator(Class<? extends Cache> decorator) {
    if (decorator != null) {
      this.decorators.add(decorator);
    }
    return this;
  }

  public CacheBuilder size(Integer size) {
    this.size = size;
    return this;
  }

  public CacheBuilder clearInterval(Long clearInterval) {
    this.clearInterval = clearInterval;
    return this;
  }

  public CacheBuilder readWrite(boolean readWrite) {
    this.readWrite = readWrite;
    return this;
  }

  public CacheBuilder blocking(boolean blocking) {
    this.blocking = blocking;
    return this;
  }
  
  public CacheBuilder properties(Properties properties) {
    this.properties = properties;
    return this;
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
      cache = new SynchronizedCache(cache);
      if (blocking) {
        cache = new BlockingCache(cache);
      }
      return cache;
    } catch (Exception e) {
      throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
    }
  }

  private void setCacheProperties(Cache cache) {
    if (properties != null) {
      MetaObject metaCache = SystemMetaObject.forObject(cache);
      for (Map.Entry<Object, Object> entry : properties.entrySet()) {
        String name = (String) entry.getKey();
        String value = (String) entry.getValue();
        if (metaCache.hasSetter(name)) {
          Class<?> type = metaCache.getSetterType(name);
          if (String.class == type) {
            metaCache.setValue(name, value);
          } else if (int.class == type
              || Integer.class == type) {
            metaCache.setValue(name, Integer.valueOf(value));
          } else if (long.class == type
              || Long.class == type) {
            metaCache.setValue(name, Long.valueOf(value));
          } else if (short.class == type
              || Short.class == type) {
            metaCache.setValue(name, Short.valueOf(value));
          } else if (byte.class == type
              || Byte.class == type) {
            metaCache.setValue(name, Byte.valueOf(value));
          } else if (float.class == type
              || Float.class == type) {
            metaCache.setValue(name, Float.valueOf(value));
          } else if (boolean.class == type
              || Boolean.class == type) {
            metaCache.setValue(name, Boolean.valueOf(value));
          } else if (double.class == type
              || Double.class == type) {
            metaCache.setValue(name, Double.valueOf(value));
          } else {
            throw new CacheException("Unsupported property type for cache: '" + name + "' of type " + type);
          }
        }
      }
    }
    // 如果Cache类继承了InitializingObject接口，则调用其initialize方法继续自定义的初始化操作
    if (InitializingObject.class.isAssignableFrom(cache.getClass())){
      try {
        ((InitializingObject) cache).initialize();
      } catch (Exception e) {
        throw new CacheException("Failed cache initialization for '" +
            cache.getId() + "' on '" + cache.getClass().getName() + "'", e);
      }
    }
  }

  private Cache newBaseCacheInstance(Class<? extends Cache> cacheClass, String id) {
    Constructor<? extends Cache> cacheConstructor = getBaseCacheConstructor(cacheClass);
    try {
      return cacheConstructor.newInstance(id);
    } catch (Exception e) {
      throw new CacheException("Could not instantiate cache implementation (" + cacheClass + "). Cause: " + e, e);
    }
  }

  private Constructor<? extends Cache> getBaseCacheConstructor(Class<? extends Cache> cacheClass) {
    try {
      return cacheClass.getConstructor(String.class);
    } catch (Exception e) {
      throw new CacheException("Invalid base cache implementation (" + cacheClass + ").  " +
          "Base cache implementations must have a constructor that takes a String id as a parameter.  Cause: " + e, e);
    }
  }

  private Cache newCacheDecoratorInstance(Class<? extends Cache> cacheClass, Cache base) {
    Constructor<? extends Cache> cacheConstructor = getCacheDecoratorConstructor(cacheClass);
    try {
      return cacheConstructor.newInstance(base);
    } catch (Exception e) {
      throw new CacheException("Could not instantiate cache decorator (" + cacheClass + "). Cause: " + e, e);
    }
  }

  private Constructor<? extends Cache> getCacheDecoratorConstructor(Class<? extends Cache> cacheClass) {
    try {
      return cacheClass.getConstructor(Cache.class);
    } catch (Exception e) {
      throw new CacheException("Invalid cache decorator (" + cacheClass + ").  " +
          "Cache decorators must have a constructor that takes a Cache instance as a parameter.  Cause: " + e, e);
    }
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
#### mapper.xml中<select>、<update>、<delete>、<insert>节点解析
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
#### MyBatis StatementHandler
JDBC中Statement使用：  
```java
public class TestJDBC {

    public static void main(String[] args)   {
        Connection con=null;
        PreparedStatement ps =null;
        ResultSet rs=null;
        try {
            //加载数据库驱动
            Class.forName("com.mysql.jdbc.Driver");
            //通过驱动管理类获取数据库链接
            con = DriverManager.getConnection("jdbc:mysql://localhost:3306/student", "root", "root");
            String sql="select stu.name,stu.age,stu.id from student stu where id = ? ";
            //执行sql语句
            ps = con.prepareStatement(sql);
            ps.setInt(1, 8);
            //遍历查询结果集
            rs = ps.executeQuery();
            while(rs.next()){
                System.out.println("姓名： "+rs.getString("name")+" 年龄："+rs.getInt("age")+" id: "+rs.getInt("id"));
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (SQLException e){
            e.printStackTrace();
        }finally{
            //关闭资源
            try{
                if(rs != null){
                    rs.close();
                }
                if(ps != null){
                    ps.close();
                }
                if(con != null){
                    con.close();
                }
            }catch(SQLException e){
                e.printStackTrace();
            }
        }
    }
}
```
StatementHandler功能很多，例如创建Statement对象，为SQL语句绑定实参，执行select、insert等多种类型的SQL语句，批量执行SQL语句，将结果集映射成结果对象。  
```java
public interface StatementHandler {

  // 从连接中获取一个Statement
  Statement prepare(Connection connection, Integer transactionTimeout)
      throws SQLException;

  // 绑定Statement执行时所需的实参
  void parameterize(Statement statement)
      throws SQLException;

  // 批量执行SQL语句
  void batch(Statement statement)
      throws SQLException;

  // 执行update/insert/delete语句
  int update(Statement statement)
      throws SQLException;

  // 执行select语句
  <E> List<E> query(Statement statement, ResultHandler resultHandler)
      throws SQLException;

  <E> Cursor<E> queryCursor(Statement statement)
      throws SQLException;

  BoundSql getBoundSql();

  ParameterHandler getParameterHandler(); // 获取其中封装的ParameterHandler

}
```

### MyBatis SqlSession执行流程
**SqlSession的获取：**  
```java
public class MyBatisUtil {

    public static SqlSessionFactory getSqlSessionFactory() {
        return sqlSessionFactory;
    }

    private static SqlSessionFactory sqlSessionFactory = null;

    static {
        String resource = "configuration.xml";
        try {
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(
                    Resources.getResourceAsStream(resource)
            );
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

public class MyBatisExample {

    public static void main(String[] args) {
        SqlSession sqlSession = null;
        try {
            // sqlSession默认关闭事务的自动提交
            sqlSession = MyBatisUtil.getSqlSessionFactory().openSession();
            ...           
    }
}

public class DefaultSqlSessionFactory implements SqlSessionFactory {

  private final Configuration configuration;

  public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
  }
  
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);

      // 创建指定类型的被事务包裹的Executor
      final Executor executor = configuration.newExecutor(tx, execType);

      // 返回创建好的SqlSession，每个sqlSession对象对应一个Executor对象
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
}

public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }
}
```
**SqlSession.getMapper实现：**  
```java
public class MyBatisExample {

    public static void main(String[] args) {
        SqlSession sqlSession = null;
        try {
            // sqlSession默认关闭事务的自动提交
            sqlSession = MyBatisUtil.getSqlSessionFactory().openSession();

            // 猜测应该是根据mapper文件创建了一个实现了UserDao接口的代理对象
            UserDao userDao = sqlSession.getMapper(UserDao.class);
            User user = userDao.getById(1);
            System.out.println(user);
        } finally {
            if (sqlSession != null) {
                sqlSession.close();
            }
        }
    }
}

public class DefaultSqlSession implements SqlSession {

  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
}

public class Configuration {
  
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
}

public class MapperRegistry {
   
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // key值为要被代理的接口类，value为其对应的MapperProxyFactory
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
}

// MapperProxyFactory类对象在Configuration.addMapper方法中被创建，初始化时methodCache中并没有存储任何东西
// 一个mapperInterface对应一个MapperProxyFactory对象
public class MapperProxyFactory<T> {

  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }
  
  // 每次调用session.getMapper都会返回一个新的mapperInterface接口的动态代理对象(MapperProxy类对象)，但是多个对象中持有的sqlSession对象是相同的
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

  protected T newInstance(MapperProxy<T> mapperProxy) {
    // JDK动态代理创建mapperInterface的代理实现，invocationHandler实现是mapperProxy
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
}
```
userDao.getById方法执行流程：userDao是一个动态代理对象，对其方法的调用实际是从与其对应的MapperProxy中的invoke方法开始。  
```java
// 调用userDao.getById方法方法实际上是从MapperProxy中的invoke方法开始调用
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;

  // 一个mapperInterface下所有SqlSession对象共用一个methodCache，就是产生MapperProxy的MapperProxyFactory中的methodCache
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        // 被代理的方法来源于Object类
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    // 从缓存中获取MapperMethod对象，如果缓存中没有，则创建新的MapperMethod对象并添加到缓存中
    final MapperMethod mapperMethod = cachedMapperMethod(method);

    // 调用mapperMethod.execute方法执行SQL语句
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
}

```
可见实际方法的反射调用是先找到与当前method对应的MapperMethod，然后调用mapperMethod.execute方法执行SQL语句：  
```java
public class MapperMethod {

  private final SqlCommand command;
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
  
  // SqlCommand中并没有存储实际的sql语句信息，实际sql信息还存储在Configuration.mappedStatements代表的map中
  public static class SqlCommand {

    private final String name;
    // 枚举类型，type可以为UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH;
    private final SqlCommandType type;

    // 将MappedStatement中相关信息封装为SqlCommand对象
    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      final String methodName = method.getName();
      final Class<?> declaringClass = method.getDeclaringClass();
      MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
          configuration);
      if (ms == null) {
        if (method.getAnnotation(Flush.class) != null) {
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          throw new BindingException("Invalid bound statement (not found): "
              + mapperInterface.getName() + "." + methodName);
        }
      } else {
        name = ms.getId();
        type = ms.getSqlCommandType(); 
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
    }

    public String getName() {
      return name;
    }

    public SqlCommandType getType() {
      return type;
    }

    // 从Configuration对象中获取mapper.xml中与当前方法对应的MappedStatement
    private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName,
        Class<?> declaringClass, Configuration configuration) {
      String statementId = mapperInterface.getName() + "." + methodName;
      if (configuration.hasStatement(statementId)) {
        return configuration.getMappedStatement(statementId);
      } else if (mapperInterface.equals(declaringClass)) {
        return null;
      }
      for (Class<?> superInterface : mapperInterface.getInterfaces()) {
        if (declaringClass.isAssignableFrom(superInterface)) {
          MappedStatement ms = resolveMappedStatement(superInterface, methodName,
              declaringClass, configuration);
          if (ms != null) {
            return ms;
          }
        }
      }
      return null;
    }
  }

  public static class MethodSignature {

    private final boolean returnsMany;
    private final boolean returnsMap;
    private final boolean returnsVoid;
    private final boolean returnsCursor;
    private final Class<?> returnType;
    private final String mapKey;
    private final Integer resultHandlerIndex;
    private final Integer rowBoundsIndex;
    // 使用paramNameResolver处理Mapper接口中定义的方法的参数列表，内部有一个map维护方法参数名称与参数索引之间的关系
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
      if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
      } else if (resolvedReturnType instanceof ParameterizedType) {
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
      } else {
        this.returnType = method.getReturnType();
      }
      this.returnsVoid = void.class.equals(this.returnType);
      this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
      this.returnsCursor = Cursor.class.equals(this.returnType);
      this.mapKey = getMapKey(method);
      this.returnsMap = this.mapKey != null;
      this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
      this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
      this.paramNameResolver = new ParamNameResolver(configuration, method);
    }

    public Object convertArgsToSqlCommandParam(Object[] args) {
      return paramNameResolver.getNamedParams(args);
    }

    ...  
  }

  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) { // 根据其sql command实际类型进行相应操作
      case INSERT: {
        // 把args参数数组与method中参数绑定起来 
        // method.convertArgsToSqlCommandParam返回值可能是一个map，存储了参数名称与传入的参数值之间的对应关系
        Object param = method.convertArgsToSqlCommandParam(args);
        // command.getName()返回结果是mapper.xml中namespace + sql语句中的id属性，比如为com.michael.mybatisStudy.dao.UserDao.getById
        // 根据command.getName()返回结果可以找到其对应的MappedStatement对象，其中含有实际的sql语句信息
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        // rowCountResult方法的作用是将sqlsession中的insert等方法返回的int值转换为Mapper接口中对应方法的返回值
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          // 使用sqlSession进行实际的数据库操作
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
}
```
接下来介绍sqlSession.selectOne方法，看其具体是如何执行sql语句的：  
```
// SqlSession的创建过程
public class DefaultSqlSessionFactory implements SqlSessionFactory {

  private final Configuration configuration;

  public DefaultSqlSessionFactory(Configuration configuration) {
    this.configuration = configuration;
  }
  
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }

  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);

      // 创建指定类型的被事务包裹的Executor
      final Executor executor = configuration.newExecutor(tx, execType);

      // 返回创建好的SqlSession，每个sqlSession对象对应一个Executor对象，
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
}

public class Configuration {
   
  // 默认创建的是SimpleExecutor对象
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) { // 如果二级缓存功能开启的话
      executor = new CachingExecutor(executor);// CachingExecutor是普通Executor的装饰器，在其上添加二级缓存功能
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
}

public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  // 默认是SimpleExecutor，每个DefaultSqlSession对应一个Executor，每个Executor对应一个Transaction接口实现(比如ManagedTransaction）  
  private final Executor executor;

  private final boolean autoCommit;
  private boolean dirty;
  private List<Cursor<?>> cursorList;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.configuration = configuration;
    this.executor = executor;
    this.dirty = false;
    this.autoCommit = autoCommit;
  }

  @Override
  public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }

  @Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
  }

  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      // 获取对应的MappedStatement对象
      MappedStatement ms = configuration.getMappedStatement(statement);

      // Executor.NO_RESULT_HANDLER为null
      // 如果二级缓存开启的话，这里调用的是CachingExecutor.query方法
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
}

public class CachingExecutor implements Executor {

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    // 创建在缓存中的key值
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        // 先从二级缓存中查找，看此sql语句之前执行的结果是否尚在缓存中
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    // 调用BaseExecutor的query(ms, parameterObject, rowBounds, resultHandler, key, boundSql)方法
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
}

public abstract class BaseExecutor implements Executor {

  private static final Log log = LogFactory.getLog(BaseExecutor.class);

  protected Transaction transaction; // BaseExecutor使用此对象进行事务控制
  protected Executor wrapper;

  protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
  // 内部通过HashMap实现的一级缓存
  protected PerpetualCache localCache; // 每个sqlSession对应一个Executor，每个Executor对应一个一级缓存localCache
  protected PerpetualCache localOutputParameterCache;
  protected Configuration configuration;

  protected int queryStack;
  private boolean closed;

  protected BaseExecutor(Configuration configuration, Transaction transaction) {
    this.transaction = transaction;
    this.deferredLoads = new ConcurrentLinkedQueue<DeferredLoad>();
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    this.closed = false;
    this.configuration = configuration;
    this.wrapper = this;
  } 

  @SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 二级缓存查找完后在一级缓存中查找
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 缓存中查找不到的话就在数据库中查找
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }

  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      // 调用真正的Executor实现类实现查找功能，比如SimpleExecutor
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
}

// SimpleExecutor整个就相当于模板模式(封装Statement的相关操作) + 策略模式的组合
public class SimpleExecutor extends BaseExecutor {

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
     
      // 实际返回的是RoutingStatementHandler对象，根据ms.StatementType选择具体的StatementHandler实现。默认是PreparedStatementHandler
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      // statement处理好之后进行真正的查询操作
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

  // 方法的作用是完成java.sql.Statement对象的创建和初始化
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    // 创建preparedStatement对象
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 处理Statement对象中的占位符
    handler.parameterize(stmt);
    return stmt;
  }

  protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) { // 添加log代理
      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
      return connection;
    }
  }
}

public abstract class BaseStatementHandler implements StatementHandler {

  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      // 实际调用的是PreparedStatementHandler.instantiateStatement(...)方法，返回的是一个PreparedStatement对象
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }
}

public class PreparedStatementHandler extends BaseStatementHandler {

  @Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() != null) {
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      return connection.prepareStatement(sql);
    }
  }

  @Override
  public void parameterize(Statement statement) throws SQLException {
    // 使用parameterHandler设置statement中sql参数
    parameterHandler.setParameters((PreparedStatement) statement);
  }
}
```
接下来来看真正的查询操作是怎么进行的：  
```java
public class SimpleExecutor extends BaseExecutor {

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
     
      // 实际返回的是RoutingStatementHandler对象，根据ms.StatementType选择具体的StatementHandler实现。默认是PreparedStatementHandler
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      // statement处理好之后进行真正的查询操作
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
}

public class RoutingStatementHandler implements StatementHandler {

  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    return delegate.<E>query(statement, resultHandler);
  }
}

public class PreparedStatementHandler extends BaseStatementHandler {

  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 进行实际查询操作
    ps.execute();
    // 进行resultSet封装
    return resultSetHandler.<E> handleResultSets(ps);
  }
}
```

JDBC中Statement使用：  
```java
public class TestJDBC {

    public static void main(String[] args)   {
        Connection con=null;
        PreparedStatement ps =null;
        ResultSet rs=null;
        try {
            //加载数据库驱动
            Class.forName("com.mysql.jdbc.Driver");
            //通过驱动管理类获取数据库链接
            con = DriverManager.getConnection("jdbc:mysql://localhost:3306/student", "root", "root");
            String sql="select stu.name,stu.age,stu.id from student stu where id = ? ";
            //执行sql语句
            ps = con.prepareStatement(sql);
            ps.setInt(1, 8);
            //遍历查询结果集
            rs = ps.executeQuery();
            while(rs.next()){
                System.out.println("姓名： "+rs.getString("name")+" 年龄："+rs.getInt("age")+" id: "+rs.getInt("id"));
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (SQLException e){
            e.printStackTrace();
        }finally{
            //关闭资源
            try{
                if(rs != null){
                    rs.close();
                }
                if(ps != null){
                    ps.close();
                }
                if(con != null){
                    con.close();
                }
            }catch(SQLException e){
                e.printStackTrace();
            }
        }
    }
}
```