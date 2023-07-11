# Redis

![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213034.png))
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">redis五大基础数据结构</div>
</center>

> 随着 Redis 版本的更新，后面又支持了四种数据类型：BitMap（2.2 版新增）、HyperLogLog（2.8 版新增）、GEO（3.2 版新增）、Stream（5.0 版新增）。Redis 五种数据类型的应用场景：

- String 类型的应用场景：缓存对象、常规计数、分布式锁、共享 session 信息等。
- List 类型的应用场景：消息队列（但是有两个问题：1. 生产者需要自行实现全局唯一 ID；2. 不能以消费组形式消费数据）等。
- Hash 类型：缓存对象、购物车等。
- Set 类型：聚合计算（并集、交集、差集）场景，比如点赞、共同关注、抽奖活动等。
- Zset 类型：排序场景，比如排行榜、电话和姓名排序等。

>Redis 后续版本又支持四种数据类型，它们的应用场景如下：

- BitMap（2.2 版新增）：二值状态统计的场景，比如签到、判断用户登陆状态、连续签到用户总数等；
- HyperLogLog（2.8 版新增）：海量数据基数统计的场景，比如百万级网页 UV 计数等；
- GEO（3.2 版新增）：存储地理位置信息的场景，比如滴滴叫车；
- Stream（5.0 版新增）：消息队列，相比于基于 List 类型实现的消息队列，有这两个特有的特性：自动生成全局唯一消息ID，支持以消费组形式消费数据。

---
**为热点信息加速查询**
## 数据类型

---
### redis的基本操作
---
1. 添加信息
 功能:设置key value 数据
- 命令
```
set key value
```
2. 信息查询
 功能:根据key查询对应的value 若不存在 返回空 nil
- 命令:
```
get key
```
### redis 的数据类型
---
string list hash set zset
#### string
---
- 删除指令
```
del key
```
- 添加/修改多个数据
```
mset key1 value 1 key2 value2 ...
```
- 获取多个数据
```
get key1 key2 ...
```
- 获取数据字符个数
```
strlen key
```
- 追加信息到原始信息的后补 若原始信息存在就追加 否则新建
```
append key value
```
##### 单数据操作与多数据操作的选择
数量大的话尽量选择多数据操作 但是对于大型系统 要考虑数据的分割
- 设置数值数据增加指定范围的值
```
incr key
incr key increment
incrbyfloat key increment
```
- 设置数值数据减少指定范围的值
```
decr key
decrby key increment
```
redis的所有操作都是原子性的 采用单线程处理所有业务
- 设置数据具有指定的生命周期
```
setex key seconds value
psetex key milliseconds value
```
redis控制数据的生命周期,通过数据是否失效控制业务行为,适用于所有具有时效性限定控制的操作
##### string类型数据操作的注意事项
- 数据操作不成功的反馈与数据正常操作之间的差异
    1.  表示允许结果是否成功
        - (integer)0 --> false  失败
        - (integer)1 --> true   成功
    2.  表示运行结果值
        - (integer)3 --> 3 3个
        - (integer)1 --> 1 1个
- 数据未获取到
    nil等同与null
- 数值型数据计算的最大范围
    Java中long的范围
#### list
---
- 数据存储需求:存储多个数据,并对数据进入存储空间的顺序进行区分
- 需要的存储结构: 一个存储空间保存多个数据 且通过数据可以体现进入顺序
- list类型: 保存多个数据,地城使用双向链表存储结构实现
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210825234059.png)
##### 基本操作
- 添加/修改数据
```
lpush key value1 [value2] ...
rpush key value1 [value2] ...
```
- 获取数据
```
lrange key start stop
lindex key index
llen key
```
- 获取并移除数据
```
lpop key
rpop key
```
- 规定时间内获取并移除数据
```
blpop key1 [key2] timeout
brpop key1 [key2] timeout
```
- 移除指定数据
```
lrem key count value
```
redis应用与具有操作先后顺序的数据控制
##### 操作注意事项
- list中保存的数据都是string类型
- list具有索引的概念,但是操作数据时通常以队列的形式进行入队出队操作,或以栈的形式进行入栈出栈操作
- 获取全部数据操作结束索引设置为-1
- list可以对数据进行分页操作,通常第一页的信息来自list,第二页及更多的信息通过数据库的形式加载

