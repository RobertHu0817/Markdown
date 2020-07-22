# Redis

## 数据类型

  1. *string*，字符串

     底层使用Redis自己构建的SDS（Simple Dynamic String，简单动态字符串）类型[^1]，二进制安全

  2. *hash*，哈希

  3. *list*，列表

  4. *set*，集合

  5. *zest*，有序集合（sorted set）

## 默认16个数据库（0-15）

  * Redis是一个字典结构的存储服务器，而实际上一个Redis实例提供了多个用来存储数据的字典，客户端可以指定将数据存储在哪个字典中。这与我们熟知的在一个关系数据库实例中可以创建多个数据库类似，所以可以将其中的每个字典都理解成一个独立的数据库
  * Redis不支持自定义数据库的名字，每个数据库对外都是一个从0开始的递增数字命名，Redis默认支持16个数据库，可以通过配置文件支持更多，无上限。客户端与Redis建立连接后会自动选择0号数据库，可使用 *SELECT* 命令更换数据库
  * Redis不支持为每个数据库设置不同的访问密码
  * 多个数据库之间并不是完全隔离的，比如 *FLUSHALL* 命令可以清空一个Redis实例中所有数据库中的数据
  * 单机才有

## RedisDb结构体

  ```c
  typedef struct redisDb {
          dict *dict;                 /* 数据库的键空间，保存数据库中的所有键值对 */
          dict *expires;              /* 保存所有有过期时间的键 */
          dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
          dict *ready_keys;           /* Blocked keys that received a PUSH */
          dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
          int id;                     /* 数据库ID字段，代表不同的数据库 */
          long long avg_ttl;          /* Average TTL, just for stats */
  } redisDb;
  ```

## Redis过期策略

过期策略通常有以下三种：

  1. 定时删除

     为每一个设置了过期时间的key创建一个Timer，到过期时间时就会立即清除

     内存友好，但是会占用大量CPU资源去处理过期数据，从而影响缓存的响应时间和吞吐量

  2. 惰性删除

     当一个key被访问时，程序会对这个key进行检查，如果key已经过期，那么该key将被删除

     最大化节省CPU资源，但对内存不够友好。极端情况可能出现大量过期key因为不再使用而未被及时清理，占用大量内存

  3. 定期删除

     每隔一段时间扫描数据库，删除已过期的key

     前两者的折中方案，通过调整定时扫描的时间间隔和每次扫描的限定耗时，可以在不同情况下使得CPU和内存资源达到最优的平衡效果



Redis采用了惰性删除+定期删除的方案。其中定期删除是随机处理一部分key——不是所有，删除过期key的执行步骤如下（函数 *activeExpireCycle(int type)* ）：

  1. 依次遍历所有的DB
  2. 从DB的过期列表（设置了过期时间的key）中随机取20(**有文章说是100个。。**)个key，判断是否过期，如果过期则清理
  3. 如果有25%（5个）以上的key过期，则重复步骤2
  4. 如果过期率低于25%则继续处理下一个DB
  5. 整个过程中，一旦清理时间达到timelimit则退出整个清理过程



由于Redis是单线程，对key的清理不能占用过多的时间和CPU，否则会导致Redis无法提供正常服务，所以Redis设有一个执行时间上限 *timelimit*，它的值由执行函数时参数指定的模式决定：

  * **ACTIVE_EXPIRE_CYCLE_FAST**，“快速过期”模式，函数的执行时限 `timelimit = EXPIRE_FAST_CYCLE_DURATION μs`，并且在 *EXPIRE_FAST_CYCLE_DURATION* 之内不会再重新执行。
  * **ACTIVE_EXPIRE_CYCLE_SLOW**，“正常过期”模式，函数的执行时限 `timelimit = 1000000 * ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC / server.hz / 100 μs`

  >1. *EXPIRE_FAST_CYCLE_DURATION* 默认值 1000(μs)
  >2. *ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC* 默认值 25
  >3. *server.hz*，定期删除任务的执行频率，默认是每秒10次（配置文件:hz 10，修改的话Redis作者建议不要超过100），使用“正常过期”模式执行activeExpireCycle
  >4. 每一次Redis访问事件到来，除了执行惰性删除，还会使用“快速过期”模式执行activeExpireCycle,在 *EXPIRE_FAST_CYCLE_DURATION* 的时间内主动清理部分过期key（**TODO 无论什么请求都会触发吗，还是只有访问key乃至只有访问设置了过期时间的key才触发？**）
  >5. 总结：默认情况下，启动Redis，定期删除过期key的定时任务启动，每 1 / 10 秒执行一次，每次最长执行 25% * 1 / 10 秒。每次依次遍历各个库，如果某库的过期率大于 25% 则继续处理此库直至过期率低于 25% 后处理下一个库 ；每次有访问key的请求来临，Redis会判断key是否过期并作出处理，同时还会执行1ms删除过期key任务

