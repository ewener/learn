### 一.使用场景

> 可以做什么

1. 缓存：加速访问速度，降低数据库压力，提供键值过期设置、灵活控制最大内存和内存溢出后的淘汰策略。
2. 排行榜系统：提供了列表和有序集合数据结构。
3. 计数器应用：天然支持计数功能，计数性能非常好。
4. 社交网络
5. 消息队列系统

> 不可以做什么

1. 数据规模不宜太大，内存成本有限。
2. 不宜存放冷数据。

### 二. 可执行文件说明

1. redis-server：启动redis
2. redis-cli：命令行客户端
3. redis-benchmark：基准测试工具
4. redis-check-aof：aof持久化文件检测和修改工具
5. redis-check-dump：rdb持久化文件检测和修改工具
6. redis-sentinel：启动哨兵。

### 三.启动

####一 . redis-server

1. 默认配置：redis-server
2. 运行启动：redis-server --configKey1 configValue1 --configKey2 configValue2
3. 指定配置文件启动：redis-server /opt/redis/redis.conf

#### 二. redis-cli

1. 交互方式：redis-cli -h host -p port
2. 直接命令方式返回结果：redis-cli -h host -p port  get hello

> 注意：没有-h默认127.0.0.1，没有-p默认6379

#### 三.停止redis服务

1. redis-cli shutdown

> 注意：
>
> > 关闭过程：断开客户端连接，持久化文件生成，优雅的关闭方式。
>
> > 可以使用kill进程关闭redis，但是不能使用kill -9强制杀死，不但不会做持久化，还会造成缓冲区等资源不能优雅关闭，极端情况造成AOF和复制数据丢失。
>
> > redis-cli shutdown nosave|save : 是否在关闭redis前生成持久化文件。

###四. API理解并使用

#### 一.预备

> 全局命令

1. 查看所有的键：keys *，会遍历所有键，时间复杂度o(n)，**当保存了大量键值，线上禁止使用**。
2. 键总数：dbsize，不会遍历所有键，直接获取redis内置的键总数变量，时间复杂度为o(1)。
3. 检查键是否存在：exists key，存在返回1，不存在返回0。
4. 删除键：del key [key ...]，返回删除键个数，无论什么数据类型都能删除。
5. 键过期：expire key seconds，可用**“ttl key”**查看键剩余过期时间，大于等于0->键剩余过期时间，-1->键没设置过期时间，-2->键不存在
6. 键的数据结构类型：type key，如果键不存在，返回none。

> 数据结构和内部编码

1. redis对外数据结构：string，hash，list，set，zset。
2. redis的数据结构都有两种以上的内部编码实现。用命令：**object encoding key** 可查看内部编码。

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.jpg)

 这种设计好处：

1. 可以改进内部编码，而对外数据结构和命令没有影响。
2. 多种内部编码实现可以在不同场景下发挥各自优势。如ziplist比较节省内存，但在列表元素比较多时，性能会有所下降，这时候redis会根据配置选项将内部实现转换为linkedlist。

> 单线程架构

1. 单线程模型：发送命令->命令进入服务端的一个队列->逐个执行。队列命令执行顺序不会保证，但不会有两条命令同时执行，不会产生并发。
2. 为什么单线程还这么快？：①纯内存访问。②非阻塞IO，使用epoll作为IO多路复用技术的实现，加上redis自身的事件处理模型将epoll中的连接、读写、关闭都转换为事件，不在网络IO上浪费过多时间。③单线程避免了线程切换和竞态产生的消耗。
3. 单线程带来的好处：①简化数据结构和算法的实现。②单线程避免了线程切换和竞态产生的消耗，对于服务端来说，锁和线程切换通常是性能杀手。
4. 单线程带来的问题：命令执行时间是有要求的。命令执行过长，会造成其他命令阻塞。所以redis是面向快速执行场景的数据库。

#### 二.字符串