redis应用于最新消息的展示
#### hash
---
新的存储需求: 对一系列存储的数据进行编组 方便管理 典型的应用就是存储对象信息
需要的存储结构: 一个存储空间保存多个键值对数据
hash类型:底层使用哈希表结构实现数据存储
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210825225054.png)
hash存储结构优化:
 - 如果field数量较少 存储结构优化为类数组结构
 - 如果field数量较多 存储数据使用HashMap结构
##### 基本操作
- 添加/修改数据
```
hset key field value
```
- 获取数据
```
hget key field 
hgetall key
```
- 删除数据
```
hdel key field1 [field2]
```
- 添加/修改多个数据
```
hmset key field1 value1 field2 value2...
```
- 获取多个数据
```
hmget key field1 field2 ... 
```
- 获取哈希表中字段数量
```
hlen key
```
- 获取哈希表中是否存在指定的字段
```
hexists key field
```
- 获取哈希表中所有的字段名或字段值
```
hkeys key
hvals key
```
- 设置指定字段的数值数据增加指定范围的值
```
hincrby key field increment
hincrbyfloat key field increment 
```
- field对应的有值 则添加失败 没有值 则添加成功
```
hsetnx key field value 
```
操作注意事项:
- hash类型显得value只能存储字符串 不允许存储其他数据类型 
- 每个hash可以存储的键值对是有上限的
- hash类型十分贴近对象的存储形式,并且可以灵活添加删除对象属性 但hash的设计初衷并不是为了存储大量对象而设计的,切记不可滥用
- hgetall操作可以获取全部属性 但是要慎用
#### set
---
- 新的存储需求: 存储大量数据,在查询方面提供更高效的效率
- 需要的存储结构: 能够保存大量的数据,搞笑的内部存储机制,便于查询
- set类型: 与hash存储结构完全相同,进存储键,不存储值nil 并且值是不允许重复的
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210826003532.png)
##### 基本操作
- 添加数据
```
sadd key member1 [member2] 
```
- 获取全部数据
```
smembers key
```
- 删除数据
```
srem key member1 [member2]
```
- 获取集合数据总量
```
scard key
```
- 判断集合中是否包含指定数据
```
sisimember key member
```
- 随机获取集合中指定数量的数据
```
srandmember key [count]
```
- 随机获取集合中的某个数据并将该数据移出集合
```
spop key
```
redis应用与随机推荐类信息检索
- 求两个集合的交/并差集
```
sinter key1 [key2]
sunion key1 [key2]
sdiff key1 [key2]
```
- 求两个集合的交/并/差集并存储到指定集合中
```
sinterstore destination key1 [key2]
sunionstore destination key1 [key2]
sdiffstore destination key1 [key2]
```
- 将指定数据从原始集合中移动到目标集合中
```
smove source destination member
```
redis应用于同类心底的关联搜索 二度关联搜索 深度关联搜索

redis应用与同类型不重复数据的合并操作

redis应用于同类型数据的快速去重

redis应用于基于黑名单与白名单设定的服务控制

##### 注意事项
- set类型不允许数据重复 重复添加会失效
- set虽然与hash的存储结构相同,但是无法启用hash中存储值的空间