## **Redis内存淘汰策略**

当Redis使用内存超过配置文件 maxmemory 的设定，对于所有的读写请求，都会触发 *freeMemoryIfNeeded(void)* 函数以清理超出的内存，而且这个清理过程是阻塞的。每次清理并不针对某个库的所有key，而是以配置文件中的 *maxmemory-samples*[^2] 个key作为样本池进行抽样清理。清理策略来源于配置文件的 *maxmemory-policy* ：

  * **noeviction**，新写入操作报错
  * **allkeys-random**，在键空间中，随机移除某个key
  * **allkeys-lru**，在键空间中，移除最近最少使用的key
  * **volatile-random**，在设置了过期时间的键空间中，随机移除某个key
  * **volatile-lru**，在设置了过期时间的键空间中，移除最近最少使用的key
  * **volatile-ttl**，在设置了过期时间的键空间中，有更早过期时间的key优先移除

  >TTL：Time To Live，生存时间
  >LRU：Least Recently Used，最近最少使用

## Redis删除key

  使用 *del* 指令删除一个非常大的key，或者使用 *flushdb*、 *flushall* 指令删除包含大量key的数据库时，会导致redis阻塞；另外reids在清理过期key和淘汰内存超限数据时，如果碰巧撞到了大体积的key时同样也会造成阻塞。
  为此，Redis4.0之后引入了 **lazyfree** 的机制，可以将删除键/库的操作交由后台子线程（BIO,Background I/O）[^3]执行，从而有效避免主线程阻塞。

***lazyfree 机制的使用：***

  * 主动删除
  1. 新增*unlink* 指令，如果删除对象大小超过设定值（包含元素个数大于 *LAZYFREE_THRESHOLD*(64) ），它就将删除操作丢给后台线程，由后台线程来异步回收内存
  2. 为 flush类命令新增option：async，当flush命令后跟 async 选项时就会进入后台删除
  * 被动删除
    新增4个后台删除配置项——默认都是关闭（no）
  1. **lazyfree-lazy-eviction**

     针对内存淘汰策略；开启可能导致淘汰key的内存释放不及时，导致redis内存超用，不建议开启

  2. **lazyfree-lazy-expire**

     针对设置有TTL的键

  3. **lazyfree-lazy-server-del**

     针对部分带有隐式 `del` 操作的指令（如 `rename` 命令，当目标键已存在，将先删除目标键）

  4. **slave-lazy-flush**

     针对slave进行全量数据同步——slave在加载master的RDB文件前会运行 `flushall` 来清理自身当前数据

## AOF、RDB和复制功能对过期键的处理

  * Redis 支持两种不同的持久化操作：

      1. 快照（snapshotting，RDB）持久化：

         Redis默认采用的持久化方式，在redis.conf配置文件中默认有此下配置：

         ```
         save 900 1       #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
         save 300 10      #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
         save 60 10000    #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
         ```

         

      2. 只追加文件（append-only file，AOF）持久化：

         与快照持久化相比，AOF持久化 的实时性更好，因此已成为主流的持久化方案。默认情况下Redis没有开启
         AOF（append only file）方式的持久化，可以通过appendonly参数开启：

         ```
         appendonly yes
         ```

         开启 AOF 持久化后每执行一条会更改 Redis中 的数据的命令，Redis就会将该命令写入硬盘中的 AOF 文件。AOF 文件的
         保存位置和 RDB 文件的位置相同，都是通过 dir 参数设置的，默认的文件名是 appendonly.aof。

         在Redis的配置文件中存在三种不同的 AOF 持久化方式，它们分别是：

         ```
         appendfsync always   #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
         appendfsync everysec #每秒钟同步一次，显示地将多个写命令同步到硬盘
         appendfsync no   #让操作系统决定何时进行同步
         ```

  * Redis 4.0 对于持久化机制的优化

      Redis 4.0 开始支持 RDB 和 AOF 的混合持久化（默认关闭，可以通过配置项 aof-use-rdb-preamble 开启）。如果把混合持久化打开，AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头。这样做的好处是可以结合 RDB 和 AOF 的优点, 快速加载同时避免丢失过多的数据。当然缺点也是有的，AOF 里面的 RDB 部分是压缩格式不再是 AOF 格式，可读性较差。

  * AOF 重写
