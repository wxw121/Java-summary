## Redis总结

#### **1.Redis 的基本操作**

- **set** key value  信息添加

- **get** key  信息查询

- clear  清楚屏幕信息

- quit/exit  退出客户端

- help  **命令名称**

  help  @**组名**    获取命令帮助文档，获取组中所有命令信息名称

####  **2.Redis基本数据存储类型**

##### 2.1String类型

- 存储的数据：单个数据，最简单的数据存储类型，也是最常用的数据存储类型
- 存储数据的格式：一个存储空间保存一个数据
- 存储内容：通常使用字符串，如果字符串以整数的形式展示，可以作为数字操作使用

```String类型
set key value   添加/修改数据
get key          获取数据
del key          删除数据
mset key1 value1 key2 value2 …   添加/修改多个数据
mget key1 key2 …                  获取多个数据
strlen key       获取数据字符个数（字符串长度）
append key value  追加信息到原始信息后部（如果原始信息存在就追加，否则新建）

扩展操作：
设置数值数据增加指定范围的值
incr key
incrby key increment
incrbyfloat key increment

设置数值数据减少指定范围的值
decr key
decrby key increment

设置数据具有指定的生命周期
setex key seconds value
psetex key milliseconds value
```

业务场景：①redis用于控制数据库表主键id（redis所有操作都是原子性），为数据库表主键提供生成策略，保障数据库表的主键唯一性（此方案适用于所有数据库，且支持数据库集群）。②redis 控制数据的生命周期，通过数据是否失效控制业务行为，适用于所有具有时效性限定控制的操作。③主页高频访问信息显示控制，例如新浪微博大V主页显示粉丝数与微博数量（redis应用于各种结构型和非结构型高热度数据访问加速）。

##### 2.2Hash类型

- 新的存储需求：对一系列存储的数据进行编组，方便管理，典型应用存储对象信息

-  需要的存储结构：一个存储空间保存多个键值对数据

-  hash类型：底层使用哈希表结构实现数据存储

  *hash存储结构优化：如果field数量较少，存储结构优化为类数组结构，如果field数量较多，存储结构使用HashMap结构。*

  *hash类型数据操作注意事项：①hash类型下的value只能存储字符串，不允许存储其他数据类型，不存在嵌套现象。如果数据未获取到， 对应的值为（nil）。②hgetall 操作可以获取全部属性，如果内部field过多，遍历整体数据效率就很会低，有可能成为数据访问 瓶颈。*

```hash
hset key field value   添加/修改数据
hget key field
hgetall key              获取数据
hdel key field1 [field2] 删除数据
hmset key field1 value1 field2 value2 …    添加/修改多个数据
hmget key field1 field2 …                   获取多个数据
hlen key                                 获取哈希表中字段的数量
hexists key field                     获取哈希表中是否存在指定的字段

扩展操作：
获取哈希表中所有的字段名或字段值
hkeys key
hvals key

设置指定字段的数值数据增加指定范围的值
hincrby key field increment
hincrbyfloat key field increment

为哈希表中不存在的的字段赋值
hsetnx key field value
```

业务场景：①string存储对象（json）与hash存储对象（电商网站购物车设计与实现）。②双11活动日，销售手机充值卡的商家对移动、联通、电信的30元、50元、100元商品推出抢购活动，每种商 品抢购上限1000张。

##### 2.3list类型

-  数据存储需求：存储多个数据，并对数据进入存储空间的顺序进行区分
-  需要的存储结构：一个存储空间保存多个数据，且通过数据可以体现进入顺序
-  list类型：保存多个数据，底层使用双向链表存储结构实现

```list
添加/修改数据
lpush key value1 [value2] ……
rpush key value1 [value2] ……

获取数据
lrange key start stop
lindex key index
llen key

获取并移除数据
lpop key
rpop key

扩展操作
规定时间内获取并移除数据
blpop key1 [key2] timeout
brpop key1 [key2] timeout
brpoplpush source destination timeout

移除指定数据
lrem key count value
```

业务场景：①微信朋友圈点赞，要求按照点赞顺序显示点赞好友信息 如果取消点赞，移除对应好友信息。（ redis 应用于具有操作先后顺序的数据控制）。②twitter、新浪微博、腾讯微博中个人用户的关注列表需要按照用户的关注顺序进行展示，粉丝列表需要将最 近关注的粉丝列在前面。③ redis 应用于最新消息展示。

##### 2.4set类型

- 新的存储需求：存储大量的数据，在查询方面提供更高的效率
- 需要的存储结构：能够保存大量的数据，高效的内部存储机制，便于查询
- set类型：与hash存储结构完全相同，仅存储键，不存储值（nil），并且值是不允许重复的

```set
sadd key member1 [member2]     添加数据
smembers key                   获取全部数据
srem key member1 [member2]     删除数据
scard key                      获取集合数据总量
sismember key member           判断集合中是否包含指定数据

扩展操作
随机获取集合中指定数量的数据
srandmember key [count]

随机获取集合中的某个数据并将该数据移出集合
spop key [count]

求两个集合的交、并、差集
sinter key1 [key2] 
sunion key1 [key2] 
sdiff key1 [key2]

求两个集合的交、并、差集并存储到指定集合中
sinterstore destination key1 [key2] 
sunionstore destination key1 [key2] 
sdiffstore destination key1 [key2]

将指定数据从原始集合中移动到目标集合中
smove source destination member
```

应用场景：①redis 应用于随机推荐类信息检索，例如热点歌单推荐，热点新闻推荐，热卖旅游线路，应用APP推荐， 大V推荐等。②公司对旗下新的网站做推广，统计网站的PV（访问量）,UV（独立访客）,IP（独立IP）。 **PV**：网站被访问次数，可通过刷新页面提高访问量 **UV**：网站被不同用户访问的次数，可通过cookie统计访问量，相同用户切换IP地址，UV不变 **IP**：网站被不同IP地址访问的总次数，可通过IP地址统计访问量，相同IP不同用户访问，IP不变。③redis 应用于基于黑名单与白名单设定的服务控制。

