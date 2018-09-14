---
layout:     post
title:      MyBatis学习系列之MyBatis的SqlSession执行流程分析
date:       2018-9-13
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - MyBatis
---
>本文记录我对MyBati SqlSession执行流程分析相关过程及注意事项的理解(基于MyBatis 3.4版本)。     

_ _ _
### **前言**
MyBatis中SqlSession通常使用方式如下：  
```java
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

public class MyBatisExample {

    public static void main(String[] args) {
        SqlSession sqlSession = null;
        try {
            // sqlSession默认关闭事务的自动提交
            sqlSession = MyBatisUtil.getSqlSessionFactory().openSession(false);
            // 实际是根据mapper文件创建了一个实现了UserDao接口的代理对象，JDK动态代理
            UserDao userDao = sqlSession.getMapper(UserDao.class);
            User user = userDao.getById(1);
            System.out.println(user);
          
            sqlSession.commit();
        } finally {
            if (sqlSession != null) {
                sqlSession.close();
            }
        }
    }
}
```
接下来就以这个Demo为例，分析其内部实现机制。  

_ _ _
### SqlSession的获取
```java
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
  // 每个sqlSession对象对应一个Executor对象,每个Executor对应一个Transaction类对象
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

_ _ _
### SqlSession.getMapper实现
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
获取的userDao是一个动态代理对象，对其方法的调用实际是从与其对应的MapperProxy中的invoke方法开始：  
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
接下来介绍实际对数据库进行操作的sqlSession.selectOne方法，看其具体是如何执行sql语句的：  
```java
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
由上面的分析过程可以知道，查找过程为：二级缓存中查找 ——> 一级缓存中查找 ——> 数据库中查找。  

具体sqlSession执行过程图示如下：  
<img src="/img/2018-9-13/sqlSessionExecutorProcess.png" width="700" height="700" alt="sqlSession执行流程" />
<center>图1：sqlSession执行流程图</center>  
  

分析完上面的整个查找流程之后，我们再对sqlSession执行过程中的事务控制和缓存两个点做一下详细分析。    

_ _ _
### SqlSession执行过程中的事务控制   
由上面的分析，我们知道每个SqlSession对象内部都持有一个与自己对应的Executor对象用来执行之后的查找操作，Executor进行实际的数据库操作的时候会通过其内部持有的transaction实例对象来获取一个数据库连接：  
```java
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
```
以JdbcTransaction为例看其getConnection()实现：    
```java
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

      // 根据environment中的配置可获取自定义的TransactionFactory，从而可以创建自定义的Transaction实现
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

public class JdbcTransaction implements Transaction {

  protected Connection connection;
  protected DataSource dataSource;
  protected TransactionIsolationLevel level;
  // MEMO: We are aware of the typo. See #941
  protected boolean autoCommmit;

  public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
    dataSource = ds;
    level = desiredLevel;
    autoCommmit = desiredAutoCommit;
  }

  @Override
  public Connection getConnection() throws SQLException {
    // 之前没创建过Connection才创建新的Connection
    if (connection == null) {
      openConnection();
    }
    return connection;
  }

  protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
      log.debug("Opening JDBC Connection");
    }
    connection = dataSource.getConnection();
    if (level != null) {
      connection.setTransactionIsolation(level.getLevel());
    }
    // autoCommmit默认为false
    setDesiredAutoCommit(autoCommmit);
  }

  protected void setDesiredAutoCommit(boolean desiredAutoCommit) {
    try {
      if (connection.getAutoCommit() != desiredAutoCommit) {
        if (log.isDebugEnabled()) {
          log.debug("Setting autocommit to " + desiredAutoCommit + " on JDBC Connection [" + connection + "]");
        }
        connection.setAutoCommit(desiredAutoCommit);
      }
    } catch (SQLException e) {
      // Only a very poorly implemented driver would fail here,
      // and there's not much we can do about that.
      throw new TransactionException("Error configuring AutoCommit.  "
          + "Your driver may not support getAutoCommit() or setAutoCommit(). "
          + "Requested setting: " + desiredAutoCommit + ".  Cause: " + e, e);
    }
  }

}
```
可以看到每个SqlSession对象对应的Connection对象都是唯一的，之后使用SqlSession进行的所有对数据库的操作都是使用这个Connection对象来进行的，并且默认开启了事务控制，所以我们使用完sqlSession之后要记得进行commit()操作，否则我们进行的插入数据等操作是不生效的。    