#### zset
---
- 新的存储需求: 数据排序需要有利于数据的有效展示(排序)
- 需要的存储结构: 新的存储模型,可以保存可排序的数据
- zset类型:在set的存储结构基础上添加可排序字段
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210826013650.png)
##### 基本操作
- 添加数据
```
zadd key score1 member [score2 member2]
```
- 获取全部数据
```
zrang key start stop [WITHSCORES]
zrevrang key start stop [WITHSCORES]
```
- 删除数据
```
zrem key member [member]
```
- 按条件获取数据
```
zrangebyscore key min max [WITHSCORES] [LIMIT]
zrevrangebyscore key max min [WITHSCORES]
```
- 条件删除数据
```
zremrangebyrank key start stop //按索引删除
zremrangebyscore key min max
```
- 获取集合数据总量
```
zcard key
zcount key min max
```
- 集合交/并操作
```
zinterstore destination numkeys key [key ...] 
zunionstore destination numkeys key [key ...]
```
- 获取数据对应的索引 (排名)
```
zrank key member
zrevrank key member
```
- score值的获取与修改
```
zscore key member
zincrby key increment member
```
redis应用于计数器组合排序功能对应的排名
##### 操作注意事项
- score存储的数据存储空间是64位,数字有范围
- score保存的数据也可以是一个双精度的double值,基于双精度浮点数的特征,使用时可能会丢失精度,使用时要慎重
- zset地城存储韩式基于set结构的,因此数据不能重复,若添加相同的数据,score值将被反复覆盖,保存最后一次修改的结果

redis应用于定时任务执行顺序管理或任务过期管理

redis用于带有权限的即时任务/消息队列执行管理

### 数据类型实践案例

redis应用于限时按次的服务控制 
     
     优化:设置值为long的最大范围-限制次数

redis应用与基于时间顺序的数据操作 而不关注时间
---
## 通用命令
---
### key通用命令
---
- key是一个字符串 通过key获取redis保存的数据
基本操作
- 删除指定key
```
del key
```
- 获取key是否存在
```
exists key
```
- 获取key的类型
```
type key
```
- 为指定key设置有效期
```
expire key seconds
pexpire key milliseconds
expireat key timestamp //使用时间戳
pexpireat key milliseconds-timestamp 
```
- 获取key的有效时间
```
ttl key
pttl key
```
- 切换key从时效性转为永久性
```
persist key
```
- 查询key
``` 
keys pattern
```
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210826030320.png)

- 为key改名
```
rename key newkey //重复覆盖
renamenx key newkey // 重复失败
```
- 对所有key排序
```
sort
```
- 其他key通用操作
```
help #generic
```

### 数据库通用命令
---
##### 基本操作
---
- 切换数据库
```
select index
```
- 其他
```
quit
ping
echo message
```
- 数据移动
```
move key db //移动删除
```
---
### redis的持久化
---
#### 持久化过程保存什么
---
- 将当前数据状态进行保存,快照形式/存储数据结果/存储格式简单,关注点在数据 RDB
- 将数据额操作过程进行保存,日志形式/存储操作过程/存储格式复杂 关注点在数据的操作过程 AOF
---
##### RDB
RDB启动方式 --->save命令
```
save
```
作用: 
    
    手动执行一次保存操作

结果:
    
    生成一个dump.rdb的二进制文件 为保存的数据快照
```
bgsave
```
手动启动后台保存操作 但是不是立即执行
linux fork一个子进程 完成save

```
save second changes
```
    作用:   满足限定时间范围内key的变化数量达到指定数量即执行持久化
    位置:   在conf文件中进行配置
---
##### AOF
---
概念:**改记录数据为记录数据产生的过程**
- AOF持久化:以独立日志的方式记录每次写命令 重启时再重新执行AOF文件中的命令达到恢复数据的目的
- AOF的主要作用时解决了数据持久化的实时性 (主流方式)

AOF写数据过程:
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210826170035.png)
AOF写数据的三种策略:
- always(每次)
- everysec(每秒)
- no(系统控制)
AOF功能开启
- 配置
```
appendonly yes|no
```
- 作用

 是否开启AOF持久化功能 默认为不开启状态
- 配置
```
appendfsync always|everysec|no
```
- 作用

配置AOF的写策略

##### AOF重写
为了解决AOF文件体积膨胀的问题，Redis提供了AOF重写功能：Redis服务器可以创建一个新的AOF文件来替代现有的AOF文件，新旧两个文件所保存的数据库状态是相同的，但是新的AOF文件不会包含任何浪费空间的冗余命令，通常体积会较旧AOF文件小很多。

AOF重写方式
- 手动重写
```redis
bgrewriteaof
```
- 自动重写
```
auto-aof-rewrite-min-size size
auto-aof-rewrite-percentage percentage //百分比
```
- 自动重写出发比对参数
```
aof_current_size
aof_base_size
```
RDB 和 AOF 对比