##### 2.5sorted_set类型

-  新的存储需求：数据排序有利于数据的有效展示，需要提供一种可以根据自身特征进行排序的方式
-  需要的存储结构：新的存储模型，可以保存可排序的数据
-  sorted_set类型：在set的存储结构基础上添加可排序字段

```sorted_set
zadd key score1 member1 [score2 member2]    添加数据
zrange key start stop [WITHSCORES]
zrevrange key start stop [WITHSCORES]       获取全部数据
zrem key member [member ...]                删除数据
zrangebyscore key min max [WITHSCORES] [LIMIT]
zrevrangebyscore key max min [WITHSCORES]   按条件获取数据
zremrangebyrank key start stop
zremrangebyscore key min max                条件删除数据
zcard key
zcount key min max                          获取集合数据总量
zinterstore destination numkeys key [key ...]
zunionstore destination numkeys key [key ...]    集合交、并操作

扩展操作
获取数据对应的索引（排名）
zrank key member
zrevrank key member

score值获取与修改
zscore key member
zincrby key increment member
```

业务场景：①redis 应用于计数器组合排序功能对应的排名。②redis 应用于定时任务执行顺序管理或任务过期管理。③任务/消息权重设定应用当任务或者消息待处理，形成了任务队列或消息队列时，对于高优先级的任务要保障对其优先处理。

#### 3.通用指令

##### 3.1key通用指令

```key
del key      删除指定key
exists key   获取key是否存在
type key     获取key的类型

扩展操作（时效性控制）
为指定key设置有效期
expire key seconds
pexpire key milliseconds
expireat key timestamp
pexpireat key milliseconds-timestamp

获取key的有效时间
ttl key
pttl key

切换key从时效性转换为永久性
persist key

查询模式
keys pattern
查询模式规则：* 匹配任意数量的任意符号 ? 配合一个任意符号 [] 匹配一个指定符号

其他操作
为key改名
rename key newkey
renamenx key newkey

对所有key排序
sort

其他key通用操作
help @generic
```

##### 3.2数据库通用指令

为了防止redis使用过程中，数据量急剧增加产生重复和冲突，redis为每个服务创建16个数据库，编号从0到15且数据库之间的数据相互独立。

```db
select index  切换数据库
quit
ping
echo message   其他操作
move key db    数据移动
dbsize
flushdb
flushall       数据清除
```

#### 4.jedis

##### jedis操作redis数据库步骤：

1.导入依赖

```jedis
<dependency>
 <groupId>redis.clients</groupId>
 <artifactId>jedis</artifactId>
 <version>2.9.0</version>
</dependency>
```

2.客户端连接jedis

```
1.连接redis
Jedis jedis = new Jedis("localhost", 6379);

2.操作redis
jedis.set("name", "itheima");
jedis.get("name");

3.关闭redis连接
jedis.close();
```

##### 基于连接池获取连接：

JedisPool：Jedis提供的连接池技术     poolConfig:连接池配置对象      host:redis服务地址     port:redis服务端口号

```JedisPool
public JedisPool(GenericObjectPoolConfig poolConfig, String host, int port) {
 this(poolConfig, host, port, 2000, (String)null, 0, (String)null);
}
```

封装连接参数：jedis.properties

```jedis.properties
jedis.host=localhost
jedis.port=6379
jedis.maxTotal=30
jedis.maxIdle=10
```

加载配置信息：静态代码块初始化资源

```static
static{
 //读取配置文件 获得参数值
 ResourceBundle rb = ResourceBundle.getBundle("jedis");
 host = rb.getString("jedis.host");
 port = Integer.parseInt(rb.getString("jedis.port"));
 maxTotal = Integer.parseInt(rb.getString("jedis.maxTotal"));
 maxIdle = Integer.parseInt(rb.getString("jedis.maxIdle"));
 poolConfig = new JedisPoolConfig();
 poolConfig.setMaxTotal(maxTotal);
 poolConfig.setMaxIdle(maxIdle);
 jedisPool = new JedisPool(poolConfig,host,port);
}
```

获取连接：对外访问接口，提供jedis连接对象，连接从连接池获取

```getJedis
public static Jedis getJedis(){
 Jedis jedis = jedisPool.getResource();
 return jedis;
}
```

#### 5.redis持久化

持久化：利用永久性存储介质将数据进行保存，在特定的时间将保存的数据进行恢复的工作机制称为持久化。

redis持久化技术有RDB持久化和AOF持久化

##### 5.1RDB持久化

```rdb-save
RDB启动方式——save指令
save   手动执行一次保存操作

save指令相关配置
dbfilename dump.rdb
说明：设置本地数据库文件名，默认值为 dump.rdb
经验：通常设置为dump-端口号.rdb
dir
说明：设置存储.rdb文件的路径
经验：通常设置成存储空间较大的目录中，目录名称data
rdbcompression yes
说明：设置存储至本地数据库时是否压缩数据，默认为 yes，采用 LZF 压缩
经验：通常默认为开启状态，如果设置为no，可以节省 CPU 运行时间，但会使存储的文件变大（巨大）
rdbchecksum yes
说明：设置是否进行RDB文件格式校验，该校验过程在写文件和读文件过程均进行
经验：通常默认为开启状态，如果设置为no，可以节约读写性过程约10%时间消耗，但是存储一定的数据损坏风险
```

**注意：save指令的执行会阻塞当前Redis服务器，直到当前RDB过程完成为止，有可能会造成长时间阻塞，线上环境不建议使用。**（单线程任务执行序列）



```rdb-bgsave
RDB启动方式--bgsave指令
bgsave   手动启动后台保存操作，但不是立即执行

bgsave指令的相关配置
在save的基础上增加
stop-writes-on-bgsave-error yes
说明：后台存储过程中如果出现错误现象，是否停止保存操作
经验：通常默认为开启状态
```

