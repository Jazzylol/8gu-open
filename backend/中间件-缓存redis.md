##### redis 数据结构有几种？

string，hash，set ，zset，list

其中set是使用hash表实现，不允许重复数据

zset是使用跳表来实现的，也不允许重复数据

##### redis 持久化有哪几种类型？

redis的持久化分为aof，rdb两种。

rdb就是把数据持久化到磁盘的过程，触发rdb的方式有两种，一种手动，一种自动。

save：阻塞当前进程，一直到rdb完成

bgsave ：fork出一个子进程，由子进程来负责 rdb过程。

具体流程如下：

- redis客户端执行bgsave命令或者自动触发bgsave命令；
- 主进程判断当前是否已经存在正在执行的子进程，如果存在，那么主进程直接返回；
- 如果不存在正在执行的子进程，那么就fork一个新的子进程进行持久化数据，fork过程是阻塞的，fork操作完成后主进程即可执行其他操作；
- 子进程先将数据写入到临时的rdb文件中，待快照数据写入完成后再原子替换旧的rdb文件；
- 同时发送信号给主进程，通知主进程rdb持久化完成，主进程更新相关的统计信息（info Persitence下的rdb_*相关选项）。

###### 自动触发rdb条件

1. 配置文件里配置的 save  m n  m秒次有n次修改。
2. 主从复制时，从节点要从主节点进行全量复制时也会触发bgsave操作，生成当时的快照发送到从节点；
3. 默认情况下执行shutdown命令时，如果没有开启aof持久化，那么也会触发bgsave操作；



##### RDB的过程中如何保持redis的可用性，以及数据一致性

rdb过程中只有fork的时候会阻塞，子进程fork完毕以后，主进程可以继续接受请求处理，所以不会导致redis无法处理请求。另外一方面，在rdb的过程中，子进程采用copy on write的方式，被修改的key 会同时再保存一份副本，子进程也会读取这个副本文件 写到rdb文件里。

##### 写RDB写一半的时候 崩溃了如何处理

就是写失败，以上一次rdb文件为准，所以可能存在丢失数据

##### 可以高频的RDB吗

具体要看情况，高频RDB存在2个问题

1. fork进程需要消耗较大的资源，还会阻塞主进程，过于频繁，redis的对外能力要极大的下降
2. rdb是写磁盘，如果数据过大，上一次rdb还没结束，立刻又写rdb，磁盘IO会扛不住，恶性循环，崩溃。

所以结论是不可以 太高频



##### RDB优点缺点

优点

1. RDB采用压缩算法，磁盘空间要远小于内存空间，适用于主从 备份等多种场景
2. redis从rdb恢复的速度要快于aof

缺点

1. 实时性不够
2. 执行成本比价高  fork进程消耗资源大
3. rdb文件是二进制 无可读性，还有就是不同版本兼容性问题

##### AOF持久化

aof持久化其实就是把redis执行的命令 按照顺序记录下来。

redis采用先写内存再记录日志的方式 写aof，主要原因就是为了保持高性能，不阻塞当前写操作，另外个原因就是aof是不校验命令正确的，如果先写日志再操作内存的话，命令有问题还要混滚日志。当时这种也存在问题，如果写内存后，直接宕机，那就会丢数据

##### AOF是如何实现的？

AOF主要分步骤

1. 命令追加，redis执行完命令以后，追加命令到缓冲区
2. 命令写入，redis有策略把缓冲区中的数据 刷到磁盘中，主要有三种

always：同步写回：每个写命令执行完，立马同步地将日志写回磁盘；

Everysec  每秒写回：每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘

No：操作系统控制的写回：每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。