| 持久化方式 | RDB             | AOF             |
| ---------- | --------------- | --------------- |
| 占用空间   | 小(数据级:压缩) | 大(指令级:重写) |
| 存储速度   | 慢              | 快              |
| 恢复速度   | 快              | 慢              |
| 数据安全性 | 会丢失数据      | 依据策略决定    |
| 资源消耗   | 高/重量级       | 低/轻量级       |
| 启动优先级 | 低              | 高              |
---
RDB与AOF的选择
- 对数据非常敏感 AOF
- 数据呈现阶段有效性 建议使用RDB持久化方案
---
### Redis事务
---
基本操作
- 开启事务
```
multi
```
- 作用 : 设定事务的开启位置,此指令执行后.后续的所有指令均加入到事务中
- 执行事务
```
exec
```
- 作用: 设定事务的结束位置,同时执行事务. 与multi成对出现,成对使用
- 取消事务
```
discard
```
- 作用: 终止当前事务的定义
- 事务的工作流程
创建队列 将指令先入队列 exec后执行
# 事务的注意事项
- 语法错误 --->事务中的所有命令均不执行
- 运行错误 ---> 能正确运行的命令会执行,运行错误的命令不会被执行

注意: 已经执行完毕的命令对应的数据不会自动回滚,需要自己再代码中实现回滚
### 锁---基于特定条件的事务执行
- 对key添加监视器 再执行exec之前如果key发生了变化,终止事务执行
```
watch key1 [key2 ...]
```
- 取消对所有key的监视
```
unwatch key
```
### 分布式锁
---
- 使用setnx 设置一个公共锁
```
setnx lock-key value
```
利用setnx命令返回值特征:优质则返回设置失败,无值则返回设置成功

    - 对于返回设置成功的,拥有控制权  进行下一步具体业务操作
    - 对于返回设置失败的,不具有控制权 排队或等待
操作完毕通过del操作释放锁
### 分布式锁改良
- 使用expire为锁key添加时间限定 到时不释放 放弃锁
- 先加锁 再加时效
```
expire lock-key second
pexpire lock-key milliseconds
```
## 删除策略
---
#### 过期数据
已经超出了为数据设置的生存时间而仍然在内存中的数据
超出了ttl的数据不一定是被真真正正地删除了
#### 数据删除策略
- 定时删除: 创建一个定时器,当key设置有过期时间,且过期时间到达时,由定时器任务立即执行对键的删除操作
  
    优点:节约内存<br>
    缺点:CPU压力大会影响redis服务器响应时间和指令吞吐量
- 惰性删除: 不做处理,等下次访问该数据时
    - 如果已过期 返回数据
    - 发现已过期 删除 返回不存在<br>
    优点:节约cpu性能<br>
    缺点:内存的压力大
- 定期删除: 
- ![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210826232325.png)
#### 逐出算法--->内存不足
- 检查易失数据(可能会过期的数据集)
  - volatile-lru 最近最少使用
  - volatile-lfu 最近使用次数最少
  - volatile-ttl 将要过期
  - volatile-random 随机
- 检测全库数据
  - allkeys-lru
  - allkeys-lfu
  - allkeys-random
- 放弃数据驱逐
  - no-enviction(驱逐) 禁止驱逐数据 回引发OOM错误
---
### 高级数据类型
---
#### Bitmaps
存储需求: 对string的位操作
##### 基础操作
- 获取指定key对应偏移量上的bit值
```
getbit key offset
```
- 设置指定key对应偏移量上的bit值,value只能是0或1
```
setbit key offset value
```
- 对指定key按位进行交/并/非/异或操作,并将结果存储到destKey中
```
bitop op destKey key [key2...]
```
    - and: 交
    - or : 并
    - not: 非
    - xor: 异或