bgsave指令工作原理

![image-20210714121515564](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210714121515564.png)

注意： bgsave命令是针对save阻塞问题做的优化。Redis内部所有涉及到RDB操作都采用bgsave的方式。



```配置自动启动RDB
RDB自动启动--save
save second changes    满足限定时间范围内key的变化数量达到指定数量即进行持久化
second：监控时间范围     changes：监控key的变化量
在conf文件中进行配置
```

```特殊启动方式
RDB特殊启动方式--全量复制
debug reload       服务器运行过程中重启
shutdown save      关闭服务器时指定保存数据
```

**注意：默认情况下执行shutdown命令时，自动执行 bgsave(如果没有开启AOF持久化功能)**

RDB优点：

-  RDB是一个紧凑压缩的二进制文件，存储效率较高
- RDB内部存储的是redis在某个时间点的数据快照，非常适合用于数据备份，全量复制等场景
- RDB恢复数据的速度要比AOF快很多
- 应用：服务器中每X小时执行bgsave备份，并将RDB文件拷贝到远程机器中，用于灾难恢复。

RDB缺点：

- RDB方式无论是执行指令还是利用配置，无法做到实时持久化，具有较大的可能性丢失数据
- bgsave指令每次运行要执行fork操作创建子进程，要牺牲掉一些性能
- Redis的众多版本中未进行RDB文件格式的版本统一，有可能出现各版本服务之间数据格式无法兼容现象

##### 5.2AOP持久化

AOF持久化：以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中命令 达到恢复数据的目的。与RDB相比可以简单描述为改记录数据为记录数据产生的过程。（相比RDB，解决了数据持久化的实时性）。

**AOF写数据的三种策略：always(每次，数据零误差，性能较低)，everysec(每秒，准确率较高，性能较高，推荐使用也是默认配置)，no(系统控制，由操作系统控制同步到AOF文件的周期，不可控)。**

```AOF
配置:	appendonly yes|no
作用:	是否开启AOF持久化功能，默认为不开启状态
配置AOF写数据的三种策略: appendfsync always|everysec|no

配置:	appendfilename filename
作用:	AOF持久化文件名，默认文件名未appendonly.aof，建议配置为appendonly-端口号.aof

配置:	dir
作用:	AOF持久化文件保存路径，与RDB持久化文件保持一致即可
```



AOF重写：随着命令不断写入AOF，文件会越来越大，为了解决这个问题，Redis引入了AOF重写机制压缩文件体积。（简单说就是将对同一个数据的若干个条命令执行结 果转化成最终结果数据对应的指令进行记录）

AOF重写规则：

-  进程内已超时的数据不再写入文件
- 忽略无效指令，重写时使用进程内数据直接生成，这样新的AOF文件只保留最终数据的写入命令
- 对同一数据的多条写命令合并为一条命令



```AOF重写
AOF重写方式
手动重写:	bgrewriteaof
自动重写:	auto-aof-rewrite-min-size size
           auto-aof-rewrite-percentage percentage
           
AOF自动重写方式
自动重写触发条件设置
auto-aof-rewrite-min-size size
auto-aof-rewrite-percentage percent

自动重写触发比对参数（ 运行指令info Persistence获取具体信息 ）
aof_current_size
aof_base_size

自动重写触发条件
aof_current_size > auto-aof-rewrite-min-size
aof_current_size - aof_base_size >= auto-aof-rewrite-percentage * aof_base_size
```

![image-20210714152431310](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210714152431310.png)

#### 6.redis事务

##### 6.1事务基本操作

```redis事务
事务的基本操作
multi    开启事务（开始位置）
exec     执行事务（结束位置）
discard  取消事务（开始与结束之间）
```

注意：①如果定义的事务中所包含的命令存在语法错误，整体事务中所有命令均不会执行。包括那些语法正确的命令。②已经执行完毕的命令对应的数据不会自动回滚，需要程序员自己在代码中实现回滚。

##### 6.2锁

```锁
watch key1 [key2……]    对 key 添加监视锁，在执行exec前如果key发生了变化，终止事务执行
unwatch                取消对所有 key 的监视
```

##### 6.3基于特定事务执行——分布式锁

例如超卖场景：天猫双11热卖过程中，对已经售罄的货物追加补货，且补货完成。客户购买热情高涨，3秒内将所有商品购买完毕。本次补货已经将库存全部清空，如何避免最后一件商品不被多人同时购买？

```分布式锁
解决方案
setnx lock-key value   使用 setnx 设置一个公共锁
利用setnx命令的返回值特征，有值则返回设置失败，无值则返回设置成功
对于返回设置成功的，拥有控制权，进行下一步的具体业务操作
对于返回设置失败的，不具有控制权，排队或等待
操作完毕通过del操作释放锁
```

特定事务2场景：依赖分布式锁的机制，某个用户操作时对应客户端宕机，且此时已经获取到锁。如何解决？

```
使用 expire 为锁key添加时间限定，到时不释放，放弃锁
expire lock-key second
pexpire lock-key milliseconds
```

#### 7.redis删除策略

##### 7.1数据删除策略

数据删除策略的目标：在内存占用与CPU占用之间寻找一种平衡，顾此失彼都会造成整体redis性能的下降，甚至引发服务器宕机或 内存泄露。

###### 1.定时删除

- 创建一个定时器，当key设置有过期时间，且过期时间到达时，由定时器任务立即执行对键的删除操作
- 优点：节约内存，到时就删除，快速释放掉不必要的内存占用
- 缺点：CPU压力很大，无论CPU此时负载量多高，均占用CPU，会影响redis服务器响应时间和指令吞吐量
- 总结：用处理器性能换取存储空间（拿时间换空间）

###### 2.惰性删除

- 数据到达过期时间，不做处理。等下次访问该数据时：1. 如果未过期，返回数据。2. 发现已过期，删除，返回不存在。
- 优点：节约CPU性能，发现必须删除的时候才删除
- 缺点：内存压力很大，出现长期占用内存的数据
- 总结：用存储空间换取处理器性能（拿空间换时间）