> 最基础的数据结构，redis键都是字符串类型，其他几种数据结构都是在字符串的基础上构建的。字符串类型可以是字符串(简单字符串，复杂字符串(json\xml等))、数字、二进制(图片、音频、视频)，但是**最大值不能超过512M**

##### 一.命令

> 常用命令

1. 设置值：set key value \[ex seconds] \[px milliseconds] [nx|xx]

| 选项            | 说明                                                         |
| :-------------- | :----------------------------------------------------------- |
| ex seconds      | 为键值设置秒级过期时间                                       |
| px milliseconds | 为键值设置毫秒级过期时间                                     |
| nx\|xx          | nx：键必须不存在，才能设置成功，用于添加；                                                                   xx：键必须存在，才能设置成功，用于更新。 |

另外两个相似命令：

setex  key  seconds  value

setnx key value

2. 获取值：get key，不存在返回nil

3. 批量设置值：mset key value [key value ...]

4. 批量获取值：mget key value [key value ...]，有不存在的对应返回nil

5. 计数：incr key，对值做自增操作，返回分三种情况：①值不是整数，返回错误。②值是整数，返回自增后的结果。③键不存在，按照值为0自增，返回结果为1。

   类似命令：decr(自减)，incrby(自增指定数字)，decrby(自减指定数字)，incrbyfloat(自增浮点数)

> 不常用命令

1. 追加值：append key value
2. 字符串长度：strlen key
3. 设置并返回原值：getset key value
4. 设置指定位置的字符：setrange key offeset value，偏移量从0开始
5. 获取部分字符串：getrange key start end，偏移量从0开始

![](https://github.com/XwDai/learn/raw/master/notes/image/%E5%AD%97%E7%AC%A6%E4%B8%B2%E7%B1%BB%E5%9E%8B%E5%91%BD%E4%BB%A4%E6%97%B6%E9%97%B4%E5%A4%8D%E6%9D%82%E5%BA%A6.jpg)

##### 二.内部编码

1. int：8个字节的长整型
2. embstr：小于等于39个字节的字符串
3. raw：大于39个字节的字符串

##### 三.使用场景

> 1.缓存功能
>
> > 开发提示：合理设置键，有利于防止冲突和可维护性，推荐方式“**业务名：对象名：id：[属性]**”，在能描述键含义的前提下适当减少键长度，减少内存浪费。
>
> 2.计数：incr(key)
>
> 3.共享session
>
> 4.限速：set(key,1,"EX 60","NX")
>
> ```java
> String phoneNum = "1305454516";
> String key = "shortMsg:limit:"+phoneNum
> Boolean isExists = redis,set(key,1,"EX 60","NX");
> if(isExists!=null || redis.incr(key)<=5){
>     //通过
> }else{
>     //限速
> }
> ```

#### 三.哈希

> 哈希类型是指键值本身又是一个键值对结构。

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E5%AD%97%E7%AC%A6%E4%B8%B2%E4%B8%8Ehash%E7%B1%BB%E5%9E%8B%E5%AF%B9%E6%AF%94.jpg)

##### 一.命令

1. 设置值：hset key field value
2. 获取值：hget key field，如果key不存在，返回nil。
3. 删除field：hdel key field [field ...]，删除一个或多个field，返回成功删除field个数。
4. 计算field个数：hlen key
5. 批量设置或获取：hmset key field value [field value ...]，hmget key field [field ...]
6. 判断field是否存在：hexists key field，包含返回1，不包含返回0。
7. 获取所有field：hkeys key
8. 获取所有value：hvals key
9. 获取所有的field-value：hgetall key，**如果元素个数比较多，会存在阻塞redis的可能，如果一定要获取全部的field-value，可以使用hscan，该命令会渐进式遍历hash类型**。
10. hincrby key field，  hincrbyfloat key field ：与incrby和incrbyfloat一样，但是它们作用域的field。
11. 计算value的字符串长度：hstrlen key value

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E5%93%88%E5%B8%8C%E7%B1%BB%E5%9E%8B%E6%97%B6%E9%97%B4%E5%A4%8D%E6%9D%82%E5%BA%A61.jpg)

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E5%93%88%E5%B8%8C%E7%B1%BB%E5%9E%8B%E6%97%B6%E9%97%B4%E5%A4%8D%E6%9D%82%E5%BA%A62.jpg)

