---
layout:     post
title:      Redis基础知识及Jedis相关操作
date:       2018-3-3
author:     W-M
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 数据库
---

>之前做项目时在Spring中集成Redis实现了赞踩功能，现在做的新的项目又涉及到了赞踩功能，发现之前学的Redis相关知识已经忘的差不多了，因此将相关知识记录一下，以便以后温习。 本篇文章主要是网络中多篇文章的个人总结综合，主要涉及到了Redis的常见的使用方面的知识，有待以后继续深入。   

_ _ _ 
### **Redis中支持的5种不同Value存储的数据结构** 
Redis最为常用的数据类型主要有五种：String、Hash、List、Set、SortedSet，在具体描述这几种数据类型之前，我们先通过一张图了解下Redis 内部内存管理中是如何描述这些不同数据类型的：  
<img src="/img/2018-3-3/RedisObject.jpg" width="500" height="500" alt="redisObject" />
<center>图1：Redis内部对于不同数据类型的管理</center>  
首先Redis内部使用一个redisObject对象来表示所有的key和value,redisObject最主要的信息如上图所示：type代表一个value对象具体是何种数据类型，encoding是不同数据类型在redis内部的存储方式，比如：type=string代表value存储的是一个普通字符串，那么对应的encoding可以是raw或者是int,如果是int则代表实际redis内部是按数值型类存储和表示这个字符串的，当然前提是这个字符串本身可以用数值表示，比如:"123" "456"这样的字符串。下面就来介绍下这五种数据类型及其在Java中的使用方式：  
* String:String 是最常用的一种数据类型，代表key对应的value直接按照String类型进行存储，String 在 redis 内部存储默认就是一个字符串，被 redisObject 所引用，当遇到 incr,decr 等操作时会转成数值型进行计算，此时 redisObject 的 encoding 字段为int。
* Hash:Redis 的 Hash 实际是内部存储的 Value 为一个 HashMap，并提供了直接存取这个 Map 成员的接口，示例图如下：  
<img src="/img/2018-3-3/RedisHash.jpg" width="500" height="500" alt="RedisHash" />
<center>图2：Redis内Hash数据结构存储示意图</center>  
也就是说，Key 仍然是用户 ID，value 是一个 Map，这个 Map 的 key 是成员的属性名，value 是属性值，这样对数据的修改和存取都可以直接通过其内部 Map 的 Key（Redis 里称内部 Map 的 key 为 field），也就是通过 key（用户 ID） + field（属性标签）就可以操作对应属性数据了，需要注意的一点是Redis 提供了接口（hgetall）可以直接取到全部的属性数据，但是如果内部 Map 的成员很多，那么涉及到遍历整个内部 Map 的操作，由于 Redis 单线程模型的缘故，这个遍历操作可能会比较耗时，而另其它客户端的请求完全不响应。
* List:即value中存储的是一个双向链表，可以支持双向查找和遍历。  
* Set：Set 对外提供的功能与 List 类似，也是一个列表的功能，特殊之处在于 Set 是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择;或者当数据增删操作较多时，性能上Set也要优于List。  
* SortedSet：是一个有序的并且不重复的集合列表，通过用户提供的一个优先级(score)的参数来为成员排序。  