###### 3.定期删除（随机抽查，重点抽查）

定时删除和惰性删除的折中方案

- Redis启动服务器初始化时，读取配置server.hz的值，默认为10
- 每秒钟执行server.hz次serverCron()  -->  databasesCron()  -->  activeExpireCycle()
- activeExpireCycle()对每个expires[*]逐一进行检测，每次执行250ms/server.hz
- 对某个expires[*]检测时，随机挑选W个key检测：
  -  如果key超时，删除key
  - 如果一轮中删除的key的数量>W*25%，循环该过程
  - 如果一轮中删除的key的数量≤W*25%，检查下一个expires[*]，0-15循环
  - W取值=ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP属性值

-  参数current_db用于记录activeExpireCycle() 进入哪个expires[*] 执行
- 如果activeExpireCycle()执行时间到期，下次从current_db继续向下执行                               

##### 7.2逐出算法

```逐出算法
逐出算法的相关配置
maxmemory      最大可使用内存（生产环境一般设置为50%）
maxmemory-samples    每次选取待删除数据的个数（选取数据时并不会全库扫描，导致严重的性能消耗，降低读写性能。）
maxmemory-policy     删除策略（达到最大内存后的，对被挑选出来的数据进行删除的策略）
```

逐出算法：

- 检测易失数据（可能会过期的数据集server.db[i].expires ）
  - volatile-lru：挑选最近最少使用的数据淘汰
  - volatile-lfu：挑选最近使用次数最少的数据淘汰
  - volatile-ttl：挑选将要过期的数据淘汰
  - volatile-random：任意选择数据淘汰

-  检测全库数据（所有数据集server.db[i].dict ）
  - allkeys-lru：挑选最近最少使用的数据淘汰
  - allkeys-lfu：挑选最近使用次数最少的数据淘汰
  -  allkeys-random：任意选择数据淘汰

- 放弃数据驱逐
  - no-enviction（驱逐）：禁止驱逐数据（redis4.0中默认策略），会引发错误OOM（Out Of Memory）

#### 8.redis核心配置

服务端设定

```核心配置
daemonize yes|no   设置服务器以守护进程的方式运行
bind 127.0.0.1     绑定主机地址
port 6379          设置服务器端口号
databases 16       设置数据库数量

日志配置
loglevel debug|verbose|notice|warning    设置服务器以指定日志记录级别
logfile 端口号.log    日志记录文件名

多服务器快捷配置
include /path/server-端口号.conf   导入并加载指定配置文件信息，用于快速创建redis公共配置较多的redis实例配置文件，便于维护
```

注意：日志级别开发期设置为verbose即可，生产环境中配置为notice，简化日志输出量，降低写日志IO的频度。



客户端配置

```客户端配置
maxclients 0       设置同一时间最大客户端连接数，默认无限制。当客户端连接到达上限，Redis会关闭新的连接
timeout 300        客户端闲置等待最大时长，达到最大值后关闭连接。如需关闭该功能，设置为 0
```

#### 9.高级数据类型

##### 9.1Bitmaps

```Bitmaps
基础操作
getbit key offset        获取指定key对应偏移量上的bit值
setbit key offset value  设置指定key对应偏移量上的bit值，value只能是1或0

扩展操作
bitop op destKey key1 [key2...]   对指定key按位进行交、并、非、异或操作，并将结果保存到destKey中
bitcount key [start end]          统计指定key中1的数量
```

业务场景：应用于信息状态统计

电影网站

- 统计每天某一部电影是否被点播
- 统计每天有多少部电影被点播
- 统计每周/月/年有多少部电影被点播
- 统计年度哪部电影没有被点播

##### 9.2HyperLogLog

基数： 基数是数据集去重后元素个数

HyperLogLog 是用来做**基数**统计的，运用了LogLog的算法

eg：{1, 3, 5, 7, 5, 7, 8}    基数集： {1, 3, 5 ,7, 8}    基数：5

```Hyperloglog
pfadd key element [element ...]             添加数据
pfcount key [key ...]                       统计数据
pfmerge destkey sourcekey [sourcekey...]    合并数据
```

业务场景：应用于独立信息的统计

注意：

-  用于进行基数统计，不是集合，不保存数据，只记录数量而不是具体数据
- 核心是基数估算算法，最终数值存在一定误差
- 误差范围：基数估计的结果是一个带有 0.81% 标准错误的近似值
- 消耗空间极小，每个hyperloglog key占用了12K的内存用于标记基数
- pfadd命令不是一次性分配12K内存使用，会随着基数的增加内存逐渐增大
- Pfmerge命令合并后占用的存储空间为12K，无论合并之前数据量多少

##### 9.3GEO

```GEO
基本操作
geoadd key longitude latitude member [longitude latitude member ...]    添加坐标点
geopos key member [member ...]                                          获取坐标点
geodist key member1 member2 [unit]                                      计算坐标点距离

georadius key longitude latitude radius m|km|ft|mi [withcoord] [withdist] [withhash] [count count]  添加坐标点
georadiusbymember key member radius m|km|ft|mi [withcoord] [withdist] [withhash] [count count]      获取坐标点
geohash key member [member ...]     计算经纬度
```

#### 10.主从复制

目的：为了避免单点Redis服务器故障，准备多台服务器，互相连通。将数据复制多个副本保存在不同的服务器上，连接在一起，并保证数据是同步的。即使有其中一台服务器宕机，其他服务器依然可以继续 提供服务，**实现Redis的高可用**，同时实现数据冗余备份。



主从复制：主从复制即将master中的数据即时、有效的复制到slave中。

主从复制的作用：