AOF重写可以产生一个新的 AOF 文件，这个新的 AOF 文件和原有的 AOF 文件所保存的数据库状态一样，但体积更小。AOF 重写是一个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序无须对现有 AOF 文件进行任伺读入、分析或者写入操作。在执行 BGREWRITEAOF 命令时，Redis 服务器会维护一个 AOF 重写缓冲区，该缓冲区会在子进程创建新 AOF 文件期间，记录服务器执行的所有写命令。当子进程完成创建新 AOF 文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新 AOF 文件的末尾，使得新旧两个 AOF 文件所保存的数据库状态一致。最后，服务器用新的 AOF 文件替换旧的 AOF 文件，以此来完成 AOF 文件重写操作
  
  * 生成 RDB 文件
    
       执行 *save* 或 *bgsave* 时，已过期的键不会被保存到新建的RDB文件中   
       
        *save*：同步，阻塞Redis进行持久化
       
        *bgsave*：异步，非阻塞，可以一边持久化一边对外提供读写服务，新写的数据对持久化不可见。原理：
   
    1. fork()：fork出子进程准备持久化，而后将主进程中所有内存页权限设置为read-only，子进程数据地址指向主进程相同的内存空间；
    2. copyOnWrite：主进程写时，CPU检测到内存页是read-only，触发页异常中断，此时内核就会把触发异常的页复制一份（而非所有数据），主进程此数据的指针指向新的拷贝出的地址，在其上进行操作，此操作对子进程不可见
  * 载入RDB文件
    1. Master 载入RDB时，已过期的键则会被忽略
    2. Slave 载入RDB时，文件中的所有键都会被载入。但因为主从服务器在进行数据同步的时候，从服务器的数据库会被清空，最终与 Master 一致，所以过期键对载入RDB文件的从服务器也不会造成影响
    
  * AOF文件写入
    当过期键被惰性删除或者定期删除时，Redis会追加一条del命令到AOF文件中，显式地记录该键已被删除
    
  * 重写AOF文件
    执行 *BGREWRITEAOF* 时，已过期的键不会被保存到AOF文件中
    
  * 复制
    当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制[^4]：
    1. Master 删除过期键后，会向所有 Slave 服务器发送一个 *del* 命令，告知从服务器删除这个过期键
    2. Slave 在被动的读取过期键时，不会做出操作，而是继续返回该键
    3. Slave 只有在接到 Master 发送来的 *del* 命令之后，才会删除过期键

## Redis集群