五种数据类型在Java中使用示例如下：  
```java
public static void print(int index, Object obj) {
    System.out.println(String.format("%d, %s", index, obj.toString()));
}

public static void main(String[] args) {
    // Redis中默认有16个DataBase
    Jedis jedis = new Jedis("redis://localhost:6379/1");
    // 删除选中的数据库中之前含有的keys，即使此keys已经持久化到磁盘上
    jedis.flushDB();

    /**
     * String类型，普通key-value存储
     * 可用来实现验证码，页面PV，缓存
     */
    // 设置新的key-value
    jedis.set("hello", "world");
    print(1, jedis.get("hello"));
    jedis.rename("hello", "newhello");
    print(1, jedis.get("newhello"));
    // 设置"hello"对应的value为world并在15s之后从redis中自动删除此键值对
    // 这种使用方式应用场景可为 验证码功能(比如30min内激活有效)  的实现
    jedis.setex("hello", 15, "world");
    // redis中进行简单数值操作，适用于点赞，浏览量等应用场景
    jedis.set("pv", "100");
    jedis.incr("pv");
    jedis.incrBy("pv", 5);
    print(2, jedis.get("pv"));
    jedis.decrBy("pv", 2);
    print(2, jedis.get("pv"));
    print(3, jedis.keys("*"));

    /**
     * list存储
     * 可从左右两端分别插入，应用场景为最新列表，关注列表
     */
    String listName = "list";
    jedis.del(listName);
    for (int i = 0; i < 10; i++) {
        // 左端插入对应lpush，右端插入对应rpush
        jedis.lpush(listName, "a" + String.valueOf(i));
    }
    print(4, jedis.lrange(listName, 0, 12));

    /**
     * hash存储
     * 可用来存储对象属性，不定长属性数，在redis中使用hash数据结构,可以随意为对象添加属性，
     * 是对传统关系型数据库灵活性不足的一种补充
     */
    String userKey = "userxx";
    jedis.hset(userKey, "name", "jim");
    jedis.hset(userKey, "age", "12");
    jedis.hset(userKey, "phone", "18618181818");
    print(12, jedis.hget(userKey, "name"));
    print(13, jedis.hgetAll(userKey));
    jedis.hdel(userKey, "phone");
    print(13, jedis.hgetAll(userKey));
    print(14, jedis.hexists(userKey, "phone"));
    print(15, jedis.hkeys(userKey));
    print(16, jedis.hvals(userKey));
    // 不存在则设置
    jedis.hsetnx(userKey, "school", "zju");
    jedis.hsetnx(userKey, "name", "yxy");
    print(17, jedis.hgetAll(userKey));

    /**
     * set存储
     * 集合set，赞踩功能适用于set实现,抽奖、已读、微博共同关注好友
     */
    String likeKey1 = "commentLike1";
    String likeKey2 = "commentLike2";
    for (int i = 0; i < 10; i++) {
        jedis.sadd(likeKey1, String.valueOf(i));
        jedis.sadd(likeKey2, String.valueOf(i * i));
    }
    print(18, jedis.smembers(likeKey1));
    print(19, jedis.smembers(likeKey2));
    // 集合的交并补操作
    print(20, jedis.sunion(likeKey1, likeKey2));
    print(21, jedis.sdiff(likeKey1, likeKey2));
    print(22, jedis.sinter(likeKey1, likeKey2));
    print(23, jedis.sismember(likeKey1, "12"));
    print(24, jedis.sismember(likeKey2, "16"));
    // 删除操作
    jedis.srem(likeKey1, "5");
    print(25, jedis.smembers(likeKey1));
    // 查看集合中有多少元素
    print(26, jedis.scard(likeKey1));

    /**
     * ZSET存储
     * ZSET 优先队列的实现 应用场景：排行榜实现
     */
    String rankKey = "rankKey";
    jedis.zadd(rankKey, 15, "jim");
    jedis.zadd(rankKey, 30, "Ben");
    jedis.zadd(rankKey, 60, "Lee");
    jedis.zadd(rankKey, 80, "Mei");
    print(30, jedis.zcard(rankKey));
    print(31, jedis.zcount(rankKey, 61, 100));
    // 查看指定用户的权重
    print(32, jedis.zscore(rankKey, "Lee"));
    jedis.zincrby(rankKey, 2, "Lee");
    print(33, jedis.zscore(rankKey, "Lee"));
    // 找出排名1到3的，默认从小到大排序
    print(34, jedis.zrange(rankKey, 1, 3));
    // 从大到小排序
    print(35, jedis.zrevrange(rankKey, 1, 3));
    // 取出分值在指定范围内的数据
    for (Tuple tuple : jedis.zrangeByScoreWithScores(rankKey, "60", "100")) {
        print(36, tuple.getElement() + ":" + String.valueOf(tuple.getScore()));
    }
    // 查看指定用户排名
    print(37, jedis.zrank(rankKey, "Ben"));
    print(38, jedis.zrevrank(rankKey, "Ben"));

    String setKey = "zset";
    jedis.zadd(setKey, 0, "a");
    jedis.zadd(setKey, 1, "a");
    jedis.zadd(setKey, 1, "b");
    jedis.zadd(setKey, 1, "c");
    jedis.zadd(setKey, 1, "d");
    jedis.zadd(setKey, 1, "e");
    jedis.zadd(setKey, 2, "f");
    // 字典序，-、+代表负无穷到正无穷
    print(39, jedis.zlexcount(setKey, "-", "+"));
    print(40, jedis.zlexcount(setKey, "[b", "[d"));
    // 大于b小于等于d
    print(41, jedis.zlexcount(setKey, "(b", "[d"));

    jedis.zrem(setKey, "b");
    print(42, jedis.zrange(setKey, 0, 10));
    // 删除大于c的
    jedis.zremrangeByLex(setKey, "(c", "+");
    print(43, jedis.zrange(setKey, 0, 2));
}

```