MyBatis自身对于Transaction接口的实现有两种：JdbcTransaction以及ManagedTransaction，分别JdbcTransactionFactory和ManagedTransactionFactory创建。    

* 使用JDBC的事务管理机制,就是利用java.sql.Connection对象完成对事务的提交
* 使用MANAGED的事务管理机制，这种机制mybatis自身不会去实现事务管理，而是让程序的容器（JBOSS,WebLogic）来实现对事务的管理

在实践中，MyBatis通常会与Spring集成使用，数据库的事务交由Spring管理，具体实现在之后的博客中会介绍。  

_ _ _
### SqlSession执行过程中的缓存使用
上面已经说过，查找过程是是先在二级缓存，之后是一级缓存，最后是实际数据库，mapper.xml中对应的namespace对应一个二级缓存对象，多个mapper.xml可以对应同一个二级缓存对象，具体如下图：   
<img src="/img/2018-9-13/CacheUseProcess.png" width="700" height="700" alt="缓存使用过程" />
<center>图2：缓存使用过程</center>  

#### 二级缓存对象的创建
二级缓存对象的创建是在mapper.xml解析cache节点的过程中创建MapperBuilderAssistant对象的过程时创建的：  
```java
public class XMLMapperBuilder extends BaseBuilder {

  private final MapperBuilderAssistant builderAssistant;

  // mybaits的二级缓存是mapper范围级别，除了在SqlMapConfig.xml设置二级缓存的总开关，还要在具体的mapper.xml中开启二级缓存。
  // 负责解析<cache>节点
  private void cacheElement(XNode context) throws Exception {
    if (context != null) {
      ...  
      // 通过MapperBuilderAssistant创建cache对象，并添加到Configuration.caches集合中保存
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }  
}
```
每个mapper.xml对应一个XMLMapperBuilder对象，每个XMLMapperBuilder对象对应一个builderAssistant对象，每个builderAssistant对象对应一个Cache类对象：  
```java
public class MapperBuilderAssistant extends BaseBuilder {

  private Cache currentCache;

  public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
    // 通过CacheBuilder类创建Cache,默认实现类是PerpetualCache
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }
}
```
之后在此mapper.xml的创建每个MappedStatement的过程中都将此currentCache引用传入，这样就可以通过MappedStatement.getCache获取此Cache二级缓存对象：  
```java
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
    ...
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
        .cache(currentCache); // 引用了每个namepspace对应的唯一的Cache二级缓存对象

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
```
#### 二级缓存的查找
对于缓存的查找过程是通过CachingExecutor实现的，CachingExecutor是一个装饰器，装饰了实际的Executor，为其添加了缓存功能：
```java
public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  public CachingExecutor(Executor delegate) {
    this.delegate = delegate;
    delegate.setExecutorWrapper(this);
  }

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
```
可见对于缓存的查找是通过TransactionalCacheManager.getObject方法实现的，向缓存中存数据是通过putObject方法先存入到TransactionalCache对象中的：  
```java
public class TransactionalCacheManager {

  // transactionalCaches中的key值默认是PerpetualCache类对象，可能被	其它类装饰
  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();

  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }

  public void putObject(Cache cache, CacheKey key, Object value) {
    getTransactionalCache(cache).putObject(key, value);
  }

  private TransactionalCache getTransactionalCache(Cache cache) {
    TransactionalCache txCache = transactionalCaches.get(cache);
    if (txCache == null) {
      txCache = new TransactionalCache(cache);
      transactionalCaches.put(cache, txCache);
    }
    return txCache;
  }

  public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    }
  }

  public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.rollback();
    }
  }
}

public class TransactionalCache implements Cache {
  // 保存事务提交之前缓存的数据
  private final Map<Object, Object> entriesToAddOnCommit;
}
```
为什么这里还要使用TransactionalCacheManager组件和TransactionalCache组件呢，而不是直接将数据存入二级缓存Cache对象中呢？     