1. **Redis Cluster（官方集群方案）**
   
   * Redis3.0推出，采用去中心化结构，是一种服务端分片技术
   * 每个集群节点都需要开启两个TCP链接
     1. 用于服务客户端的常规TCP端口，如6379
     2. 用于集群总线（Cluster bus）的特殊端口，端口号为常规端口号加10000，如16379
   * Sharding没有使用一致性哈希算法，而是采用hash slot(哈希槽)的概念，划分出 [0,16383] 共16384个槽，对每个进入Redis的键值对，根据 *CRC16(KEY) mod 16384* 的结果将其放入对应的hash slot中
   * 特殊的，使用了hash tag的 KEY——`a{b}c`（使用左右中括号包裹了部分字符），其所属hash slot的计算方式为 *CRC16(b) mod 16384* 。通过这种方式可以实现将一类key放置在同一个hash slot中的需求。
   * 集群中的每个节点负责一部分hash slot，且由于将hash slot由一个节点移动到另一个节点不需要停止其它操作，所以添加/删除节点或者更改节点持有的hash slot的百分比时不需要任何停机时间
   * 节点间通过集群总线使用一种特殊高效的二进制协议进行通信
   * 对客户端来说整个集群被视作一个整体，客户端可以连接任意一个节点进行操作（不需要中间proxy层），当客户端操作的key没有分配到该node时，Redis会返回转向指令指向正确的node（类似网页的302 redirect跳转）
   * 集群不支持需要同时处理多个key的Redis命令，因为这些key可能分布在多个节点之上
   * 集群通过master节点投票判断是否有节点挂掉——如果半数以上的master节点与A节点通信超时（通过集群总线），即认为A节点挂掉。A节点对应的hash slot无法使用，导致整个集群无法正常工作
   * 为了增加集群的可用性，官方推荐将节点配置成主从结构——一个master主节点，挂n个slaver从节点。当某个主节点失效时，集群会根据选举算法由其从节点中选择一个上升为主节点，整个集群继续正常工作
   * 由于选举算法+主从结构的设计，Redis Cluster至少需要三主三从共6台服务器
   * 节点主从结构下，两种情况会导致整个集群不可用：
     1. 集群某主节点挂掉，且其下没有可用的从节点，集群进入fail状态
     2. 集群超过半数的主节点挂掉，无论是否有从节点，集群进入fail状态
   * 集群无法保证强一致性——例如，由于集群节点主从间使用异步复制（主节点写操作成功后将直接答复客户端“确认”并写操作传播到其从节点——回复客户端前不会等待从节点的确认-这可能导致极高的延迟），当主节点在确认写操作后由于崩溃导致未能发送写操作至从节点，而此从节点又升级为主节点，从而永远丢失此次写操作
   
2. **Redis Sharding**

   * 一种客户端分片技术——分片逻辑由客户端实现，基于此分片机制的开源产品不多见，Jedis已经支持——
   
     ShardedJedis
   
   * 采用一致性哈希算法
   
   * ShardedJedis会对每个Redis节点根据名字(没有，Jedis会赋予缺省名字)会虚拟化出160个虚拟节点进行散列。根据权重weight，也可虚拟化出160倍数的虚拟节点。
   
   * ShardedJedis支持keyTagPattern模式，即抽取key的一部分keyTag做sharding，这样通过合理命名key，可以将一组相关联的key放入同一个Redis节点，这在避免跨节点访问相关数据时很重要。
   
3. **twemproxy**

   * 利用代理中间件实现集群
   
6. **Codis**

   

## 常用命令