##### 二.内部编码

> ziplist(压缩列表)：
>
> > 当hash类型元素个数小于hash-max-ziplist-entries配置（默认512个）、同时所有值都小于hash-max-ziplist-value配置（默认64字节）时使用。
> >
> > ziplist使用更紧凑的结构实现多个元素的连续存储，所以在节省内存方面比hashtable更加优秀。
>
> hashtable(哈希表)：
>
> > 当hash类型无法满足ziplist时使用，因为此时ziplist读写效率会下降，而hashtable读写时间复杂度为o(1)

##### 三.使用场景

#### 四.列表

> 列表是用来存储多个有序的字符串，一个列表最多可存2的32次方减1个元素。

##### 一.命令

| 操作类型 | 操作                                                         |
| -------- | ------------------------------------------------------------ |
| 添加     | 1.rpush key value [value ...]：从右插入。                                                                                                                                2.lpush key value [value ...]：从左插入。                                                                                                                                3.linsert key before\|after pivot value：从列表中找到等于pivot的元素，在前或者后插入value |
| 查找     | 1.lrange key start end：获取指定范围内的元素列表。偏移量为负数时反向。                                 2.lindex key index：获取列表指定下标元素。                                                                                      3.llen key：获取列表长度。 |
| 删除     | 1.lpop key：列表左侧弹出元素。返回弹出元素。                                                                                2.rpop key：列表右侧弹出元素。返回弹出元素。                                                                                3.lrem key count value：删除等于value的元素，count>0,从左到右，删除最多count个元素；count<0，从右到左，删除最多count个元素；count=0，删除所有。                                                4.ltrim key start end：按照索引范围修剪列表。在索引范围的保留。 |
| 修改     | 1.lset key index newValue：修改指定索引下标的元素。          |
| 阻塞操作 | 1.brpop\blpop key [key ...] timeout(s)：如果列表没有元素，阻塞到timeout返回，当timeout=0，一直阻塞。                                                                                                                                注意：多个key时，会从左至右遍历键，有一个能弹出就返回。  如果多个客户端对通一个key阻塞操作，先执行命令的客户端可以获取到弹出值。 |

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E5%88%97%E8%A1%A8%E6%97%B6%E9%97%B4%E5%A4%8D%E6%9D%82%E5%BA%A6.jpg)

##### 二.内部编码

> ziplist:
>
> > 当列表元素个数小于list-max-ziplist-entries配置（默认512个）、同时所有值都小于list-max-ziplist-value配置（默认64字节）时使用。

> linkedlist:
>
> > 当无法满足ziplist条件时使用。
>
> 开发提示：redis3.2中提供了quicklist内部编码，简单说它是一个以ziplist为节点的linkedlist，结合了两者的优势。

##### 三.使用场景

1. 消息队列：lpush+brpop
2. 栈：lpush+lpop
3. 有限集合：lpush+ltrim
4. 文章列表

#### 五.集合

> 用以保存多个字符串元素，不允许重复，元素无序，不能索引下标获取元素。

##### 一.命令

> 集合内操作

1. 添加元素：sadd key element [element  ...]
2. 删除元素：srem key element [element  ...]，返回删除元素个数。
3. 计算元素个数：scard key，**时间复杂度o(1)，不会遍历所有元素，直接使用redis内部变量**。
4. 判断元素是否在集合中：sismember key element ，是，返回1，否，返回0。
5. 随机从集合返回指定个数元素：srandmember key [count]，count默认1。
6. 从结合**随机**弹出元素：spop key，redis3.2开始，支持[count]参数，

​    注意：srandmember 和spop 都是从集合中选出元素，不同的是**spop 会删除，srandmember 不会**。