![](https://pdai.tech/images/db/redis/redis-x-aof-4.jpg)



##### AOF文件过大怎么处理

redis有配置选项，一个是超过固定大小直接重写，或者超过上次重写的aof文件大小 比例 触发重写。AOF重写

##### AOF重写原理

AOF会合并原有的多条对同一个key的操作，达到缩小文件的目的。AOF是fork出进程来重写的，重写过程中发生数据改变，写操作会同时向2个进程的aof缓冲区 append 命令，保持双方数据同步。

- 主线程fork出子进程重写aof日志
- 子进程重写日志完成后，主线程追加aof日志缓冲
- 替换日志文件

##### 混合模式是什么

混合模式是redis 4.0以后提供一个持久化方式，固定频率使用rdb 快照保存数据，中间时间使用aof，优点是aof文件小，rdb压力也不大。

重启redis后加载数据的方式是，先加载aof文件，再加载rdb文件，因为aof频率更高，最多损失1s数据。

![](https://pdai.tech/images/db/redis/redis-x-aof-5.png)

##### Redis 事务是如何实现的

redis事务就是一次性的执行一系列的命令，要么成功，要么失败。redis不提供事务的回滚。

- MULTI ：开启事务，redis会将后续的命令逐个放入队列中，然后使用EXEC命令来原子化执行这个命令系列。
- EXEC：执行事务中的所有操作命令。
- DISCARD：取消事务，放弃执行事务块中的所有命令。
- WATCH：监视一个或多个key,如果事务在执行前，这个key(或多个key)被其他命令修改，则事务被中断，不会执行事务中的任何命令。
- UNWATCH：取消WATCH对所有key的监视。

redis如果事务开始以后，命令输入错误，那么数据不会变更，但是如果命令正确，数据类型错误，那么会跳过错误命令继续执行。

redis通过watch监视key，如果事务执行过程中发现 watch的key 发生变化，那么事务取消。

###### watch是如何实现监视事务的key的

Redis使用WATCH命令来决定事务是继续执行还是回滚，那就需要在MULTI之前使用WATCH来监控某些键值对，然后使用MULTI命令来开启事务，执行对数据结构操作的各种命令，此时这些命令入队列。

当使用EXEC执行事务时，首先会比对WATCH所监控的键值对，如果没发生改变，它会执行事务队列中的命令，提交事务；如果发生变化，将不会执行事务中的任何命令，同时事务回滚。当然无论是否回滚，Redis都会取消执行事务前的WATCH命令。

##### redis事务为何不可以回滚

这个主要是因为 redis是单线程的，不存在并发问题，所以redis认为事务执行失败，基本都是命令错误，但是命令错误又是程序控制的，所以这些错误理论上是不应该出现的，所以redis的事务从redis角度不存在需要回滚的问题，另外个就是 不支持，可以让redis保持简单，高效。

##### 什么是Redis主从复制

redis主从复制就是指从节点把主节点数据复制过去，并且提供备份，读取的作用。好处是：

- 数据冗余，备份
- 故障恢复：主节点出问题 可以从从节点恢复
- 负载均衡，从节点可以提供读的能力
- 提高了 redis的高可用的能力，也是 集群，哨兵的基础

主从之间采用的是读写分离的模式：	

读操作：主从都可以接受

写操作：主数据库接受 执行，然后同步写操作给从库

2.8 以前只存在全量复制，2.8版本以后存在增量复制

##### 全量复制的原理

在本redis上执行 

```bash
replicaof 172.16.19.3 6379
```

 则本redis 成为 别的节点的备库，过程分三个阶段

1. 主从建立连接，协商同步进度
2. 主库把所有数据同步给从库，主要依赖rdb文件，主库生成最新rdb文件，发给从库，从库清空数据库，然后从rdb中恢复数据
3. 主库再把rdb同步过程中的写命令，全部同步给从库
4. 进入正常同步模式

##### 增量复制原理

增量复制主要为了解决，网络突然中断导致的短期断联问题，redis2.8版本以后增加

增量复制主要就是 主库中存在一个发送给从库写命令的环形缓冲区，以及一个指针标识从库写到哪里了，如果断联的时间过长，指针被覆盖掉了，那么就是全量复制，还有一种情况就是 从库复制的时候也会发命令过来说从 多少offset开始复制，如果过大，主库也会全量复制，如果很小则会增量复制。

##### 全量复制为什么不使用AOF

主要原因是 rdb文件比较小，网络传输方便，aof文件比较大。而且aof要频繁些磁盘，性能损耗还是要比rdb来的多。

##### 为什么会有 主-从-从的模式

主要是为了解决 从库全部从主库复制 rdb的压力问题，从库从从库复制 rdb压力会小很多。

##### 主从延迟问题怎么解决

没法解决，客观存在，只能加配置加机器，加带宽，监控延迟等等，对于数据敏感的业务 还是从主库读。

##### 数据过期删除策略是怎么样的

惰性删除：不主动删除，查询key的时候判断是否过期，再删除

定期删除：后台线程，100ms一次，采样删除，不超过25ms，每次。

##### 哨兵机制是什么？

哨兵就是保障主从模式故障切换的一个机制。故障转移 2.8版本引入。

![](https://pdai.tech/images/db/redis/db-redis-sen-1.png)

哨兵节点的功能：

1. 监控，哨兵节点会检查 主从节点是否正常
2. 故障转移，主节点不能工作，那么他会选择一个可以工作的从节点成为主节点，原有的主节点会下线
3. 配置中心：客户端初始化的时候，链接哨兵集群可以获取当前redis节点所有信息
4. 通知：哨兵节点会把故障转移信息发给客户端

##### 哨兵集群是怎么组建的？

哨兵节点通过 redis 的 pub sub通知订阅的方式，相互发现。所有哨兵节点 链接主节点，有个 __sentinel__ 频道，发送hello，互相知道了对方的地址，从而建立连接，相互通信，对于主库的状态进行协商。

哨兵节点通过向主库发送info信息获取所有从库，进而链接从库，然后监控整个redis集群。

##### 主库如何判定下线

主观下线：每个哨兵都可以判断主库是否下线

客观下线：有哨兵发现主库下线的时候，他会发消息问其他的哨兵，超过配置值，则认为主库客观下线。

##### 哨兵集群如何选举新的主库

在主从故障转移之前，需要哨兵集群先进行选举，再进行故障转移。哨兵集群选举leader有2个条件

1. 某个哨兵拿到超过一半的票数（非故障之前的 机器数量，也就是要包括故障的哨兵数量）
2. 拿到票数的数量还要大于 配置文件的值。

##### 哪个从库才能担当新的主库

- 过滤掉不健康的（下线或断线），没有回复过哨兵ping响应的从节点
- 选择`salve-priority`从节点优先级最高（redis.conf）的
- 选择复制偏移量最大，只复制最完整的从节点

主从故障恢复流程如下：

![](https://pdai.tech/images/db/redis/db-redis-sen-4.png)

- 将slave-1脱离原从节点（PS: 5.0 中应该是`replicaof no one`)，升级主节点，
- 将从节点slave-2指向新的主节点
- 通知客户端主节点已更换
- 将原主节点（oldMaster）变成从节点，指向新的主节点



##### 为什么要构建主库的分布式集群

因为主库读写能力有限，从库又存在天然的劣势，比如数据延迟等，所以横向扩展必须要的。

##### redis集群

redis 3.6 以后增加了集群的方式，使用slot 哈希槽的概念，整个集群分了2^14 个哈希槽。每个节点负责一部分哈希槽。

比如集群中存在三个节点，则可能存在的一种分配如下：

- 节点A包含0到5500号哈希槽；
- 节点B包含5501到11000号哈希槽；
- 节点C包含11001 到 16384号哈希槽。

redis也提供了一种key tag的方式来保障2个key 可以在一台机器上。集群通过 gossip 算法可以快速让节点被全部机器知道，从而建立连接。

##### 客户端查询集群 key的过程？

redis cluster 采用了去中心化的架构，集群每个节点负责一部分的key，具体key在那部分主要靠 hash算法。过程如下

检查当前key是否存在当前NODE？ 

- 通过crc16（key）/16384计算出slot
- 查询负责该slot负责的节点，得到节点指针
- 该指针与自身节点比较

- 若slot不是由自身负责，则返回MOVED重定向
- 若slot由自身负责，且key在slot中，则返回该key对应结果
- 若key不存在此slot中，检查该slot是否正在迁出（MIGRATING）？
- 若key正在迁出，返回ASK错误重定向客户端到迁移的目的服务器上
- 若Slot未迁出，检查Slot是否导入中？
- 若Slot导入中且有ASKING标记，则直接操作
- 否则返回MOVED重定向

简单解释就是，根据算法计算出slot是否自身负责，如果不是则返回客户端 move，如果是 又分三种情况，

1. 有数据 直接返回
2. 数据正在转移到别的机器（有新机器主动加入集群），那告诉客户端 ask 错误，让客户端自己去新机器上找数据
3. 数据正在从别的机器转移到本机（有旧机器下线），直接返回查询数据

##### MOVE命令和 ASK命令区别？

MOVE主要是告诉客户端数据不在我这个节点上，具体在哪个节点上。具体由客户端重新查询

ASK命令是告诉客户端，数据正在迁移，数据在指定server上，客户端需要重新查询。

##### Gossip协议的使用

Redis 集群是去中心化的，彼此之间状态同步靠 gossip 协议通信，集群的消息有以下几种类型：

- `Meet` 通过「cluster meet ip port」命令，已有集群的节点会向新的节点发送邀请，加入现有集群。
- `Ping` 节点每秒会向集群中其他节点发送 ping 消息，消息中带有自己已知的两个节点的地址、槽、状态信息、最后一次通信时间等。
- `Pong` 节点收到 ping 消息后会回复 pong 消息，消息中同样带有自己已知的两个节点信息。
- `Fail` 节点 ping 不通某节点后，会向集群所有节点广播该节点挂掉的消息。其他节点收到消息后标记已下线。

##### 集群下 master节点挂了如何处理

1. 从节点成为 master，然后向集群中所有机器发送通知
2. 只有其他主节点响应通知
3. 超过半数响应认可合法，则从节点成为master
4. 新的master发送通知告诉所有集群机器，他成为master。

##### redis 集群缩扩容

流程大概一致的，比如扩容，新增机器后，加入集群，然后通过命令，可以把指定槽数的数据 迁移到该机器上。缩容则是要先把槽上机器数据 迁移到别的机器，然后下线机器。

##### 为什么redis集群的哈希槽是2^14不是2^16?

1. 心跳包里放了所有的机器以及槽的信息，16k的槽信息压缩后大概是2k左右大小的数据，这个大小作者认为在机器之间高频的传播是比较合适的。
2. 作者认为集群大概不会超过1000台机器，2^16次方个槽位算下来大概压缩后心跳包数据8k，这个数据量太大了。

##### redis集群的其他方案

Twemproxy，主要是在redis集群基础上包了一层，客户端。

![](https://pdai.tech/images/db/redis/db-redis-cluster-2.png)

优点：

- 开发简单，对应用几乎透明
- 历史悠久，方案成熟

缺点：

- 代理影响性能
- LVS 和 Twemproxy 会有节点性能瓶颈
- Redis 扩容非常麻烦
- Twitter 内部已放弃使用该方案，新使用的架构未开源

###### Codis

Codis 是由豌豆荚开源的产品，涉及组件众多，其中 ZooKeeper 存放路由表和代理节点元数据、分发 Codis-Config 的命令，Codis-Config 是集成管理工具，有 Web 界面供使用；Codis-Proxy 是一个兼容 Redis 协议的无状态代理；Codis-Redis 基于 Redis 2.8 版本二次开发，加入 slot 支持，方便迁移数据。

![](https://pdai.tech/images/db/redis/db-redis-cluster-3.png)

优点：

- 开发简单，对应用几乎透明
- 性能比 Twemproxy 好
- 有图形化界面，扩容容易，运维方便

缺点：

- 代理依旧影响性能
- 组件过多，需要很多机器资源
- 修改了 Redis 代码，导致和官方无法同步，新特性跟进缓慢
- 开发团队准备主推基于 Redis 改造的 reborndb

##### Redis缓存问题

- 缓存穿透
- 缓存穿击
- 缓存雪崩
- 缓存污染（或者满了）
- 缓存和数据库一致性

##### 缓存穿透

缓存穿透指的是，缓存数据库都不存在数据，用户用不存在key疯狂攻击。

解决办法：

1. 增加权限校验，增加id<0校验
2. 不存在的key，可以在缓存中设置 key value=null
3. 布隆过滤器。bloomfilter就类似于一个hash set，用于快速判某个元素是否存在于集合中，其典型的应用场景就是快速判断一个key是否存在于某容器，不存在就直接返回。布隆过滤器的关键就在于hash算法和容器大小，

##### 缓存击穿

缓存击穿指的是，数据库存在数据，缓存不存在，在一瞬间大量请求打到db导致服务宕机。

解决方案：

1. 热点key提前加热，永不过期
2. 接口限流，熔断，降级，快速失败返回
3. 加互斥锁
4. 增加对象正在查询db的 flag在缓存中

##### 缓存雪崩

缓存雪崩指的是缓存中大量数据同时过期，导致同时去查数据库导致宕机，和缓存击穿的却别在于，击穿是大量查一条数据，雪崩时查多条数据。

解决方案

1. 过期时间，随机。
2. 热点数据 提前加热，永不过期

##### 缓存污染

缓存污染指的是，数据长期不淘汰，然后缓存满了，只能靠淘汰策略了。

Redis共支持八种淘汰策略，分别是noeviction、volatile-random、volatile-ttl、volatile-lru、volatile-lfu、allkeys-lru、allkeys-random 和 allkeys-lfu 策略。

**怎么理解呢**？主要看分三类看：

- 不淘汰 
  - noeviction （v4.0后默认的）  再写报错
- 对设置了过期时间的数据中进行淘汰 
  - 随机：volatile-random  随机删除
  - ttl：volatile-ttl   根据ttl字段，约接近过期的 越优先删除 
  - lru：volatile-lru  lru算法删除 最近最少使用
  - lfu：volatile-lfu  根据访问次数，访问次数少的优先删除，最多记录255次访问，超过了退化成LRU算法
- 全部数据进行淘汰 
  - 随机：allkeys-random  全部数据随机 
  - lru：allkeys-lru  全部数据LRU
  - lfu：allkeys-lfu 全部数据LFU

#### 数据库和缓存的一致性问题

结论：读取数据没问题，读数据库 然后写缓存，但是更新的时候，无论先写数据库，还是先写缓存都会出现不一致的情况。

1. 先写数据库，准备写缓存的时候，服务挂了，那缓存的数据是错误的。
2. 先写缓存，还未写数据库的时候，其他线程读了数据库再更新到缓存中，导致缓存覆盖，还是不一致的。

常用的方式是

1. 读数据，先读缓存，读不到读数据库，然后塞到缓存里
2. 更新的时候，先更新数据库，然后直接删除缓存（不是更新缓存，因为更新的话，如果有并发线程更新会导致缓存脏数据问题）

但是这种方式也会导致问题，比如一个线程正在更新，一个线程读到了未更新的数据，然后更新线程 删除了缓存，读的线程又更新了缓存，这样缓存里的数据仍旧是错误的，但是这个概率比较小，一方面更新速度要慢很多，查询速度快。而且更新还要锁表。结论就是

常规方法都会有问题，要不就是2pc 或者paxios 分布式事务保持数据一致性，要不就是 用这种方式 小概率事件。

其他解决方式

##### 队列+重试

![](https://pdai.tech/images/db/redis/db-redis-cache-4.png)

通过消息队列的重试功能保障  key一定可以删除成功

缺点：业务代码侵入厉害

##### binlog 订阅

![](https://pdai.tech/images/db/redis/db-redis-cache-5.png)

优点：业务代码分离，缓存代码分离

缺点：复杂性高



##### Redis为什么快？

1. 纯内存操作
2. 数据结构设计的好，比如使用跳表不是用 b+树，可以把数据全放到内存中，不考虑磁盘
3. 单线程模型，避免了大量的上线文切换。但Redis的其他功能，比如持久化、异步删除、集群数据同步等等，实际是由额外的线程执行的。
4. IO模型，多路复用模型，可以处理并发的连接。非阻塞IO 内部实现采用epoll，采用了epoll+自己实现的简单的事件框架，比较高效。