- 读写分离：master写、slave读，提高服务器的读写负载能力。
- 负载均衡：基于主从结构，配合读写分离，由slave分担master负载，并根据需求的变化，改变slave的数量，通过多个从节点分担数据读取负载，大大提高Redis服务器并发量与数据吞吐量。
- 故障恢复：当master出现问题时，由slave提供服务，实现快速的故障恢复。
- 数据冗余：实现数据热备份，是持久化之外的一种数据冗余方式。
- 高可用基石：基于主从复制，构建哨兵模式与集群，实现Redis的高可用方案。

主从复制工作过程：1.建立连接阶段。2.数据同步阶段。3.命令传播阶段。



##### 建立连接阶段工作流程：

1. 设置master的地址和端口，保存master信息。
2. 建立socket连接。
3. 发送ping命令（定时器任务）。
4. 身份验证。
5. 发送slave端口信息。

最终状态：slave：保存master的地址与端口    master：保存slave的端口     总体：之间创建了连接的socket

![](C:\Users\wuxiaowen\Desktop\主从复制--建立连接阶段工作流程.png)

```建立连接阶段
主从连接（slave连接master）
slaveof <masterip> <masterport>                  方式一：客户端发送命令
redis-server -slaveof <masterip> <masterport>    方式二：启动服务器参数
slaveof <masterip> <masterport>                  方式三：服务器配置
主从连接断开
slaveof no one
注意：slave断开连接后，不会删除已有数据，只是不再接受master发送的数据

授权访问
requirepass <password>      master客户端发送命令设置密码
config set requirepass <password>
config get requirepass      master配置文件设置密码
auth <password>             slave客户端发送命令设置密码
masterauth <password>       slave配置文件设置密码
redis-server –a <password>  slave启动服务器设置密码
```



##### 数据同步阶段工作流程：

- 在slave初次连接master后，复制master中的所有数据到slave
-  将slave的数据库状态更新成master当前的数据库状态

步骤：

1. 请求同步数据
2. 创建RDB同步数据
3. 恢复RDB同步数据
4. 请求部分同步数据
5. 恢复部分同步数据

最终状态：slave：具有master端全部数据，包含RDB过程接收的数据     master：保存slave当前数据同步的位置      总体：之间完成了数据克隆。

![](C:\Users\wuxiaowen\Desktop\主从复制——数据同步阶段工作流程.png)

数据同步阶段master说明：

-  如果master数据量巨大，数据同步阶段应避开流量高峰期，避免造成master阻塞，影响业务正常执行。

-  复制缓冲区大小设定不合理，会导致数据溢出。如进行全量复制周期太长，进行部分复制时发现数据已经存在丢失的情况，必须进行第二次全量复制，致使slave陷入死循环状态。     repl-backlog-size 1mb
- master单机内存占用主机内存的比例不应过大，建议使用50%-70%的内存，留下30%-50%的内存用于执行bgsave命令和创建复制缓冲区。

数据同步阶段slave说明：

- 为避免slave进行全量复制、部分复制时服务器响应阻塞或数据不同步，建议关闭此期间的对外服务    slave-serve-stale-data yes|no
- 数据同步阶段，master发送给slave信息可以理解master是slave的一个客户端，主动向slave发送 命令。
- 多个slave同时对master请求数据同步，master发送的RDB文件增多，会对带宽造成巨大冲击，如果 master带宽不足，因此数据同步需要根据业务需求，适量错峰。
- slave过多时，建议调整拓扑结构，由一主多从结构变为树状结构，中间的节点既是master，也是 slave。注意使用树状结构时，由于层级深度，导致深度越高的slave与最顶层master间数据同步延迟较大，数据一致性变差，应谨慎选择。

##### 命令传播阶段：

 当master数据库状态被修改后，导致主从服务器数据库状态不一致，此时需要让主从数据同步到一致的状态，同步的动作称为**命令传播**。



命令传播阶段的部分复制：

- 命令传播阶段出现了断网现象
  - 网络闪断闪联                     忽略
  - 短时间网络中断                 部分复制
  - 长时间网络中断                 全量复制

- 部分复制的三个核心要素
  - 服务器的运行id（run id）
  - 主服务器的复制积压缓冲区
  - 主从服务器的复制偏移量

###### 服务器运行ID（runid）

- 概念：服务器运行ID是每一台服务器每次运行的身份识别码，一台服务器多次运行可以生成多个运行id。
- 组成：运行id由40位字符组成，是一个随机的十六进制字符。
- 作用：运行id被用于在服务器间进行传输，识别身份。如果想两次操作均对同一台服务器进行，必须每次操作携带对应的运行id，用于对方识别。
- 实现方式：运行id在每台服务器启动时自动生成的，master在首次连接slave时，会将自己的运行ID发送给slave，slave保存此ID，通过info Server命令，可以查看节点的runid。

###### 复制缓冲区

- 概念：复制缓冲区，又名复制积压缓冲区，是一个**先进先出（FIFO）的队列**，用于存储服务器执行过的命令，每次传播命令，master都会将传播的命令记录下来，并存储在复制缓冲区。

- 复制缓冲区内部工作原理
  - 组成：偏移量，字节值。
  - 工作原理：
    - 通过offset区分不同的slave当前数据传播的差异
    - master记录已发送的信息对应的offset
    - slave记录已接收的信息对应的offset
  - 由来：每台服务器启动时，如果开启有AOF或被连接成为master节点，即创建复制缓冲区
  
  数据同步阶段+命令传播阶段工作流程
  
  ![image-20210717072020089](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210717072020089.png)

- 心跳机制

  -  进入命令传播阶段候，master与slave间需要进行信息交换，使用心跳机制进行维护，实现双方连接保持在线。
  - master心跳：
    - 指令：PING
    - 周期：由repl-ping-slave-period决定，默认10秒
    - 作用：判断slave是否在线
    - 查询：INFO replication    获取slave最后一次连接时间间隔，lag项维持在0或1视为正常

  - slave心跳任务
    - 指令：REPLCONF ACK {offset}
    - 周期：1秒
    - 作用1：汇报slave自己的复制偏移量，获取最新的数据变更指令。
    - 作用2：判断master是否在线
  - 心跳阶段注意事项
    - 当slave多数掉线，或延迟过高时，master为保障数据稳定性，将拒绝所有信息同步操作。min-slaves-to-write 2   min-slaves-max-lag 8     slave数量少于2个，或者所有slave的延迟都大于等于10秒时，强制关闭master写功能，停止数据同步。
    - slave数量由slave发送REPLCONF ACK命令做确认
    - slave延迟由slave发送REPLCONF ACK命令做确认

