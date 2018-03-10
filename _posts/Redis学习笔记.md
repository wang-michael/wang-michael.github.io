#Redis学习笔记
推荐redis设计与实现一书
##Redis简介 
* 内存中存储数据  
* key-value存储，不同于传统关系型数据库  
* 支持主从同步

##Redis相关配置
windows版本在redis.windows.conf配置文件中存储
* databases 16 可以选16个数据库？类似于mysql数据库概念，在redis命令行下默认操作的是第0个数据库，可以通过Select + 数据库id来切换到指定的数据库
* save 900 1 至少有1个key改变之后过900s会进行一次持久化
* save 300 10 至少有10个key改变之后过300s会进行一次持久化
* save 60 10000 至少有10000个key改变之后过60s会进行一次持久化
* dbfilename dump.rdb 默认持久化数据存储在当前文件夹下的此文件中
* AOF与RDB：redis的两种不同持久化存储方式，RDB方式保存更改后的最终结果，AOF保存操作使用过的所有命令
* redis五种数据结构：String， Hash， List, Set， ZSet， 
* String：一个key对应一个value，例如 key1 -> value1 
* Hash: 一个key对应一个map，map中最多可以存储 2^32 - 1 键值对（40多亿）。  
* List: 一个key对应一个list，比如key1 -> [value1, value2, value3 ...],list按插入时间顺序保存对应的value值，比如value1即为插入时间与当前时间最接近的值
* Set： 一个key对应一个Set， 每个集合中最大的成员数为 2^32 - 1 (4294967295, 每个集合可存储40多亿个成员)。
* ZSet: 一个key对应一个有序的Set，按照插入时对key指定的score进行排序，比如插入时指定ZADD runoobkey 3 mysql，ZADD runoobkey 1 redis，则ZRANGE runoobkey 0 10 WITHSCORES返回结果中redis在前，mysql在后。    
  
##Redis-KV

##Redis-List

##Redis-Zset

##Redis-Hash

##赞踩实现

##redis主从同步、读写分离

##redis实现Geo相关应用

##redis事务控制

##redis连接池

##基于redis实现的异步事件处理机制

##使用redis缓存数据与在进程中直接使用blockingQueue，list等数据结构缓存数据相比有何好处？  
feed流推数据使用redis中list缓存每个用户的timeline；推数据存储压力大，拉数据读取压力大