7. 获取所有元素：smembers key，返回结果无序。

​    注意：smembers、lrange、hgetall都属于比较重的命令，元素过多存在阻塞redis可能性，这时候需要使用sscan。

> 集合间操作

1. 多个集合交集：sinter key [key ...]

2. 多个集合并集：suion key [key ...]

3. 多个集合差集：sdiff key [key ...]

4. 将交集、并集、差集结果保存：sinterstore/suionstore/sdiffstore des key [key ...]，集合的运算在元素较多的时候比较耗时，这个命令将结果保存在des中。

   ![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E9%9B%86%E5%90%88%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4%E6%97%B6%E9%97%B4%E5%A4%8D%E6%9D%82%E5%BA%A6.jpg)

##### 二.内部编码

> intset：整数集合
>
> > 当集合中元素都是整数且元素个数小于set-max-intset-entries配置时(默认512)使用，减少内存使用。
>
> hashtable：哈希表
>
> > 当intset无法满足条件时使用。

##### 三.使用场景

1. 给用户添加标签，给标签添加用户，删除用户下的标签，删除标签下的用户，计算用户共同感兴趣的标签。

​    注意：用户和标签应该在一个事务里执行。

> 开发提示：sadd = tagging(标签) ，spop/srandmember = random item(生成随机数，比如抽奖)，sadd+sinter = social graph(社交需求)

#### 六.有序集合

> 可以排序的集合，与列表使用索引下标为排序依据不同的是，它给每个元素设置一个分数score作为排序依据。**有序集合元素不可以重复，但是score可以重复**。

##### 一.命令

> 集合内

1. 添加成员：zadd key score member [score member ...] ，返回添加成功个数。

   注意：

   ①redis3.2为zadd命令添加了nx、xx、ch、incr四个选项 

   * nx：不存在才能设置成功，用于添加。
   * xx：存在才能设置成功，用于更新。
   * ch：返回此次操作后，有序集合和分数发生变化的个数。
   * incr：对score做增加，相当于zincrby。

   ②有序集合相比集合提供了排序，但是也产生了代价，zadd的时间复杂度为o(log(n))，sadd为o(1)。

2. 计算成员个数：zcard key，时间复杂度为o(1)。

3. 计算成员分数：zscore key member

4. 计算成员排名：zrank key member->返回从低到高排名，zrevrank key member->反之

5. 删除成员：zrem key member [member ...]

6. 增加成员分数：zincrby key increment member，increment 为增加的分数。

7. 返回指定排名范围成员：zrange key start end [withscores] ->从低到高返回，zrevrange key start end [withscores] -> 反之；加上withscores，会返回成员的分数。

8. 返回指定分数范围成员：zrangebyscore key min max [withscores]\[limit offset count]->按照分数从低到高返回，zrevrangebyscore key **max  min** [withscores]\[limit offset count] 反之；加上withscores，会返回成员的分数。**min和max支持开区间(小括号)和闭区间(中括号)，-inf和+inf代表无限小和无限大**。

9. 返回指定分数范围成员个数：zcount key min max

10. 删除指定排名内的升序元素：zremrangebyrank key start end

11. 删除指定分数范围成员：zremrangebyscore key mix max，返回删除个数。

> 集合间操作

1. 交集：zinterstore des numkeys key [key ...]\[weights weight [weight ...]]\[aggregate sum|min|max]

* des：保存结果键
* numkeys：需要做交集计算键的个数。
* key [key ...]：需要做交集计算的键。
* weights weight [weight ...]：每个键的权重，在做交集运算时，每个键的每个成员会将自己分数乘以这个权重，权重默认值是1。
* aggregate sum|min|max：计算交集后，分值可以按sum、min、max做汇总，默认是sum。

2. 并集：zunionstore des numkeys key [key ...]\[weights weight [weight ...]]\[aggregate sum|min|max]，与交集类似。



##### 二.内部编码

#####三.使用场景