是由于事务控制的原因。通常情况下每个线程使用一个SqlSession对象完成对数据库的操作，多个SqlSession可能同时操作同一个二级缓存对象，如果直接将某个SqlSession更改后未提交的查询结果存入缓存中的话，就有可能有另一个线程读取到缓存中的脏数据。比如事务T1中添加了记录A，之后查询A记录，事务尚未提交；之后事务T2查询记录A，如果事务T1在尚未提交之前就将A记录的查询结果放入二级缓存中的话，事务T2就会查询到二级缓存中的事务T1尚未提交的脏数据，这显然不是我们期望的结果。  

所以在一个SqlSession的某个事务尚未提交之前的查询结果先缓存在transactionalCaches中的TransactionalCache对象的entriesToAddOnCommit的Map中，在事务提交的时候再将此结果放置在二级缓存当中。    

#### 二级缓存的使用时注意事项
1. 只能在【只有单表操作】的表上使用缓存：不只是要保证这个表在整个系统中只有单表操作，而且和该表有关的全部操作必须全部在一个namespace下。
2. 在可以保证查询远远大于insert,update,delete操作的情况下使用缓存，这一点需要保证在1的前提下才可以！

#### 避免使用二级缓存
二级缓存带来的好处可能比不上他所隐藏的危害。
```
缓存是以namespace为单位的，不同namespace下的操作互不影响。

insert,update,delete操作会清空所在namespace下的全部缓存。

通常使用MyBatis Generator生成的代码中，都是各个表独立的，每个表都有自己的namespace。
```
**为什么避免使用二级缓存?**  
在符合【Cache使用时的注意事项】的要求时，并没有什么危害。其他情况就会有很多危害了。比如：  

* 针对一个表的某些操作不在他独立的namespace下进行。  

例如在UserMapper.xml中有大多数针对user表的操作。但是在一个XXXMapper.xml中，还有针对user单表的操作。
这会导致user在两个命名空间下的数据不一致。如果在UserMapper.xml中做了刷新缓存的操作，在XXXMapper.xml中缓存仍然有效，如果有针对user的单表查询，使用缓存的结果可能会不正确。
更危险的情况是在XXXMapper.xml做了insert,update,delete操作时，会导致UserMapper.xml中的各种操作充满未知和风险。

* 多表操作一定不能使用缓存

首先不管多表操作写到那个namespace下，都会存在某个表不在这个namespace下的情况。  

例如两个表：role和user_role，如果我想查询出某个用户的全部角色role，就一定会涉及到多表的操作。  
```java
<select id="selectUserRoles" resultType="UserRoleVO">
    select * from user_role a,role b where a.roleid = b.roleid and a.userid = #{userid}
</select>
```
像上面这个查询，应该写到那个xml中呢？？  

不管是写到RoleMapper.xml还是UserRoleMapper.xml，或者是一个独立的XxxMapper.xml中。如果使用了二级缓存，都会导致上面这个查询结果可能不正确。  

如果正好修改了这个用户的角色，上面这个查询使用缓存的时候结果就是错的。  
  
如果让他们都使用同一个namespace（通过<cache-ref>）来避免脏数据，那就失去了缓存的意义。  

对于上述的问题的解决方案可以是通过拦截器判断执行的Sql语句涉及到哪些表，然后将相关表的缓存自动清空，但这样做会降低使用缓存的效率。  

_ _ _
### SqlSessionManager介绍
SqlSessionManager同时实现了SqlSession和SqlSessionFactory接口，也就同时提供了SqlSessionFactory创建SqlSession对象以及SqlSession操作数据库的功能。