- 统计特定key中1的数量
```
bitcount key [start end]
```
#### Hyperloglog
基数统计:数据集合去重后的元素个数
---
##### 基本操作
- 添加数据
```
pfadd key element [element ...]
```
- 统计数据
```
pfcount key [key ...]
```
- 合并数据
```
pfmerge destkey sourcekey [sourcekey ...]
```
#### GEO
---
##### 基本操作
- 添加坐标点
```
geoadd key longitude latitude member [longitude latitude member ..]
```
- 获取坐标点
```
geopos key member [member ...]
```
- 计算坐标点距离
```
geodisk key member1 member2 [unit]
```
- 根据坐标求范围内的数据
```
georadius key longitude latitude radius m|km|ft|mi [withcoord] [withdist] [withhash] [withcount]
```
- 根据点求范围内的数据
```
georadiusbymember key member  radius m|km|ft|mi [withcoord] [withdist] [withhash] [withcount]
```
- 获取指定点对应的坐标hash
```
geohash key member [member ...]
```
### 主从复制
将master中的数据即时有效的复制到slave中

    一个master--->多个slave
    一个slave --->一个master
职责: 
    
    - master:
        - 写数据
        - 执行写操作时,将出现变化的数据自动同步到slave
        - 读数据(可忽略)
    - slave:
        - 读数据
        - 写数据(禁止)
作用:
- 读写分离
- 负载均衡
- 故障排除
- 数据冗余
- 高可用基石
工作流程:
- 建立连接阶段
  - 建立slave到master的连接,使得master能够识别slave,并保存slave端口号
  ![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210827202243.png)
- 数据同步阶段
  ![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210827203132.png)
- 命令传播阶段
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20210827204443.png)
- 指令传播阶段
    - 运行id:区分不同服务器
    - 复制缓冲区(复制挤压缓冲区)
    - 偏移量(同步信息)
#### 哨兵
一个分布式系统 用于对主从结构中的每套服务器进行**监控**,当出现故障时通过投票机制**选择**新的master并将所有的slave连接到新的master
#### 企业级解决方案
##### 缓存预热
服务器启动快速宕机
---
系统启动前,提前将相关缓存数据直接加载到缓存系统

### Redis单线程问题

Redis 单线程指的是「接收客户端请求->解析请求 ->进行数据读写等操作->发生数据给客户端」这个过程是由一个线程（主线程）来完成的，这也是我们常说 Redis 是单线程的原因。

但是，Redis 程序并不是单线程的，Redis 在启动的时候，是会启动后台线程（BIO）的：
- Redis 在 2.6 版本，会启动 2 个后台线程，分别处理关闭文件、AOF 刷盘这两个任务；
- Redis 在 4.0 版本之后，新增了一个新的后台线程，用来异步释放 Redis 内存，也就是 lazyfree 线程。例如执行 unlink key / flushdb async / flushall async 等命令，会把这些删除操作交给后台线程来执行，好处是不会导致 Redis 主线程卡顿。因此，当我们要删除一个大 key 的时候，不要使用 del 命令删除，因为 del 是在主线程处理的，这样会导致 Redis 主线程卡顿，因此**我们应该使用 unlink 命令来异步删除大key**。