_ _ _ 
### **Redis单线程模型及Redis命令操作的原子性**  
Redis内部对于网络IO的处理采用IO多路复用方式，对于命令的执行采用单线程方式，所以每个命令的执行过程都可以说是原子性的，也就是说每个命令的执行过程从开始到结束不会有其它命令插入。  

为什么Redis使用单进程单线程模型效率还如此高，可以参考：[Redis单线程架构](http://www.leexide.com/%E6%95%B0%E6%8D%AE%E5%BA%93/redis/1269.html)，[Redis 为什么使用单进程单线程方式也这么快](http://www.syyong.com/db/Redis-why-the-use-of-single-process-and-single-threaded-way-so-fast.html)  

我的粗浅理解：对于网络IO，Redis采用IO多路复用方式提升其性能，对于单线程方式进行内存数据读写相比于多线程方式内存读写带来的好处是减少代码编写复杂度，避免了线程切换和竞态产生的消耗，也许相比于单线程内存读写，多线程切换和竞态产生的消耗可能会更大；单线程方式缺点是如果某个命令执行时间过长，会造成其它命令的阻塞。     

CPU基本不可能成为Redis的性能瓶颈，然而为了最大限度利用多核CPU，可以在单机上启动多个Redis实例，并把他们设置为不同服务器；要实现横向扩容，可以在多个机器上运行多个Redis实例建立Redis集群。    

_ _ _ 
### **Redis事务控制**
下面从事务的ACID四个方面来分析下Redis的事务。  

**原子性**：通常来说，原子性就是指事务包含的一系列的操作，要么全部都执行，要么都不执行，也就是说出错的情况下事务会回滚。而Redis的事务在EXEC命令调用之后，即使事务中的某些命令执行出错也会继续执行事务中包含的其它命令，并不会回滚。为什么Redis的事务不提供回滚功能呢？

我觉得要解决这个问题，首先要分析以下在传统关系型数据库中什么时候需要用到事务的回滚功能。我能想到的有以下几种情况：

1. 由于应用层传递给数据库的SQL语句的语法有错误，抛出SQLException。
2. 由于应用层传递给数据库的SQL语句的语义有错误，抛出SQLException。
3. 在事务执行过程中，程序自身抛出的异常导致事务需要回滚，比如程序抛出NPE。
4. 事务执行过程中，操作的数据库停止服务，数据库重新启动时之前未完成的事务需要回滚。

应用层事务在关系型数据库的执行方式与在Redis的执行方式是不同的，在关系型数据库中，事务语句可以一条一条的分别传送给数据库，并且应用层可以获得每条语句的执行结果，是交互式的事务执行；而在Redis中，给我的感觉更像是命令批处理方式的事务执行：应用层把整个事务中包含的命令一次发送到Redis Server，Redis Server执行完整个事务后返回给应用层每条命令的执行结果。  

对Redis操作需要回滚的情况与上述几种情况相似：  
1. 由于应用层传递给数据库的事务中包含的Redis命令的语法有错误，结果就是整个事务根本不会执行。  
2. 由于应用层传递给数据库的事务中包含的Redis命令的语义有错误，(比如尝试对一个普通的key-value使用lpush命令为其设置值)，事务中包含的所有命令仍会执行，执行失败的命令会返回错误提示。
3. 由于Redis事务执行过程中应用层与Redis Server是没有交互的，只有在应用层调用Exec()之后事务中包含的命令才会打包交给Redis Server进行处理，所以不存在事务执行过程中应用程序抛出异常的可能性。  
4. 某个事务执行过程中Redis崩溃，重启时数据是可能丢失的，Redis对这种情况的处理可以参见下文的Redis持久化。    

对于第2种情况Redis官方给出的解释是这种异常通常是由于程序员的编程错误导致的并且通常可以在开发过程中纠正，生产过程的产品一般不会出现这种问题。对于第4种情况与应用层无关，会交由Redis持久化机制来解决，数据可能会丢失，事务中完成的部分可能不会被回滚。  

Redis官方文档中对于事务不实现回滚还有一个理由是这样可以使得Redis内部实现更简单、更快速。  

由于Redis事务不支持回滚，对于那些对于事务中一致性要求较高的需求(比如转账)，是不能尝试使用Redis并通过Redis事务控制来保证数据安全性的。  

**一致性**：关系型数据库的一致性对事务一致性的要求包含两个层面，一是对数据完整性以及合法性的检查，二是要求开发者在代码中写出正确的事务逻辑，比如银行转账，事务中的逻辑不可能只扣钱或者只加钱，这是应用层面上对于数据库一致性的要求。  

相比于关系型数据库，Redis并没有类似于关系型数据库的实体完整性、参照完整性、用户定义的完整性等数据完整性层面上的要求，至于应用层面上的一致性，也要求开发者在操作Redis的过程中使用正确的事务逻辑，比如一个用户对于一个实体点赞之后要获取此用户点赞成功后当前实体下的所有点赞数，就要把点赞操作与获取点赞数操作包装在一个事务当中执行以保证事务逻辑上的一致性。  

**隔离性**：在关系型数据库中，由于事务之间可能存在并发操作，如果不对并发提交的事务进行控制，就可能出现脏读、不可重复读、幻读等问题，为了解决这些问题在SQL标准中定义了四种数据库的事务的隔离级别：READ UNCOMMITED、READ COMMITED、REPEATABLE READ 和 SERIALIZABLE。    

对于Redis，由于其内部的单线程模型，其事务的执行是串行化的，也就相当于上述事务隔离级别中SERIALIZABLE，所以不会存在脏读、不可重复读、幻读等问题。  

**持久性**： 事务的持久性体现在一旦事务被提交，那么数据一定会被写入到数据库中并持久存储起来。Redis事务持久性由其持久化机制保证。    

jedis事务使用代码示例：  
```java
public class JedisAdapter {
	public static void main(String[] args) {
	    Jedis jedis = new Jedis("redis://localhost:6379");
	    JedisAdapter jedisAdapter = new JedisAdapter();
	
	    Transaction tx = jedisAdapter.multi(jedis);
	    tx.set("num", "10");
	    tx.lpush("num", "8", "9");
	    tx.set("num", "5");
	    // 调用exec之后才会真正的将事务中包含的命令交由Redis服务器进行处理，之前命令暂存在客户端
	    List<Object> result = tx.exec();
	    if (Objects.nonNull(result)) {
	        for (Object o : result) {
	            if (o instanceof Exception) {
	                System.out.println("异常" + o);
	                continue;
	            }
	            System.out.println(o);
	        }
	    }
	}
	
	public Transaction multi(Jedis jedis) {
	    try {
	        return jedis.multi();
	    } catch (Exception e) {
	        logger.error("发生异常" + e.getMessage());
	    } 
	    return null;
	}
}
```
要想减少Redis的事务粒度，可以使用watch。比如如下方式来进行自增操作(假设Redis中没有INCR命令)：
```java
WATCH mykey
// 在watch之后，事务执行之前，包括当前连接的客户端在内的所有客户端不能有任何操作对mykey对应的
// value做改动，即使先将其设置为另一个值，之后在将其设置回原来的值，否则事务执行会失败
val = GET mykey
MULTI
SET mykey $val
EXEC

/* 使用watch实现zpop的示例 */
WATCH zset
element = ZRANGE zset 0 0
MULTI
ZREM zset element
EXEC
```
由上面应用方式可知，WATCH采用乐观锁方式，仅仅记录某个监听的值在事务执行之前是否被修改，并不阻止其他对监听的值的修改，在提交时检测此值是否被除了事务之外的其它行为修改过，如果有则此次事务失败，可以选择重新执行直到成功为止。  

对于某个事务，在使用multi开启事务之后，exec执行事务之前，如果程序出现异常，可以使用discard命令来终止当前事务。  
```java
public List<Object> exec(Transaction tx, Jedis jedis) {
    try {
        return tx.exec();
    } catch (Exception e) {
        logger.error("发生异常" + e.getMessage());
        tx.discard();
    } finally {
        if (tx != null) {
            try {
                tx.close();
            } catch (IOException ioe) {
                // ..
            }
        }
    }
    return null;
}
```
本部分参考文章：[『浅入深出』MySQL 中事务的实现](https://draveness.me/mysql-transaction)，[Redis Transactions官方文档](https://github.com/antirez/redis-doc/blob/master/topics/transactions.md)  

**待解决问题**：关系型数据库事务实现原理？两段锁协议？MVCC？   

_ _ _  
### **Redis连接池**
在使用Jedis操作Redis时，我们通常需要用到Jedis的连接池来复用连接提升性能，具体使用方式及配置如下：  
```java
JedisPoolConfig poolConfig = new JedisPoolConfig();

/*资源设置和使用*/

// 设置连接池中可含有的最大连接数,maxTotal = Idle连接 + Active连接;
poolConfig.setMaxTotal(maxTotal);
// 控制一个pool最多有多少个状态为idle的jedis实例
poolConfig.setMaxIdle(maxIdle);
// 控制一个pool最少有多少个状态为idle的jedis实例，当后台监测线程周期检测时如果检测到池中idle连接数小于minIdle，
// 则会创建新的连接直到大于等于minIdle，但总数不会超过maxTotal
poolConfig.setMinIdle(minIdle);
// 当资源池连接用尽后，调用者的最大等待时间(单位为毫秒)
poolConfig.setMaxWaitMillis(maxWaitMillis);
// 向资源池借用连接时是否做连接有效性检测(ping)，无效连接会被移除
poolConfig.setTestOnBorrow(testOnBorrow);
// 向资源池归还连接时是否做连接有效性检测(ping)，无效连接会被移除
poolConfig.setTestOnReturn(testOnReturn);

/**
 * 空闲资源监测，上面有个参数是minIdle，需要注意连接池中的Idle连接数并不是在任何时候都大于minIdle的，
 * 后台监测线程会周期性的监测池中线程数，监测到当前Idle连接数小于于minIdle时才会创建新的连接，
 * 在后台线程一次监测完成之后到下一次监测开始期间，池中Idle连接数是很有可能小于minIdle的。  
 */

// 是否在后台开启空闲资源监测
poolConfig.setTestWhileIdle(testWhileIdle);
// 资源池中连接保持空闲而不被驱逐的最长时间(单位为毫秒)，达到此值后空闲连接将被移除
poolConfig.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
// 空闲资源的检测周期(单位为毫秒)
poolConfig.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
// 做空闲资源检测时，每次的采样数, 如果设置为-1，就是对所有连接做空闲监测
poolConfig.setNumTestsPerEvictionRun(numTestsPerEvictionRun);

JedisPool pool = new JedisPool(jedisPoolConfig, "127.0.0.1", 6379);

Jedis jedis = pool.getResource();

...
// 使用jedis对象对redis进行相关操作
...

// 在使用连接池的情况下，close()一个连接实际是把此连接交还给线程池。  
jedis.close();

```
上述设置中一种配置方案如下：
```java
RedisConfig.maxTotal=100
RedisConfig.maxIdle=100
RedisConfig.minIdle=30
#当池内没有返回对象时，最大等待时间,ms为单位
RedisConfig.maxWaitMillis=20000
#向资源池借用连接时是否做连接有效性检测(ping)，无效连接会被移除
RedisConfig.testOnBorrow=true
#向资源池归还连接时是否做连接有效性检测(ping)，无效连接会被移除
RedisConfig.testOnReturn=true

#是否开启空闲资源监测
RedisConfig.testWhileIdle=true
#空闲资源的检测周期(单位为ms)
RedisConfig.timeBetweenEvictionRunsMillis=30000
#资源池中连接保持空闲而不被驱逐的最长时间(单位为毫秒)，达到此值后空闲连接将被移除
RedisConfig.minEvictableIdleTimeMillis=60000
#做空闲资源检测时，每次的采样数
RedisConfig.numTestsPerEvictionRun=-1
```
需要注意的是，如果采用JedisPool pool = new JedisPool(jedisPoolConfig, host, port)这种方式创建连接池，对于空闲资源监测的相关配置可以使用jedisPoolConfig中的默认设置：  
```java
public class JedisPoolConfig extends GenericObjectPoolConfig {
  public JedisPoolConfig() {
    // defaults to make your life with connection pool easier :)
    setTestWhileIdle(true);
    setMinEvictableIdleTimeMillis(60000);
    setTimeBetweenEvictionRunsMillis(30000);
    setNumTestsPerEvictionRun(-1);
  }
}
```
**RedisPool预热**：由于一些原因(例如超时时间设置较小原因)，有的项目在启动成功后会出现超时。JedisPool定义最大资源数、最小空闲资源数时，不会真的把Jedis连接放到池子里，第一次使用时，池子没有资源使用，会new Jedis，使用后放到池子里，可能会有一定的时间开销，所以也可以考虑在JedisPool定义后，为JedisPool提前进行预热，例如以最小空闲数量为预热数量：  
```java
List<Jedis> minIdleJedisList = new ArrayList<Jedis>(jedisPoolConfig.getMinIdle());

for (int i = 0; i < jedisPoolConfig.getMinIdle(); i++) {
    Jedis jedis = null;
    try {
        jedis = pool.getResource();
        minIdleJedisList.add(jedis);
        jedis.ping();
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    } finally {
    }
}

for (int i = 0; i < jedisPoolConfig.getMinIdle(); i++) {
    Jedis jedis = null;
    try {
        jedis = minIdleJedisList.get(i);
        jedis.close();
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    } finally {
    
    }
}
```
为了减少与Redis交互过程中由于网络IO带来的时间延迟，可以使用Redis提供的**PileLine**方式发送命令，提升交互效率。具体可参考：[Redis Pipeline原理分析](http://www.cnblogs.com/jabnih/p/7157921.html)  

本部分参考文章：[JedisPool资源池优化](https://yq.aliyun.com/articles/236383)，[Jedis源码阅读之连接池](https://www.jianshu.com/p/80ce05090def)  

**待解决问题**：有时间时可阅读分析Jedis连接池相关源码。  

_ _ _ 
### **Redis持久化**
从使用层面来讲我认为可以阅读官网上的这篇文章，讲的十分清晰：[Redis Persistence官方文档](https://github.com/antirez/redis-doc/blob/master/topics/persistence.md)  
[《Redis官方文档》持久化译文](http://ifeve.com/redis-persistence/)  

Redis持久化机制默认仅采用RDB方式，可以同时开启RDB和AOF模式，也可以将其全关闭。  
 
上面文章中提到了一个RDB持久化方式的劣势：Redis在使用RDB方式进行数据持久化时通常需要使用fork()系统调用创建子进程来进行真正的磁盘IO操作，如果数据集很大的话，fork()比较耗时，结果就是，当数据集非常大并且CPU性能不够强大的话，Redis会停止服务客户端几毫秒甚至一秒。

我认为fork操作在这里耗时的原因可能是虽然使用了copy-on-write技术，但是子进程至少需要复制父进程一级页表等一些资源，当数据集很大，CPU或内存的性能不够好时，这个操作可能会很耗时，造成Redis对客户端停止服务甚至到1s，AOF方式仅在rewrite log的时候调用fork，可以通过控制AOF方式的频率来尽量减少fork操作对于Redis服务的影响。   

**待解决问题**：当前Redis中存储的数据量很大，每次RDB都会重写所有内存中数据到文件吗？这样RDB同步数据到磁盘太频繁怎么办？   

_ _ _ 
### **Redis扩容**
[《Redis官方教程》-FAQ](http://ifeve.com/redis-persistence/)有下面这样一段话：  
>过去为了允许数据集超过RAM大小，Redis开发人员尝试使用虚拟内存和其他系统，但是我们非常高兴可以把一件事情做好：数据服务由内存提供，磁盘用于存储数据。所以现在没有计划为Redis创建磁盘后端，毕竟Redis大部分特性都是基于其当前架构设计的。  

所以要想给Redis扩容，并不能局限于单机，需要对Redis进行分片。分片可以从三个层次上来实现：客户端分区、代理辅助分区、查询路由分区。  

* 客户端分区：比如客户端采用一致性哈希算法实现。
* 代理辅助分区：比如代理软件Twemproxy。
* 查询路由分区：比如Redis支持的集群特性的实现，混合使用了查询路由和客户端分区。

具体可参考：[《Redis官方文档》分区](http://ifeve.com/redis-partitioning/)  

**待解决问题**：Redis集群实现采用的Hash算法与一致性Hash算法对比。   

_ _ _ 
### **Redis相关配置**
在redis.conf文件中更改。
* daemonize：守护进程，默认为no。
* maxmemory及maxmemory-policy：设置redis可使用的最大内存及到达最大内存之后的Redis的行为。不配置的话可能导致堆内存用光内后redis由于oom异常崩溃。 
* logfile：配置日志记录方式
* requirepass：自定义连接密码
* pidfile：指定Redis PID写入哪个文件
* bind：绑定的主机地址
* port：指定Redis监听端口

启动方式： ./redis-server redis.conf  

本部分参考：[Redis configuration](https://github.com/antirez/redis-doc/blob/master/topics/config.md)  

_ _ _ 
### **Redis应用之赞踩实现**   
为什么要使用Redis实现赞踩？  
 
1. 每个被点赞的项目点赞数量可能比较大，属于写热点数据，并且仅存储点赞数据需要存储的数据量比较小，使用Redis在内存中存储点赞数据可以减轻DB的写压力。
2. 每个用户点赞之前需要判断这个用户之前是否点过赞，使用Redis的Set结构存储可以使此操作时间复杂度降到O(1)，在点赞数量较多时也可以保证此操作的高效性。用户浏览页面时需要显示当前展示的每个项目的点赞数量的总数，可通过SCARD命令获取Set的size，此操作时间复杂度也为O(1)。
3. Redis提供的数据持久化机制使数据安全性基本可以得到保障。  

_ _ _ 
### **Redis应用之异步事件处理**  
异步事件处理可以使得用户端可以迅速的得到操作的返回结果。比如微博大v粉丝几千万，发了一条微博，这个事件可能带来一系列的影响，(比如先将这条微博存入数据库，之后使用feed流的推模式把这条微博推给他的活跃粉丝或者在线粉丝等等)，如果等到这一系列操作都执行完之后才将结果返回给用户显然会影响用户体验，我们可以只做必要的事，比如将这条微博记录存入数据库就返回，将feed流推送记录为一个事件存入队列由后台处理线程稍后处理。  

异步事件队列可以多种方式实现，比如Java中的BlockingQueue、ActiveMQ、Redis等等。相比于在在程序内部使用BlockingQueue实现的异步事件队列，使用ActiveMQ、Redis等中间件实现显然能够更好的解除耦合，更加稳定且容易扩容。异步事件队列实现思想如下图：  
<img src="/img/2018-3-3/AsyncEventQueue.jpg" width="550" height="550" alt="AsyncEventQueue" />
<center>图3：异步事件队列实现思想简图</center> 

接下来就来介绍下结合Spring使用Redis实现的简单的异步事件处理框架，具体解释见代码中介绍。   
```java
/**
 * 使用此类统一封装各种事件模型
 */
public class EventModel {

    /**
     * 事件类型，比如点赞
     */
    private EventType eventType;

    /**
     * 事件发起者，点赞的用户Id
     */
    private int actorId;

    /**
     * 事件接收实体Id，比如用户A给用户B在某个问题下的评论点了赞，entityId就是用户B这个评论的Id
     */
    private int entityId;

    /**
     * 事件接收实体类型，比如用户A给用户B在某个问题下的评论点了赞，entityType就是评论对应的实体类型
     */
    private int entityType;

    public EventModel() {}

    public EventModel(EventType eventType) {
        this.eventType = eventType;
    }
    
    public int getActorId() {
        return actorId;
    }

    public EventModel setActorId(int actorId) {
        this.actorId = actorId;
        return this;
    }
    // 省略其它getter和setter...
}
/**
 * 使用此类封装各种事件类型
 */
public enum EventType {

    LIKE(0),
    FOLLOW(1);

    private int value;
    EventType(int value) { this.value = value; }
    public int getValue() { return value; }
}
/**
 * 使用此类封装各种实体类型
 */
public class EntityType {
    public static int ENTITY_QUESTION = 1;
    public static int ENTITY_COMMENT = 2;
    public static int ENTITY_USER = 3;
}
/**
 * 生产者，产生事件后使用此类将事件推入Redis中事件队列
 */
@Service
public class EventProducer {

    @Autowired
    private JedisAdapter jedisAdapter;

    public void fireEvent(EventModel eventModel) {
        // 直接将事件推进事件队列
        jedisAdapter.lpush(RedisKeyUtil.getEventQueueKey(), JSONObject.toJSONString(eventModel));
    }
}
/**
 * 事件处理器接口规定
 */
public interface EventHandler {

    void doHandle(EventModel eventModel);

    List<EventType> getSupportEventTypes();
}
/**
 * 点赞事件处理器示例实现
 */
@Component
public class LikeHandler implements EventHandler {
    @Override
    public void doHandle(EventModel eventModel) {
        System.out.println("LikeHandler中开始异步处理点赞事件带来的影响！");
        System.out.println("事件类型：" + eventModel.getEventType() + " 事件发起用户ID: " +
                eventModel.getActorId() + " 被点赞实体类型： " +
                eventModel.getEntityType() + " 被点赞实体ID: " + eventModel.getEntityId());
    }

    @Override
    public List<EventType> getSupportEventTypes() {
        return Arrays.asList(EventType.LIKE);
    }
}
/**
 * 消费者，要完成的任务包含两个部分：  
 *  1、存储每种事件与可以处理这个事件的处理器之间的对应关系
 *  2、初始化后台线程阻塞读取Redis中的事件队列，取出事件并交由相应的处理器进行处理
 */
@Service
public class EventConsumer implements InitializingBean, ApplicationContextAware {

    private static final Logger LOGGER = LoggerFactory.getLogger(EventConsumer.class);
    private ApplicationContext applicationContext;
    private Map<EventType, List<EventHandler>> config = new HashMap<>();

    @Autowired
    private JedisAdapter jedisAdapter;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Map<String, EventHandler> handlerMap = applicationContext.getBeansOfType(EventHandler.class);
        for (Map.Entry<String, EventHandler> entry : handlerMap.entrySet()) {
            List<EventType> supportEventTypes = entry.getValue().getSupportEventTypes();
            for (EventType eventType : supportEventTypes) {
                if (!config.containsKey(eventType)) {
                    config.put(eventType, new ArrayList<>());
                }
                config.get(eventType).add(entry.getValue());
            }
        }

        new Thread(new Runnable() {
            private String key = RedisKeyUtil.getEventQueueKey();
            @Override
            public void run() {
                while (true) {
                    // 若没有事件产生则阻塞
                    List<String> events = jedisAdapter.brpop(0, key);
                    // brpop命令返回的list中第一项为客户端指定的key，第二项为pop出来的key对应的value
                    for (String message : events) {
                        if (message.equals(key)) {
                            continue;
                        }
                        EventModel model = JSON.parseObject(message, EventModel.class);
                        if (!config.containsKey(model.getEventType())) {
                            LOGGER.error("不能识别的事件类型");
                            continue;
                        }
                        for (EventHandler handler : config.get(model.getEventType())) {
                            handler.doHandle(model);
                        }
                    }
                }
            }
        }).start();
    }
}
/**
 * 测试类
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class AsyncEventQueueTests {

    @Autowired
    private EventProducer eventProducer;

    @Test
    public void testLike() throws InterruptedException {
        int userId = 1, commentId = 13;
        eventProducer.fireEvent(new EventModel(EventType.LIKE).setActorId(userId)
                .setEntityType(EntityType.ENTITY_COMMENT).setEntityId(commentId));
        System.out.println("点赞结束！");
        Thread.sleep(30000);
    }
}

输出结果：  
    点赞结束！
    LikeHandler中开始异步处理点赞事件带来的影响！
    事件类型：LIKE 事件发起用户ID: 1 被点赞实体类型： 2 被点赞实体ID: 13
``` 
_ _ _ 
(完)

参考文章：  
[Java并发编程网Redis官方教程系列翻译](http://ifeve.com/red-faq/)  
待阅读书籍：《Redis设计与实现》 BY 黄健宏