##### 主从复制工作流程（完整）

![image-20210717075012957](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210717075012957.png)

#### 11.哨兵模式

哨兵(sentinel) 是一个分布式系统，用于对主从结构中的每台服务器进行监控，当出现故障时通过投票机制选择新的 master并将所有slave连接到新的master。

哨兵的作用：

- 监控     不断的检查master和slave是否正常运行。 master存活检测、master与slave运行情况检测。

- 通知（提醒）  当被监控的服务器出现问题时，向其他（哨兵间，客户端）发送通知。

- 自动故障转移    断开master与slave连接，选取一个slave作为master，将其他slave连接到新的master，并告知客户端新的服务器地址。

  哨兵也是一台redis服务器，只是不提供数据服务。通常哨兵配置数量为单数

#### 12.cluster

集群：集群就是使用网络将若干台计算机联通起来，并提供统一的管理方式，使其对外呈现单机的服务效果。

集群作用：

- 分散单台服务器的访问压力，实现负载均衡。
- 分散单台服务器的存储压力，实现可扩展性。
- 降低单台服务器宕机带来的业务灾难。

```
cluster配置
cluster-enabled yes|no               添加节点
cluster-config-file <filename>       cluster配置文件名，该文件属于自动生成，仅用于快速查找文件并查询文件内容
cluster-node-timeout <milliseconds>  节点服务响应超时时间，用于判定该节点是否下线或切换为从节点
cluster-migration-barrier <count>    master连接的slave最小数量

cluster节点操作
cluster nodes                        查看集群节点信息
cluster replicate <master-id>        进入一个从节点 redis，切换其主节点
cluster meet ip:port                 发现一个新节点，新增主节点
cluster forget <id>                  忽略一个没有solt的节点
cluster failover                     手动故障转移

redis-trib命令
redis-trib.rb add-node               添加节点
redis-trib.rb del-node               删除节点
redis-trib.rb reshard                重新分片
```

#### 13.企业级解决方案

##### 1.缓存预热

问题排查：

1. 请求数量较高
2. 主从之间数据吞吐量较大，数据同步操作频度较高

解决方案：

- 前置准备工作：
  - 日常例行统计数据访问记录，统计访问频度较高的热点数据。
  - 利用LRU数据删除策略，构建数据留存队列。   例如：storm与kafka配合

- 准备工作：
  - 将统计结果中的数据分类，根据级别，redis优先加载级别较高的热点数据
  - 利用分布式多服务器同时进行数据读取，提速数据加载过程
  - 热点数据主从同时预热

- 实施：
  - 使用脚本程序固定触发数据预热过程。
  - 如果条件允许，使用了CDN（内容分发网络），效果会更好

- 总结：
  - 缓存预热就是系统启动前，提前将相关的缓存数据直接加载到缓存系统。避免在用户请求的时候，先查询数据库，然后再将数据缓存的问题！用户直接查询事先被预热的缓存数据！

##### 2.缓存雪崩

问题排查：

1.  **在一个较短的时间内，缓存中较多的key集中过期**
2.  数据库同时接收到大量的请求无法及时处理
3.  Redis服务器资源被严重占用，Redis服务器崩溃

解决方案：

- 思路

  - 更多的页面静态化处理
  - 构建多级缓存架构     Nginx缓存+redis缓存+ehcache缓存
  - 检测Mysql严重耗时业务进行优化     对数据库的瓶颈排查：例如超时查询、耗时较高事务等
  - 灾难预警机制：监控redis服务器性能指标
    - CPU占用、CPU使用率
    - 内存容量
    - 查询平均响应时间
    - 线程数

  - 限流、降级    短时间范围内牺牲一些客户体验，限制一部分请求访问，降低应用服务器压力，待业务低速运转后再逐步放开访问

- 方法

  - LRU与LFU切换
  - 数据有效期策略调整
    - 根据业务数据有效期进行分类错峰，A类90分钟，B类80分钟，C类70分钟
    - 过期时间使用固定时间+随机值的形式，稀释集中到期的key的数量

  -  超热数据使用永久key
  - 定期维护（自动+人工） 对即将过期数据做访问量分析，确认是否延时，配合访问量统计，做热点数据的延时
  - 加锁（慎用）

- 总结：缓存雪崩就是瞬间过期数据量太大，导致对数据库服务器造成压力。如能够有效避免过期时间集中，可以有效解决雪崩现象的出现 （约40%），配合其他策略一起使用，并监控服务器的运行数据，根据运行记录做快速调整。

##### 3.缓存击穿

- 问题排查：
  - Redis中某个key过期，该key访问量巨大
  - 多个数据请求从服务器直接压到Redis后，均未命中
  - Redis在短时间内发起了大量对数据库中同一数据的访问

- 问题分析
  - 单个key高热数据
  -  key过期

- 解决方案

  - 预先设定

    以电商为例，每个商家根据店铺等级，指定若干款主打商品，在购物节期间，加大此类信息key的过期时长。   注意：购物节不仅仅指当天，以及后续若干天，访问峰值呈现逐渐降低的趋势

  - 现场调整     监控访问量，对自然流量激增的数据延长过期时间或设置为永久性key
  - 后台刷新数据     启动定时任务，高峰期来临之前，刷新数据有效期，确保不丢失
  - 二级缓存     设置不同的失效时间，保障不会被同时淘汰就行
  - 加锁      分布式锁，防止被击穿，但是要注意也是性能瓶颈，慎重！