SqlSessionManager作为SqlSessionFactory使用时与普通sqlSessionFactory没有什么区别：  
```java
public class MyBatisExample {

    private static String resource = "configuration.xml";

    public static void main(String[] args) throws IOException {
        SqlSession sqlSession = null;
        try {
            // 使用sqlSessionManager获取SqlSession
            SqlSessionManager sqlSessionManager = SqlSessionManager.newInstance(Resources.getResourceAsStream(resource));
            sqlSession = sqlSessionManager.openSession(false);
            // 猜测应该是根据mapper文件创建了一个实现了UserDao接口的代理对象
            UserDao userDao = sqlSession.getMapper(UserDao.class);
            User user = userDao.getById(1);
            System.out.println(user);

            User user1 = new User("heihei", 14);
            int result = userDao.insertUser(user1);
            System.out.println("result: " + result + " " + user1.getId());
            sqlSession.commit();
        } finally {
            if (sqlSession != null) {
                sqlSession.close();
            }
        }
    }
}

public class SqlSessionManager implements SqlSessionFactory, SqlSession {

  private final SqlSessionFactory sqlSessionFactory;
  private final SqlSession sqlSessionProxy;

  private final ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<SqlSession>();

  private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
    this.sqlSessionFactory = sqlSessionFactory;
    this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[]{SqlSession.class},
        new SqlSessionInterceptor());
  }

  // 使用SqlSessionManager.openSession方法获取SqlSession，与普通sqlSessionFactory没有什么区别
  @Override
  public SqlSession openSession() {
    return sqlSessionFactory.openSession();
  }
}
```
SqlSessionManager作为SqlSession使用时有两种方式，使用哪种模式取决于在调用之前是否调用了SqlSessionManager.startManagedSession()方法，下面来看看这个方法的作用：  
```java
public class SqlSessionManager implements SqlSessionFactory, SqlSession {

  private final ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<SqlSession>();
  
  public void startManagedSession() {
    this.localSqlSession.set(openSession());
  }
}
```
可见是新生成了一个数据库连接，之后将其放在了一个ThreadLocal变量中，这样做的目的是为了记录与当前线程绑定的SqlSession对象，供当前线程循环使用，从而避免在同一线程多次创建SqlSession对象带来的性能损失。如果在使用之前不调用SqlSessionManager.startManagedSession()方法，就会出现同一线程每次通过SqlSessionManager对象访问数据库时，都会创建新的DefaultSqlSession对象完成数据库操作：  
```java
public class SqlSessionManager implements SqlSessionFactory, SqlSession {

  private final SqlSessionFactory sqlSessionFactory;
  private final SqlSession sqlSessionProxy;

  private final ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<SqlSession>();

  private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
    this.sqlSessionFactory = sqlSessionFactory;
    this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[]{SqlSession.class},
        new SqlSessionInterceptor());
  }

  @Override
  public <T> T selectOne(String statement) {
    // 使用动态代理对象进行实际操作
    return sqlSessionProxy.<T> selectOne(statement);
  }
}
```
sqlSessionProxy是一个动态代理对象，对它的方法的调用会转到SqlSessionInterceptor.invoke方法中去：  
```java
public class SqlSessionManager implements SqlSessionFactory, SqlSession {

  private class SqlSessionInterceptor implements InvocationHandler {
    public SqlSessionInterceptor() {
        // Prevent Synthetic Access
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 尝试从localSqlSession中获取之前通过startManagedSession方法设置好的SqlSession对象
      final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
      if (sqlSession != null) {
        try {
          // 反射调用sqlSession的方法
          return method.invoke(sqlSession, args);
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
      } else {
        // 如果之前没有通过startManagedSession方法设置sqlSession对象，则每次
        // 通过SqlSessionManager对象访问数据库时，都会创建新的DefaultSqlSession对象
        final SqlSession autoSqlSession = openSession();
        try {
          final Object result = method.invoke(autoSqlSession, args);
          autoSqlSession.commit();
          return result;
        } catch (Throwable t) {
          autoSqlSession.rollback();
          throw ExceptionUtil.unwrapThrowable(t);
        } finally {
          autoSqlSession.close();
        }
      }
    }
  }
}
```
通过上面代码分析可以看出，SqlSessionManager的作为SqlSession使用的优势在于可以记录与当前线程绑定的SqlSession对象，劣势在于对于实际SqlSession对象方法的调用是通过反射进行的，对效率会有所影响。  

_ _ _
### SqlSessionFactory与SqlSession线程安全
SqlSessionFactory.openSession()方法我觉得是线程安全的，但创建好的SqlSession并不是是线程安全的，理由之一是每个SqlSession底层对应一个数据库的Connection连接，实际对数据库的操作都是使用这个Connection连接进行的，java.sql.Connection对象不是线程安全的，所以SqlSession对象不是线程安全的，最好在单线程中使用它。  

关于MyBatis的SqlSession的执行流程就介绍到这里。  

(完)  

参考文章：[Mybatis - 二级缓存的利弊](https://blog.csdn.net/j080624/article/details/78668649)  