```bash
#redis安装目录/src下连接redis	h:ip|p:端口号|a:密码
redis-cli -h 192.168.0.10 -p 6380 -a sabc#DAd8763249
#redis根目录/src目录下直接连接此redis
./redis-cli
#直接连接操作时可能提示没有权限： NOAUTH Authentication required，可以输入auth password
auth sabc#DAd8763249
#查看redis内存使用情况
info memory
#选择库
SELECT 10
#测试与服务器的连接是否仍然生效，或者用于测量延迟值(正常的话，会返回一个 PONG )
PING
#停止
shutdown
#退出客户端
exit
#redis安装目录/src下启动
nohup redis-server redis.conf &

#键空间通知 配置文件notify-keyspace-events "" 改为notify-keyspace-events Ex 可使redis在键失效时发出通知；或者使用以下命令-问题：不支持集群、读写分离
config set notify-keyspace-events Ex

#Redis本身提供了基准测试命令redis-benchmark，使用redis-benchmark可以进行压力测试，了解当前Redis环境下的性能表现
redis-benchmark常用参数：
-h <hostname> 主机名 (缺省 127.0.0.1)
-p <port> 端口号 (缺省 6379)
-a <password> 登录口令(如Redis设置，缺省无)
-c <clients> 并发连接数 (缺省 50)
-n <requests> 总请求数 (缺省 100000)
-d <size> SET/GET操作数据值字节大小(缺省 2)
-r <keyspacelen> SET/GET/INCR操作键值以及SADD操作数据值的随机选择范围。
-t <tests> 测试的操作集，用逗号分隔，缺省都执行


#创建/修改/删除/查看键值
#[string]
SET key value
SETNX key value（如果键已存在则不执行）
DEL key
SETEX temp 10
MSET key1 value1 key2 value2（存在的键会被覆盖）
MSETNX key1 value1 key2 value2（如果某个键已存在则不执行-要么全部设置，要么都不设置）
MGET key1 key2
#[hash]
HSET key field1 value1
HSETNX key field1 value1（如果键已存在则不执行）
HEXISTS key field1
HGET key field1
HMSET key field1 value1 field2 value2...（存在的field会被覆盖，不存在的无影响）
HMGET key field1 field2...
HKEYS key（返回所有field）
HVALS key（返回所有value）
HGETALL key（返回所有field & value）
HDEL key field1 field2...
HINCRBY key field increment（可为负数）
#[list]
LPUSH key value1 value2...（依次插入到表头）
LPUSHX key value1 value2...（key不存在时不执行）
RPUSH key value1 value2...（依次插入到表尾）
RPUSHX key value1 value2...（key不存在时不执行）
LSET key index value
LINSERT key BEFORE|AFTER pivot value
LINDEX key index
LPOP key（移除并返回列表头元素）
RPOP key（移除并返回列表尾元素）
RPOPLPUSH source destination（1.移除并返回source列表尾元素2.将其插入到destination列表表头）
LREM key count value
LRANGE key start stop（闭区间）
LTRIM key start stop
BLPOP/BRPOP key1 key2 timeout（当给定所有key都处于不存在或者为空列表的状态时，连接将被 BLPOP 命令阻塞，直到等待超时或发现可弹出元素为止）
BRPOPLPUSH source destination timeout
#[set]
SADD key member1 member2...
SISMEMBER key member1
SMEMBERS key
SREM member1 member2
SCARD key（返回列表中元素数量）
SINTER key1 key2...（返回指定集合的交集）
SINTERSTORE destination key1 key2...（交集结果保存至destination）
SUNION key1 key2...（返回给定集合的并集）
SUNIONSTORE destination key1 key2...
SDIFF key1 key2...（返回给定集合的差集-以第一个为标准）
SDIFFSTORE destination key1 key2...
#[zset]
ZADD key score1 member1 score2 member2...
ZSCORE key member
ZINCRBY key increment member（increment可为负数）
ZCARD key（返回集合中元素数量）
ZCOUNT key min max（min、max指分数区间）
ZRANGE key start stop [WITHSCORES]（start、stop为下标）
ZREVRANGE key start stop [WITHSCORES]
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
ZREVRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]



#value递增
INCR key
INCRBY key 7
INCRBYFLOAT key 7.7
#value递减
DECR key
DECRBY key 7

#查找以‘LEAPTCP’开头的key
KEYS LEAPTCP*
#增量式迭代
SCAN cursor MATCH LEAP* COUNT 1
```

## 其他

* redis开启远程访问配置：
  1. 注释掉 bind 127.0.0.1 
  2.  protected-mode yes  改为 protected-mode no

---

[好文章]
  [Redis启动过程](https://www.hoohack.me/2018/05/26/read-redis-src-how-server-start)
  [Redis4.0新特性(三)-Lazy Free](https://www.jianshu.com/p/e927e99e650d)
  [Redis的内部运作机制—贼详细](https://www.cnblogs.com/wujuntian/p/9218961.html)
  [Redis持久化RDB和AOF](https://my.oschina.net/davehe/blog/174662)

[^1]:Redis是一个使用ANSI C语言编写的开源key-value数据库，但其字符串类型并没有直接使用C语言传统的字符串表示，而是自己构建了一种名为SDS的抽象类型，其相较于C语言字符串的区别可以参见[这篇](https://www.cnblogs.com/jaycekon/p/6227442.html)
[^2]:默认配置为5，如果增加，会提高LRU或TTL的精准度。redis作者测试的结果是当这个配置为10时已经非常接近全量 LRU 的精准度了，并且增加 maxmemory-samples 会导致在主动清理时消耗更多的CPU时间
[^3]:很多人提到Redis时都会讲这是一个单线程的内存数据库，其实不然——虽然Redis把处理网络收发和执行命令这些操作都放在了主工作线程，但是除此之外还有许多bio后台线程也在兢兢业业的工作着，比如用来处理关闭文件和刷盘这些比较重的IO操作。而Redis4.0则又增加了一个 lazyfree 线程——确切的说，Redis是一个主进程是单线程工作的内存数据库
[^4]:统一、中心化的键删除策略，保证主从服务器的数据一致性