- 总结：缓存击穿就是单个高热数据过期的瞬间，数据访问量较大，未命中redis后，发起了大量对同一数据的数据库访问，导致对数据库服务器造成压力。应对策略应该在业务数据分析与预防方面进行，配合运行监控测试与即时调整策略，毕竟单个key的过期监控难度 较高，配合雪崩处理策略即可。

##### 4.缓存穿透

- 问题排查：
  - Redis中大面积出现未命中
  -  出现非正常URL访问

- 问题分析：
  - 获取的数据在数据库中也不存在，数据库查询未得到对应数据
  -  Redis获取到null数据未进行持久化，直接返回
  -  下次此类数据到达重复上述过程
  - 出现黑客攻击服务器

- 解决方案：

  - 缓存null     对查询结果为null的数据进行缓存（长期使用，定期清理），设定短时限，例如30-60秒，最高5分钟
  - 白名单策略：
    - 提前预热各种分类数据id对应的bitmaps，id作为bitmaps的offset，相当于设置了数据白名单。当加载正常数据时，放行，加载异常数据时直接拦截（效率偏低）
    - 使用布隆过滤器（有关布隆过滤器的命中问题对当前状况可以忽略）

  - 实施监控     实时监控redis命中率（业务正常范围时，通常会有一个波动值）与null数据的占比
    - 非活动时段波动：通常检测3-5倍，超过5倍纳入重点排查对象
    - 活动时段波动：通常检测10-50倍，超过50倍纳入重点排查对象 根据倍数不同，启动不同的排查流程。然后使用黑名单进行防控（运营）

  - key加密     问题出现后，临时启动防灾业务key，对key进行业务层传输加密服务，设定校验程序，过来的key校验。例如每天随机分配60个加密串，挑选2到3个，混淆到页面数据id中，发现访问key不满足规则，驳回数据访问
  - 总结：缓存击穿访问了不存在的数据，跳过了合法数据的redis数据缓存阶段，每次访问数据库，导致对数据库服务器造成压力。通常此类数据的出现量是一个较低的值，当出现此类情况以毒攻毒，并及时报警。应对策略应该在临时预案防范方面多做文章。 无论是黑名单还是白名单，都是对整体系统的压力，警报解除后尽快移除

#### 14.redis基本类型底层实现

##### String

redis的源码是由c语言编写的，但是String类型不是c语言中的字符串类型，redis自己构建了一个叫做**动态字符串（SDS）**的抽象类型，并将SDS作为Redis的默认字符串。

```SDS
struct sdshdr{
     //记录buf数组中已使用字节的数量
     //等于 SDS 保存字符串的长度
     int len;
     //记录 buf 数组中未使用字节的数量
     int free;
     //字节数组，用于保存字符串
     char buf[];
}
```

很显然，比c语言对字符串的定义多出了len属性以及free属性。这样带来很多好处：

- **常数复杂度获取字符串长度**：由于 len 属性的存在，我们获取 SDS 字符串的长度只需要读取 len 属性，时间复杂度为 O(1)。而对于 C 语言，获取字符串的长度通常是经过遍历计数来实现的，时间复杂度为 O(n)。通过 strlen key 命令可以获取 key 的字符串长度。
- **杜绝缓冲区溢出**：我们知道在 C 语言中使用 strcat 函数来进行两个字符串的拼接，一旦没有分配足够长度的内存空间，就会造成缓冲区溢出。而对于 SDS 数据类型，在进行字符修改的时候，会首先根据记录的 len 属性检查内存空间是否满足需求，如果不满足，会进行相应的空间扩展，然后在进行修改操作，所以不会出现缓冲区溢出。
- **减少修改字符串的内存重新分配次数**：C语言由于不记录字符串的长度，所以如果要修改字符串，必须要重新分配内存（先释放再申请），因为如果没有重新分配，字符串长度增大时会造成内存缓冲区溢出，字符串长度减小时会造成内存泄露。而SDS由于len属性以及free属性，可以减少字符串的内存重新分配次数，主要实现了**空间预分配**和**惰性空间释放**两种策略。
  - 空间预分配：对字符串进行空间扩展的时候，扩展的内存比实际需要的多，这样可以减少连续执行字符串增长操作所需的内存重分配次数。
  - 惰性空间释放：对字符串进行缩短操作时，程序不立即使用内存重新分配来回收缩短后多余的字节，而是使用 free 属性将这些字节的数量记录下来，等待后续使用。

- **二进制安全**：因为C字符串以空字符作为字符串结束的标识，而对于一些二进制文件（如图片等），内容可能包括空字符串，因此C字符串无法正确存取；而所有 SDS 的API 都是以处理二进制的方式来处理 buf 里面的元素，并且 SDS 不是以空字符串来判断是否结束，而是以 len 属性表示的长度来判断字符串是否结束。
- **兼容部分 C 字符串函数**

一般来说，SDS 除了保存数据库中的字符串值以外，SDS 还可以作为缓冲区（buffer）：包括 AOF 模块中的AOF缓冲区以及客户端状态中的输入缓冲区。

##### list

链表是一种常用的数据结构，c语言内部没有内置这种数据结构，所以Redis自己构建了链表的实现。

```list
typedef  struct listNode{
       //前置节点
       struct listNode *prev;
       //后置节点
       struct listNode *next;
       //节点的值
       void *value;  
}listNode

typedef struct list{
     //表头节点
     listNode *head;
     //表尾节点
     listNode *tail;
     //链表所包含的节点数量
     unsigned long len;
     //节点值复制函数
     void (*free) (void *ptr);
     //节点值释放函数
     void (*free) (void *ptr);
     //节点值对比函数
     int (*match) (void *ptr,void *key);
}list;
```

##### 字典

hash数据类型redis是以一种字典的方式存储的。字典又称为符号表或者关联数组或映射（map），是一种用于保存键值对的抽象数据结构。字典中的每一个键 key 都是唯一的，通过 key 可以对值来进行查找或修改。C 语言中没有内置这种数据结构的实现，所以字典依然是 Redis自己构建的。