![ebfe79e304e3a1b24be822433c64df2f.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213446.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">redis异步线程中的生产者与消费者</div>
</center>
主线程 ---> 生产者   后台线程 --->消费者

### 缓存设计策略 
---
#### 1. 缓存雪崩
- 问题提出:
当 **大量缓存数据在同一时间过期（失效）** 时，如果此时有大量的用户请求，都无法在 Redis 中处理，于是全部请求都直接访问数据库，从而导致数据库的压力骤增，严重的会造成数据库宕机，从而形成一系列连锁反应，造成整个系统崩溃，这就是缓存雪崩的问题。
![ffe75ac676ec96a691d2595c83398ae2.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213512.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">缓存雪崩业务情景</div>
</center>

- 解决方案:

> - 将缓存失效时间随机打散： 我们可以在原有的失效时间基础上增加一个随机值（比如 1 到 10 分钟）这样每个缓存的过期时间都不重复了，也就降低了缓存集体失效的概率。
> - 设置缓存不过期： 我们可以通过后台服务来更新缓存数据，从而避免因为缓存失效造成的缓存雪崩，也可以在一定程度上避免缓存并发问题。
#### 2. 缓存击穿
- 问题提出
如果缓存中的某个**热点数据过期**了，此时大量的请求访问了该热点数据，就无法从缓存中读取，直接访问数据库，数据库很容易就被高并发的请求冲垮
![5333759b1d7c620ce279f205183c096a.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213539.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">缓存击穿业务情景</div>
</center>
- 解决方案

> - 互斥锁方案（Redis 中使用 setNX 方法设置一个状态位，表示这是一种锁定状态），保证同一时间只有一个业务线程请求缓存，未能获取互斥锁的请求，要么等待锁释放后重新读取缓存，要么就返回空值或者默认值。(请求单一)
> - 不给热点数据设置过期时间，由后台异步更新缓存，或者在热点数据准备要过期前，提前通知后台线程更新缓存以及重新设置过期时间

#### 3. 缓存穿透
- 问题提出
当用户访问的数据，**既不在缓存中，也不在数据库中**，导致请求在访问缓存时，发现缓存缺失，再去访问数据库时，发现数据库中也没有要访问的数据，没办法构建缓存数据，来服务后续的请求。那么当有大量这样的请求到来时，数据库的压力骤增。

![9d3133a9454b3d1537b1e74cd5d614d9.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213601.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">缓存穿透业务情景</div>
</center>
- 解决方案

> - **非法请求的限制**：当有大量恶意请求访问不存在的数据的时候，也会发生缓存穿透，因此在 API 入口处我们要判断求请求参数是否合理，请求参数是否含有非法值、请求字段是否存在，如果判断出是恶意请求就直接返回错误，避免进一步访问缓存和数据库。
> - **设置空值或者默认值**：当我们线上业务发现缓存穿透的现象时，可以针对查询的数据，在缓存中设置一个空值或者默认值，这样后续请求就可以从缓存中读取到空值或者默认值，返回给应用，而不会继续查询数据库。
> - **使用布隆过滤器快速判断数据是否存在，避免通过查询数据库来判断数据是否存在**：我们可以在写入数据库数据时，使用布隆过滤器做个标记，然后在用户请求到来时，业务线程确认缓存失效后，可以通过查询布隆过滤器快速判断数据是否存在，如果不存在，就不用通过查询数据库来判断数据是否存在，即使发生了缓存穿透，大量请求只会查询 Redis 和布隆过滤器，而不会查询数据库，保证了数据库能正常运行，Redis 自身也是支持布隆过滤器的。

### 常见缓存更新策略
- **Cache Aside（旁路缓存）策略**；
![4b1a0a0470fc73c1fe3a223c72122d37.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213624.png)
注意，写策略的步骤的顺序顺序不能倒过来，即**不能先删除缓存再更新数据库**，原因是在「读+写」并发的时候，会出现缓存和数据库的数据不一致性的问题
原因分析:
1. 为什么不能先删除缓存再更新数据库?
![d571586774f46a1fc4b51eade6b7b1bd.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213647.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">先删除缓存再更新数据库情景</div>
</center>
最终，该用户年龄在缓存中是 20（旧值），在数据库中是 21（新值），缓存和数据库的数据不一致。

2. 为什么「先更新数据库再删除缓存」不会有数据不一致的问题？

![bbd5c5d9f9b42585964bf7ff51e075eb.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213708.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">先更新数据库再删除缓情景</div>
</center>

因为缓存的写入通常要远远快于数据库的写入，所以在实际中很难出现请求 B 已经更新了数据库并且删除了缓存，请求 A 才更新完缓存的情况。而一旦请求 A 早于请求 B 删除缓存之前更新了缓存，那么接下来的请求就会因为缓存不命中而从数据库中重新读取数据，所以不会出现这种不一致的情况。
**Cache Aside 策略适合读多写少的场景，不适合写多的场景**

---
- Read/Write Through（读穿 / 写穿）策略；
Read/Write Through（读穿 / 写穿）策略原则是应用程序只和缓存交互，不再和数据库交互，而是由缓存和数据库交互，相当于更新数据库的操作由缓存自己代理了。
- 1、Read Through 策略
先查询缓存中数据是否存在，如果存在则直接返回，如果不存在，则由缓存组件负责从数据库查询数据，并将结果写入到缓存组件，最后缓存组件将数据返回给应用。
- 2、Write Through 策略
当有数据更新的时候，先查询要写入的数据在缓存中是否已经存在：
如果缓存中数据已经存在，则更新缓存中的数据，并且由缓存组件同步更新到数据库中，然后缓存组件告知应用程序更新完成。
如果缓存中数据不存在，直接更新数据库，然后返回；
![537823d65467cb0b0bba82e778962432.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213731.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;"> Read/Write Through（读穿 / 写穿）策略</div>
</center>

- Write Back（写回）策略；

Write Back（写回）策略在更新数据的时候，只更新缓存，同时将缓存数据设置为脏的，然后立马返回，并不会更新数据库。对于数据库的更新，会通过批量异步更新的方式进行。
**Write Back 策略特别适合写多的场景**,写操作的响应时间降低,但是带来的问题是，数据不是强一致性的，而且会有数据丢失的风险

---
## Redis实战相关

#### Redis的大 key 如何处理？
1. 找到大 key?
- redis-cli --bigkeys 查找大key
    ```shell
    redis-cli -h 127.0.0.1 -p6379 -a "password" -- bigkeys
    ```
- 使用 SCAN 命令查找大key
- 使用 RdbTools 工具查找大 key
    ```shell
     rdb dump.rdb -c memory --bytes 10240 -f redis.csv
    ```
2. 删除大 key？
>采用分批次删除的方式：
>对于 Hash，使用 hscan 扫描法；
>对于 Set，采用 srandmember 每次随机取数据进程删除；
>对于 ZSet，可以使用 zremrangebyrank 命令直接删除；
>对于 List，直接 pop 即可了；

另外，也可以采用异步删除法，用 unlink 命令代替 del 来删除，这样 Redis 会讲这个 key 放入到一个异步线程中进行删除，这样不会阻塞主线程。
3. Redis 的大 Key 对持久化有什么影响？
***i. 大 Key 对 AOF 日志的影响***
- 当使用 Always 策略的时候，如果写入是一个大 Key，主线程在执行 fsync() 函数的时候，阻塞的时间会比较久，因为当写入的数据量很大的时候，数据同步到硬盘这个过程是很耗时的。
- 当使用 Everysec 策略的时候，由于是异步执行 fsync() 函数，所以大 Key 持久化的过程（数据同步磁盘）不会影响主线程。
- 当使用 No 策略的时候，由于永不执行 fsync() 函数，所以大 Key 持久化的过程不会影响主线程。

***ii. 大 Key 对 AOF 重写和 RDB 的影响响***
******当 AOF 日志写入了很多的大 Key，AOF 日志文件的大小会很大，那么很快就会触发 AOF 重写机制******。AOF 重写机制和 RDB 快照（bgsave 命令）的过程，都会分别通过 fork() 函数创建一个子进程来处理任务。
在创建子线程的工程中,操作系统会将父进程的页表"复制"一份给子进程。页表是系统中物理地址与虚拟地址的映射关系,与父进程的页表相比,二者虽然虚拟地址不一样,但都指向相同的物理地址。
![](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/5c4f8ac682fd45221a40c4774f26f43a.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;"> Linux 父子线程复制页表 </div>
</center>
随着 Redis 存在越来越多的大 Key，那么 Redis 就会占用很多内存，对应的页表就会越大。
在通过  fork()  函数创建子进程的时候，虽然不会复制父进程的物理内存，但是内核会把父进程的页表复制一份给子进程，如果页表很大，那么这个复制过程是会很耗时的，那么在执行 fork 函数的时候就会发生阻塞现象。
而且，fork 函数是由 Redis 主线程调用的，如果 fork 函数发生阻塞，那么意味着就会阻塞 Redis 主线程。由于 Redis 执行命令是在主线程处理的，所以当 Redis 主线程发生阻塞，就无法处理后续客户端发来的命令。

如果创建完子进程后，父进程对共享内存中的大 Key 进行了修改，那么内核就会发生写时复制，会把物理内存复制一份，由于大 Key 占用的物理内存是比较大的，那么在复制物理内存这一过程中，也是比较耗时的，于是父进程（主线程）就会发生阻塞。


所以，有两个阶段会导致阻塞父进程：

- 创建子进程的途中，由于要复制父进程的页表等数据结构，阻塞的时间跟页表的大小有关，页表越大，阻塞的时间也越长；(**复制页表耗时**)
- 创建完子进程后，如果子进程或者父进程修改了共享数据，就会发生写时复制，这期间会拷贝物理内存，如果内存越大，自然阻塞的时间也越长；(**修改共享数据写时复制**)
-  Linux 开启了内存大页，会影响 Redis写操作的执行时间，最终导致 Redis 性能变慢;

***iii.大 key 除了会影响持久化之外，还会有以下的影响:***

- 客户端超时阻塞。由于 Redis 执行命令是单线程处理，然后在操作大 key 时会比较耗时，那么就会阻塞 Redis，从客户端这一视角看，就是很久很久都没有响应。
- 引发网络阻塞。每次获取大 key 产生的网络流量较大，如果一个 key 的大小是 1 MB，每秒访问量为 1000，那么每秒会产生 1000MB 的流量，这对于普通千兆网卡的服务器来说是灾难性的。
- 阻塞工作线程。如果使用 del 删除大 key 时，会阻塞工作线程，这样就没办法处理后续的命令。
- 内存分布不均。集群模型在 slot 分片均匀情况下，会出现数据和查询倾斜情况，部分有大 key 的 Redis 节点占用内存多，QPS 也会比较大。

#### 如何用 Redis 实现分布式锁的？
![339ba8778177541e0a2e9fdde95f1975.png](https://cscgblog-1301638685.cos.ap-chengdu.myqcloud.com/java/image/20221023213809.png)
<center>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">Redis做分布式锁</div>
</center>
Redis 的 SET 命令有个 NX 参数可以实现「key不存在才插入」，所以可以用它来实现分布式锁：

- 如果 key 不存在，则显示插入成功，可以用来表示加锁成功；
- 如果 key 存在，则会显示插入失败，可以用来表示加锁失败。
加锁操作的三个条件:
- 1. 加锁包括了读取锁变量、检查锁变量值和设置锁变量值三个操作，需要以原子操作的方式完成(Lua脚本完成)
- 2. 锁变量需要设置过期时间，以免客户端拿到锁后发生异常，导致锁一直无法释放
- 3. 锁变量的值需要能区分来自不同客户端的加锁操作，以免在释放锁时，出现误释放操作

#### 基于 Redis 实现分布式锁有什么优缺点？

优点:

- 性能高
- 实现方便
- 避免单点故障问题(Redis集群部署)

缺点
- 超时时间不好设置。如果锁的超时时间设置过长，会影响性能，如果设置的超时时间过短会导致锁被误删(守护线程 续约加锁)
- Redis 主从复制模式中的数据是异步复制的，这样导致分布式锁的不可靠性。(Redlock（红锁）。它是基于多个 Redis 节点的分布式锁，即使有节点发生了故障，锁变量仍然是存在的，客户端还是可以完成锁操作。
Redlock 算法的基本思路，是让客户端和多个独立的 Redis 节点依次请求申请加锁，如果客户端能够和半数以上的节点成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁，否则加锁失败。)

## 缓存不一致问题及解决方案

> 问题：先更新数据库，再删除缓存。如果删除缓存失败了，那么会导致数据库中是新数据，缓存中是旧数据，数据就出现了不一致。

解决思路 2：延时双删。依旧是先更新数据库，再删除缓存，唯一不同的是，我们把这个删除的动作，在不久之后再执行一次

```java
public void set(key, value) {
    putToDb(key, value);
    deleteFromRedis(key);
    
    
    // ... a few seconds later
    deleteFromRedis(key);
}
```

对于删除的动作，可以有多种选择，比如：1. 使用 `DelayQueue`，会随着 JVM 进程的死亡，丢失更新的风险；2. 放在 `MQ`，但编码复杂度为增加。