```hash
typedef struct dictht{
     //哈希表数组
     dictEntry **table;
     //哈希表大小
     unsigned long size;
     //哈希表大小掩码，用于计算索引值
     //总是等于 size-1
     unsigned long sizemask;
     //该哈希表已有节点的数量
     unsigned long used;
 
}dictht

哈希表是由数组 table 组成，table 中每个元素都是指向 dict.h/dictEntry 结构
typedef struct dictEntry{
     //键
     void *key;
     //值
     union{
          void *val;
          uint64_tu64;
          int64_ts64;
     }v;
 
     //指向下一个哈希表节点，形成链表
     struct dictEntry *next;
}dictEntry
```

(用拉链法来解决hash冲突问题)

好处

- **哈希算法**

  ```
  #1、使用字典设置的哈希函数，计算键 key 的哈希值
  hash = dict->type->hashFunction(key);
  #2、使用哈希表的sizemask属性和第一步得到的哈希值，计算索引值
  index = hash & dict->ht[x].sizemask;
  ```

- **解决哈希冲突**：通过字典里面的 *next 指针指向下一个具有相同索引值的哈希表节点。
- **扩容和收缩**：当哈希表保存的键值对太多或者太少时，就要通过 rerehash(重新散列）来对哈希表进行相应的扩展或者收缩。具体步骤：
  1. 如果执行扩展操作，会基于原哈希表创建一个大小等于 ht[0].used*2n 的哈希表（也就是每次扩展都是根据原哈希表已使用的空间扩大一倍创建另一个哈希表）。相反如果执行的是收缩操作，每次收缩是根据已使用空间缩小一倍创建一个新的哈希表。
  2. 重新利用上面的哈希算法，计算索引值，然后将键值对放到新的哈希表位置上。
  3. 所有键值对都迁徙完毕后，释放原哈希表的内存空间。

- **触发扩容的条件**：
  1. 服务器目前没有执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子（哈希表已保存节点数量 / 哈希表大小）大于等于1。
  2. 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于5。

- **渐近式 rehash**：

  也就是说扩容和收缩操作不是一次性、集中式完成的，而是分多次、渐进式完成的。如果保存在Redis中的键值对只有几个几十个，那么 rehash 操作可以瞬间完成，但是如果键值对有几百万，几千万甚至几亿，那么要一次性的进行 rehash，势必会造成Redis一段时间内不能进行别的操作。所以Redis采用渐进式 rehash,这样在进行渐进式rehash期间，字典的删除查找更新等操作可能会在两个哈希表上进行，第一个哈希表没有找到，就会去第二个哈希表上进行查找。但是进行增加操作，一定是在新的哈希表上进行的。

##### 跳跃表

跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其它节点的指针，从而达到快速访问节点的目的。

```
typedef struct zskiplistNode {
     //层
     struct zskiplistLevel{
           //前进指针
           struct zskiplistNode *forward;
           //跨度
           unsigned int span;
     }level[];
 
     //后退指针
     struct zskiplistNode *backward;
     //分值
     double score;
     //成员对象
     robj *obj;
 
} zskiplistNode

多个跳跃表节点构成一个跳跃表：
typedef struct zskiplist{
     //表头节点和表尾节点
     structz skiplistNode *header, *tail;
     //表中节点的数量
     unsigned long length;
     //表中层数最大的节点的层数
     int level;
 
}zskiplist;
```

搜索过程：从最高层的链表节点开始，如果比当前节点要大和比当前层的下一个节点要小，那么则往下找，也就是和当前层的下一层的节点的下一个节点进行比较，以此类推，一直找到最底层的最后一个节点，如果找到则返回，反之则返回空。

插入过程：首先确定插入的层数，有一种方法是假设抛一枚硬币，如果是正面就累加，直到遇见反面为止，最后记录正面的次数作为插入的层数。当确定插入的层数k后，则需要将新元素插入到从底层到k层。

删除过程：在各个层中找到包含指定值的节点，然后将节点从链表中删除即可，如果删除以后只剩下头尾两个节点，则删除这一层。

##### 整数集合（inset）

intset是Redis用于保存整数值的集合抽象数据类型，它可以保存类型为int16_t、int32_t 或者int64_t 的整数值，并且保证集合中不会出现重复元素。

```
typedef struct intset{
     //编码方式
     uint32_t encoding;
     //集合包含的元素数量
     uint32_t length;
     //保存元素的数组
     int8_t contents[];
 
}intset;
```

整数集合的每个元素都是 contents 数组的一个数据项，它们按照从小到大的顺序排列，并且不包含任何重复项。

length 属性记录了 contents 数组的大小。需要注意的是虽然 contents 数组声明为 int8_t 类型，但是实际上contents 数组并不保存任何 int8_t 类型的值，其真正类型有 encoding 来决定。

**升级**

当我们新增的元素类型比原集合元素类型的长度要大时，需要对整数集合进行升级，才能将新元素放入整数集合中。

①根据新元素类型，扩展整数集合底层数组的大小，并为新元素分配空间。

②将底层数组现有的所有元素都转成与新元素相同类型的元素，并将转换后的元素放到正确的位置，放置过程中，维持整个元素顺序都是有序的。

③将新元素添加到整数集合中（保证有序）。

升级可以极大的节省内存

**降级**

整数集合不支持降级操作，一旦对数组进行了升级，编码就会一直保持升级后的状态。

##### 压缩列表（ziplist）

压缩列表（ziplist）是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

**原理：压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存。**

压缩列表的每个节点构成

1. previous_entry_ength：记录压缩列表前一个字节的长度。
2. encoding：节点的encoding保存的是节点的content的内容类型以及长度，encoding类型一共有两种，一种字节数组一种是整数，encoding区域长度为1字节、2字节或者5字节长。
3. content：content区域用于保存节点的内容，节点内容类型和长度由encoding决定。