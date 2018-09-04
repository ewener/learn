

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

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E6%9C%89%E5%BA%8F%E9%9B%86%E5%90%88%E5%91%BD%E4%BB%A4%E6%97%B6%E9%97%B4%E5%A4%8D%E6%9D%82%E5%BA%A6.jpg)

##### 二.内部编码

> ziplist：压缩列表
>
> > 当有序集合元素个数小于zset-max-ziplist-entries配置时（默认128个），同时每个元素的值都小于zset-max-ziplist-value配置（默认64字节）时使用。
>
> skiplist：跳跃表
>
> > 当ziplist条件不满足是使用。

#####三.使用场景

1. 添加用户赞数

#### 七.键管理

##### 一.单个键管理

1. 键重命名：rename key newkey

​      注意：

* 如果rename前，此newKey已经存在，那么它的值将被覆盖。为了防止强行rename，redis提供了renamenx，确保不存在时才被覆盖。
* **重命名前会先del旧的键，如果键值比较大，会存在阻塞redis的可能性**。
* 如果重命名的key和newkey相同，redis3.2之前会报错。

2. 随机返回一个当前库中的键：randomkey
3. 键过期：

* expire key seconds

* expireat key timestamp：键在秒级时间戳timestamp后过期

* pexpire key milliseconds：键在毫秒后过期。

* pexpireat key milliseconds-timestamp：键在毫秒级时间戳后过期。

  注意：

  * expire key键不存在时，返回0，这与ttl/pttl是有区别的。
  * 如果过期时间为负值，键会被立即删除。跟del一样。
  * persist key命令将键的过期时间清除。ttl查看过期时间会返回-1。
  * **对于字符串类型键，执行set命令会去掉过期时间。**
  * redis不支持二级数据结构（哈希、列表等）内部元素的过期功能，例如不能对列表类型的一个元素设置过期时间。
  * setex=set+expire，原子性，减少了一次网络通信时间。

* ttl/pttl key：查询键剩余过期时间，pttl精度更高，可达到毫秒级。

  三种返回值：

  - 大于等于0：键剩余过期时间。
  - -1：没有设置过期时间。
  - -2：键不存在。

4. 键迁移

* move key db：把key迁移到另外个db。

* dump+restore：可实现不同redis实例之间的数据迁移。分为以下两步：

  * dump key：在源redis上，将键值序列化，格式采用的是RDB格式。命令执行之后返回一个长字符串，把它作为第二步的value。
  * restore key ttl value：在目标redis上，将上面序列化的值进行复原，ttl代表过期时间，为0代表没有过期时间。

* migrate host port key|“ ” des-db timeout [copy]\[replace]\[keys key [key ...]]：用于redis实例间数据迁移，实际上就是将dump、restore、del三个命令组合。具有原子性，只需要在源redis上执行就可以了。migrate 在水平扩容中起到作用。

  参数说明：

  * key|“ ”：redis3.0.6后支持迁移多个键，如果需要迁移多个键，这里为空字符串。
  * copy：添加此项，迁移之后不删除源键。
  * replace：添加此项，不管目标redis是否存在此键，都会覆盖值。
  * \[keys key [key ...]]：迁移多键时填写，与上面的key为空串的参数对应。

5. 遍历键

* 全量遍历键：keys pattern，pattern使用的是glob风格的通配符：

  * *代表匹配任意字符。
  * ？代表匹配一个字符。
  * []代表匹配部分字符，如，[1,3]代表匹配1,3，[1-10]匹配1-10任意数字。
  * \x用了做转义，例如匹配星号、问号需要进行转义。

* 渐进式遍历：scan，有效解决了keys的问题。时间复杂度o(1)。但是需要执行多次scan才能达到keys的效果。

  * 用法：scan cursor [match pattern]\[count number]

     参数说明：

  ​    cursor ：游标，从0开始每次scan返回当前游标值。知道游标为0，标识遍历结束。

     [match pattern]：匹配键。

     \[count number]：每次遍历的键个数。默认10。

  * hgetall、smembers、zrange对应hscan、sscan、zscan。

  注意：**如果scan过程中键有变化，那么可能出现新增的键可能没有遍历到，遍历出重复的键等情况。**

6. 数据库管理

* 切换数据库（redis默认16个数据库，默认使用的0）

  * select dbindex：如select 0。

  注意：redis3.0后已逐渐弱化了这个功能，例如redis cluster只允许使用0。

  为什么要废弃这个功能？：

  1. redis单线程，即使是多个db，使用的仍然是一个cpu。彼此还是有影响。
  2. 调试运维不同业务困难。例如慢查询依然会影响其他数据库，影响别的业务方定位问题。
  3. 部分redisclient不支持这种方式，即使支持，还得来回切换，容易混乱。

* flushdb/flushall：清除数据库数据

  * 区别：flushdb只清除当前数据库，flushall清除所有。

  注意：

  1. 误操作不可设想。rename-command配置规避这个问题。
  2. 当前库键值比较多，存在阻塞redis可能性。



### 五.小功能大用处

#### 一.慢查询分析

1. 配置参数：

   * slowlog-log-slower-than：阈值，单位微妙，默认10000。=0会记录所有命令，<0不会记录任何命令。
   * slowlog-max-len：内存存储日志的列表长度。插入记录大于长度，最早的那条将移除。这个列表的键不会暴露。

   修改配置：

   ①修改配置文件。

   ②

   ```redis
   config set slowlog-log-slower-than 20000
   config set slowlog-max-len 1000
   config rewrite //持久化本地配置文件
   ```

2. 获取慢查询日志：slowlog get [n]   //n可指定条数。

3. 日志组成：日志标识id、发生时间戳、命令耗时、执行命令、参数。

4. 获取慢查询日志列表当前长度：slowlog len。

5. 慢查询日志重置：slowlog reset。

6. 最佳实践：

   * slowlog-max-len配置建议：线上建议调大，记录慢查询的redis会对长命令做截断操作，并不会占用大量内存，增大慢查询列表可以减缓慢查询被剔除的可能，例如线上可设置1000以上。
   * slowlog-log-slower-than：默认10毫秒，需要根据并发量调整，由于redis单线程，对应高流量场景，如果命令执行1毫秒以上，redis 最多支撑ops不到1000，因此高ops场景建议设置1毫秒。
   * 慢查询只记录命令执行时间，并不包括排队和网络传输时间。因此客户端执行时间>命令实际执行时间。慢查询会导致其他命令级联阻塞，因此当客户端请求超时时，检查是否有慢查询。
   * 慢查询列表的先进先出特点，慢查询多的时候，会丢失部分慢查询命令，为了防止这种情况，可以定期执行slow get将慢查询日志持久化到其他存储中。

#### 二.Redis Shell

1. redis-cli
   * -r：repeat，代表命令执行多次。例：redis-cli -r 3 ping  ->执行3次ping命令。
   * -i：interval(s)：每隔几秒执行一次，-i必须和-r同时使用。例：redis-cli -r 3 -i 1 ping
   * -x：从标准输入读取数据作为redisl-cli的最后一个参数，例：$echo “world”|redis-cli -x set hello
   * -c：cluster，连接redis cluster使用。可以防止moved和ask异常。
   * --scan和--pattern：用于扫描指定模式的键，相当于scan。
   * --slave：把当前客户端模拟成当前redis的从节点。可以用来获取当前redis节点的更新操作。当节点有数据更新时，从节点能获取到更新数据。
   * --rdb：请求redis实例生成并发送rdb文件，保存到本地。可使用它做持久化文件定期备份。
   * --pipe：将命令封装成redis通信协议定义的数据格式，批量发送给redis执行。
   * --bigkeys：使用scan对redis的键进行采样，从中找出内存占用比较大的键值。
   * --eval：执行指定lua脚本。
   * --latency：检测网络延迟。只要一条数据。
     * --latency-history：默认每隔15s输出一次，可以通过-i控制间隔时间。
     * --latency-dist：使用统计图表从控制台输出延迟统计信息。
   * --stat：实时获取redis重要统计信息，info虽然能看到更全，但不实时。
   * --raw和--no-raw：--no-raw命令返回结果必须是原始格式，--raw相反。有时需要--raw查看格式话的结果，比如中文，而不是十六进制。
2. redis-server
   * 除了之前说的启动，还有个参数，--test-memory：检测当前操作系统能否稳定的分配指定容量的内存给redis。通过检测有效避免因为内存原因造成redis崩溃。
3. redis-benchmark
   * -c：clients，客户端并发数量，默认50。
   * -n：num，代表客户端请求总量，默认100000。
   * -q：仅仅显示redis-benchmark的requests per second信息。
   * -r：random
   * -p：每个请求pipeline的数据量，默认为1。
   * -k：客户端是否使用keeplive，1为使用，0不使用，默认1。
   * -t：对指定命令基准测试。例：redis-benchmark -t get,set -q。
   * --csv：将结果按csv格式输出。

#### 三.Pipeline

1. 概念：将一组redis命令进行组装，通过一次rtt传输给redis，再将这组redis命令执行结果按顺序返回给客户端。
2. 批量命令和pipeline对比：
   * 批量命令是原子的，pipeline非原子。
   * 批量命令是一个命令对应对个key，pipeline支持多个命令。
   * 批量命令是服务端支持实现，pipeline服务端和客户端共同实现。
3. 最佳实践：
   * 组装命令的个数不能没有节制，否则一次组装数据量过大，一方面增大客户端等待时间，另一方面造成网络堵塞，可以拆成多个小的pipeline执行。
   * pipeline只能操作一个redis实例。

#### 四.事务与Lua

1. 事务：multi->执行命令->exec，停止事务：discard代替exec。

   注意：

   * 命令错误，造成事务回滚。
   * 运行时错误，语法正确，但命令用错了，事务提交报错，成功的一部分不支持回滚。
   * 为了确保事务执行时，key'没有被其他客户端修改，redis提供watch key命令解决，如果期间发生改变，则事务不执行。
   * redis的事务简单，主要是因为不支持事务中的回滚特性，同时无法实现命令之间的逻辑关系计算。

2. lua

   * eval：直接发送脚本
   * evalsha：首先将脚本加载到服务器，得到该脚本的SHA1校验和，evalsha命令使用SHA1作为参数可以直接执行对应lua脚本，避免每次发送lua脚本的开销。
     * 加载脚本：script load
     * 执行脚本：eval SHA1值  key个数  key列表  参数列表
   * lua redis api：
     * redis.call：执行失败，会返回错误。
     * redis.pcall：忽略错误继续执行脚本。
     * redis.log：将lua脚本日志输出到redis日志文件，一定要控制日志级别。redis3.2提供了lua script debugger 功能来调试复杂的lua。参考：http://redis.io/topics/ldb
   * 案例
   * redis如何管理lua脚本
     * script load：script load script，将脚本加载到redis内存中。
     * script exists：scripts exsits sha1 [sha1...]：判断sha1是否加载到redis内存。
     * script flush：清除redis内存中的所有脚本。
     * script kill：用命令杀掉正在执行的lua脚本。如果lua脚本比较耗时，甚至脚本存在问题，那么此脚本执行会阻塞redis。
       * redis 提供了lua-time-limit参数，默认5秒，但是这个超时时间仅仅是当lua脚本时间超过lua-time-limit后，向其它命令调用发送busy的信号，但不会停止服务端和客户端的脚本执行，客户端执行正常命令的时候会收到busy错误。
       * 执行script kill，客户端调用会恢复。
       * 注意：**当前lua脚本正在执行写操作，script kill 将不会生效。这时，要么等待脚本执行结束，要么使用shutdown save 停掉redis服务。**

#### 五.Bitmaps

1. 数据结构模型

   * 本身不是一种数据结构，实际上它就是字符串，但是可以对字符串的位进行操作。
   * 可以想象成一个以位为单位的数组，数组的每个单元只能存储0和1，数组的下标叫Bitmaps的偏移量。

2. 命令

   * 设置值：setbit key offset value。如果第一次初始化，偏移量很大的话，初始化过程会比较慢，可能会造成redis阻塞。

   * 获取值：getbit key offset

   * 获取Bitmaps指定范围值为1的个数：bitcount key [start]\[end]，start和end代表起始和结束的字节数。

   * Bitmaps间的运算：bitop op destkey key [key ...] ，将op运算结果保存在destkey中。

     op：

     * and：交集
     * or：并集
     * not：非
     * xor：异或

   * 计算Bitmaps中第一个值为targetBit的偏移量：bitpos key targetBit [start]\[end]，start和end代表起始和结束的字节数。

3. Bitmaps分析

   * 网站访问统计

   * Bitmaps适合数据量大的存储，数据量小反而更占内存。

#### 六.HyperLogLog

1. HyperLogLog实际为字符串类型。是一种基数算法。可利用极小空间完成独立总数的统计。
2. 命令
   * 添加：pfadd key element [element ...]，向HyperLogLog添加元素，成功返回1。
   * 计算独立用户数：pfcount key [key...]，计算一个或多个HyperLogLog的独立总数。意思是不相同的值的数量。
   * 合并：pfmerge destkey sourcekey [sourcekey ...]，求出多个HyperLogLog的并集并赋值给destkey。
3. 注意：HyperLogLog在数据量大的时候节省内存很明显，但是用如此小空间估算如此巨大的数据，必然不是100%正确，redis官方给出的是0.81%的丢失率。在使用的时候需要确认一下两条：
   * 只为了计算独立总数，不需要获取单条数据。
   * 可以容忍一定误差。

#### 七.发布订阅

1. 发布者向指定频道(channel)发布消息，订阅频道的每个客户端都可以收到该消息。
2. 命令：
   * 发布消息：publish channel message
   * 订阅消息：subscribe channel [channel...]
   * 取消订阅：unsubscribe [channel [channel...]]
   * 按照模式订阅和取消订阅,意思是可以支持glob风格的匹配：
     * psubscribe pattern [pattern ]
     * punsubscribe pattern [pattern ]
   * 查询活跃的频道：pubsub channels [pattern]，活跃是指当前频道至少有一个订阅者。
   * 查看频道订阅数：pubsub numsub [channel...]
   * 查看模式订阅数：pubsub numpat
3. 注意：
   * 客户端执行订阅命令之后进入了订阅状态，只能接收subscribe、psubscribe、unsubscribe、punsubscribe。
   * 新开启的客户端无法接收到改频道之前的信息。因为redis不会对消息持久化。

#### 八.GEO（地理信息定位）

1. redis3.2版本提供，支持存储地理位置信息实现依赖地理位置信息的功能。

2. 命令

   * 增加地理位置信息：geoadd key longitude latitude member [longitude latitude member...]

     * longitude ：经度
     * latitude ：纬度
     * member ：成员

   * 更新地理位置信息：仍然可以使用geoadd，虽然已存在成员返回结果为0。

   * 获取地理位置：geopos key member[member...]

   * 获取两个地理位置距离：geodist key member1 member2 [unit]，unit：m/km/mi/ft

   * 获取指定位置范围内的地理信息位置集合：

     * georadius key longitude latitude radiusm|km|ft|mi  [withcoord] \[withdist]\[withhash]\[COUNT count]\[asc|desc]\[store key]\[storedist key]
     * georadiusbymember key member radiusm|km|ft|mi  [withcoord] \[withdist]\[withhash]\[COUNT count]\[asc|desc]\[store key]\[storedist key]
       * withcoord：返回结果包含经纬度
       * withdist：返回结果包含离中心节点位置的距离
       * withhash：返回结果中包含geohash。
       * COUNT count：指定返回结果的数量。
       * asc|desc：返回结果按离中心节点的距离做升序和降序。
       * store key：将返回结果信息保存到指定键。
       * storedist key：将返回结果离中心节点的距离保存在指定键。

   * 获取geohash：geohash key member [member...]，将二维经纬度转换为一维字符串。

     特点：

     * GEO的数据类型为zset，redis将所有地理位置信息的geohash存放在zset中。
     * 字符串越长位置越精确。
     * 字符串越相似，距离越近。redis利用前缀匹配算法。
     * geohash编码和经纬度是可以相互转换的。

   * 删除地理位置信息：zrem key member

### 六.客户端

#### 一.客户端通信协议

1. 发送命令格式：

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E5%8D%8F%E8%AE%AE.jpg)

2. 返回结果格式

   * 状态回复：在RESP中的第一个字节为“+”
   * 错误回复：在RESP中的第一个字节为“-”
   * 整数回复：在RESP中的第一个字节为“：”
   * 字符串回复：在RESP中的第一个字节为“$”
   * 多条字符串回复：在RESP中的第一个字节为“*”

   注意：无论是字符串回复还是多条字符串回复，如果有nil值，那么会返回$-1。

#### 二.java客户端Jedis

####三.java客户端Lettuce 

#### 四.客户端管理

1. client list：列出与服务端连接的所有客户端。

   * id：客户端连接唯一标识，随着redis连接自增，重启重置为0.

   * addr：客户端连接ip。

   * fd：Socket文件描述符。fd=-1代表当前客户端不是外部客户端，而是redis内部伪装客户端。

   * name：客户端名。

   * 输入缓冲区：qbuf(总量)，qbuf-free(余量)。将客户端发送的命令临时保存，redis会从输入缓冲区拉取命令执行。**注意：没有配置来规定每个缓冲区大小，会根据输入内容大小不同来动态调整，仅要求每个客户端缓冲区大小不能超过1G，超过客户端将关闭。**

     使用不当产生的问题：

     * 某个客户端输入缓冲区超过1G，客户端关闭。
     * 输入缓冲区不受最大内存控制，输入缓冲区使用超过了最大内存，会产生数据丢失、键值淘汰、OOM等情况。

     造成缓冲区过大原因：

     * redis的处理速度跟不上输入缓冲区的输入速度，并且每次进入输入缓冲区的命令包含了大量bigkey。
     * redis发生的阻塞。

     监控：

     * client list：精确分析。但执行慢，频繁执行可能阻塞
     * info clients：分析过程简单，执行速度快。

   * 输出缓冲区：obl、oll、omem。保存命令执行结果返回给客户端。

     * 容量可以通过client-output-buffer-limit设置。
     * 按照客户端不同分为三种：普通客户端、发布订阅客户端、slave客户端。
     * 输入缓冲区不受最大内存控制，输入缓冲区使用超过了最大内存，会产生数据丢失、键值淘汰、OOM等情况。
     * 由固定缓冲区(16k)和动态缓冲区组成。固定缓冲区返回比较小的结果。固定缓冲区使用的是字节数组，动态缓冲区使用的是列表。当固定缓冲区满了之后，会存放动态缓冲区。obl代表固定缓冲区长度。oll代表动态缓冲区列表长度

   * 客户端存活状态：age、idle分别表示当前客户端已连接的时间、最近一次空闲时间。

   * 客户端限制：maxclients(最大客户端连接数，默认10000)、timeout(最大空闲时间，默认为0，不检测)

   * 客户端类型：flag。flag=S代表是slave客户端，flag=N代表普通客户端，flag=O代表客户端正在执行monitor命令。

   ​    ![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%B1%BB%E5%9E%8B.jpg)

   ![](https://github.com/XwDai/learn/raw/master/notes/image/redisclientlist%E5%91%BD%E4%BB%A4%E7%BB%93%E6%9E%9C%E5%B1%9E%E6%80%A7%E4%B8%80.jpg)

   ![](https://github.com/XwDai/learn/raw/master/notes/image/redisclientlist%E5%91%BD%E4%BB%A4%E7%BB%93%E6%9E%9C%E5%B1%9E%E6%80%A7%E4%BA%8C.jpg)

2. client setName、client getName

   > client setName：给客户端设置名字，标识客户端来源。
   >
   > client getName：获取客户端名字。

3. client kill

   > client kill ip:port ：杀死指定ip:port的客户端。可用来杀死timeout=0的长期idle的客户端。

4. client pause

   > client pause timeout（毫秒）：阻塞客户端timeout毫秒数。在此期间，客户端被阻塞。

   使用场景：

   1. 只对普通和发布订阅客户端有效。对于主从复制(从节点内部伪装的客户端)无效。也就是此期间主从复制是正常运行的。所以此命令可以用来让主从复制保持一致。
   2. 可以用一种可控方式将客户端连接从一个redis节点切换到另一个redis节点。

   **注意：生产环境暂停客户端成本非常高**。

5. monitor

   > 用于监控redis正在执行的命令。

   **注意：每个客户端都有自己的输出缓冲区，既然monitor能监控到所有命令，一旦redis并发量过大，monitor客户端输出缓冲区会暴涨，瞬间占用大量内存。**

6. 客户端其他配置

   1. tcp-keeplive：检测tcp连接活性周期，默认为0，不检测，设置可防止大量死链接占用系统资源。
   2. tcp-backlog：tcp三次握手之后，会将接收的连接放入队列中，tcp-backlog就是队列的大小。redis默认值是511。通常不需要调整，但会受到操作系统的影响。如果比操作系统的设置大，redis启动时会有警告提示。

7. 客户端统计：

   1. info clients
   2. info stats

####五.客户端常见异常

1. could not get a resource from the pool...timeout waiting for idle object。

   > 在maxWaitMillis时间内仍然无法获取到jedis对象。

   当设置了blockWhenExhausted=false，那么发现池中没有资源之后，会立即抛异常：pool exhausted。

   造成原因：

   * 连接池设置过小。
   * 没有正确使用连接池。比如使用之后没有释放。
   * 存在慢查询操作。归还速度慢。
   * 客户端正常，服务器由于一些原因造成客户端命令执行过程的阻塞。

2. 客户端读写超时。read time out

   * 读写超时设置过短。
   * 命令本身比较慢。
   * 客户端与服务端网络不正常。
   * redis自身发生阻塞。

3. 客户端连接超时。connect time out。

   * 连接超时设置过短。
   * redis发生阻塞。造成tcp-backlog已满。
   * 客户端与服务端网络不正常。

4. 客户端缓冲区异常。Unexpected end of stream。

   * 输出缓冲区满。
   * 长时间闲置连接被服务端断开。连接已关闭。
   * 不正常并发读写。jedis的多线程并发问题，lettuce客户端解决了这个问题。

5. Lua脚本正在执行。busy redis is busy running a script .you can only call script kill or shutdown nosave.

   * 脚本执行耗时，阻塞redis。
   * 脚本本身有问题。

6. redis正在加载持久化文件。LOADING Redis is loading the dataset in memory.

   * jedis调用redis时，如果redis正在加载持久化文件，会报此错。

7. redis使用内存超过maxmemory。OOM command not allowed when use memory > 'maxmemory'

8. 客户端连接数过大。ERR max number of clients reached 

   无法通过redis命令修复。

   从两个方面着手解决：

   * maxClients参数不是很小的话，应用客户端基本不会超过，通常是客户端使用不当。分布式环境中可先下线部分应用节点。再查找程序bug或调整maxClients进行修复。
   * 如果客户端无法处理，当redis为高可用模式(哨兵和集群)，可以考虑将当前redis做故障转移。

   


#### 六.客户端案例分析

1. redis 内存陡增。

   > 现象：
   >
   > > 服务端：主节点内存陡增，几乎满了，从节点内存没有变化。
   >
   > > 客户端：客户端OOM。
   >
   > 原因分析：
   >
   > 1. 确实有大量写入，但是主从复制出现问题。
   >    * 查询主从节点键个数：dbsize ，一样说明复制是正确的。
   > 2. 其他原因：排查是否由于客户端缓冲去造成主节点内存陡增。
   >    * info clients。查看缓冲区是否正常。
   >    * client list。找出omem不正常的连接。
   >
   > 处理方法：
   >
   > 1. 静止monitor。
   > 2. 限制输出缓冲区大小。
   > 3. 使用专用的运维工具，报警。

2. 客户端周期性超时。

   > 现象：
   >
   > > 客户端：大量超时。超时为周期性的。connect time out。
   >
   > > 服务端：没有明显异常，只是有一些慢查询操作。
   >
   > 分析：
   >
   > 1. 网络原因。
   > 2. redis本身。
   > 3. 客户端。
   >
   > 处理方法：
   >
   > 1. 监控慢查询。
   > 2. 避免使用慢查询命令。

### 七.持久化

#### 一.RDB

> 把当前进程数据生成快照保存到硬盘的过程。分为手动触发和自动触发。

1. 触发机制

   * 手动触发：
     * save(废弃)：阻塞redis服务器，直到RDB完成。内存比较大的实例阻塞时间长，线上不建议使用。
     * bgsave：fork子进程持久化。完成后自动结束。阻塞只发生在fork阶段，一般时间很短。
   * 自动触发：
     * 使用save相关配置，如" save m n"，m表示m秒内数据存在修改时，自动触发bgsave。
     * 从节点执行全量复制，主节点自动执行bgsave生成RDB文件发送给从节点。
     * 执行debug reload 重新加载redis，也会触发save操作。
     * 默认情况下执行shutdown，如果没有开启AOF持久化则执行bgsave。

   ![](https://github.com/XwDai/learn/raw/master/notes/image/redisBgsave.jpg)

   通过info Persistence查看rdb_*相关选项。

2. RDB文件处理。

   1. 保存：保存在dir配置指定目录下，文件名通过dbfilename指定。

   2. 压缩：默认采用LZF算法对生成RDB文件压缩，压缩后的文件远远小于内存大小，默认开启。

      > 提示：虽然压缩会消耗CPU，但可大幅降低文件体积。方便保存或通过网络发送。

   3. 检验：如果redis加载损坏的RDB文件时拒绝启动。Short read or OOM loading DB. Unrecoverable error ,aborting now.

      > 可以使用redis-check-dump工具检测RDB文件并获取对应的错误报告。

3. RDB优缺点。

   优点：

   * RDB是一个紧凑压缩的二进制文件(因此读取RDB回复速度更快)，代表redis某个时间点上的数据快照。非常适合用于备份，全量复制等场景。
   * redis加载RDB回复数据远远快于AOF。

   缺点：

   * 没办法做到实时持久化/秒级持久化。fork属于重量级操作，频繁执行成本过高。
   * 使用特定二进制格式保存，redis版本演进过程中有多个格式的RDB版本，存在兼容性问题。

#### 二.AOF

> 默认不开启，开启需配置：appendonly yes。
>
> AOF文件名通过appendfilename设置，默认文件名appendonly.aof。
>
> 保存路径与RDB一致，通过dir指定。
>
> AOF工作流程：命令写入(append)、文件同步(sync)、文件重写(rewrite)、重启加载(load)。
>
> ![](https://github.com/XwDai/learn/raw/master/notes/image/redisAof%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpg)

1. 命令写入

   > AOF命令写入的格式直接是文本协议格式。
   >
   > 为什么要这样？
   >
   > 1. 文本协议具有很好兼容性。
   > 2. 开启AOF后，所有写入命令都包含追加操作，直接采用协议操作，避免二次处理开销。
   > 3. 文本协议具有可读性，方便直接修改和处理。
   >
   > 为什么把命令追加到aof_buf？
   >
   > 1. redis使用单线程响应命令，如果每次写都直接追加到硬盘，那么性能完全取决于当前硬盘负载。
   > 2. 可提供多种缓冲区同步硬盘策略。

2. 文件同步

   redis提供了多种AOF缓冲区同步策略，由参数appendfsync控制。

   ![](https://github.com/XwDai/learn/raw/master/notes/image/redisAof%E7%BC%93%E5%86%B2%E5%8C%BA%E5%90%8C%E6%AD%A5%E6%96%87%E4%BB%B6%E7%AD%96%E7%95%A5.jpg)

   * write操作系统会触发延迟写机制。linux提供页缓冲区提高硬盘IO性能，write写入缓冲区后直接返回。同步硬盘依赖系统调度机制。如果系统宕机，数据将丢失。
   * fsync针对单个文件操作，做强制硬盘同步，fsync阻塞到写入硬盘完成后返回。保证数据持久化。

   1. always：每次都同步AOF文件，TPS低，性能不高，不建议配置。
   2. no：操作系统调度周期不可控，性能提升了，但是数据安全性不高。
   3. everysec：建议的同步策略。默认配置。兼顾性能和数据安全。理论上只有在系统宕机情况下丢失1s的数据。

3. 重写机制

   > 随着命令不断写入，文件会越来越大。redis引入AOF重写机制压缩文件体积。
   >
   > AOF重写是把redis进程内的数据转化为写命令同步到新的AOF文件的过程。
   >
   > 重写文件为何可以变小？
   >
   > 1. 进程内超时的数据不再写入文件。
   > 2. 旧的AOF文件含有无效命令。像del key等这些命令就没用了，我是直接用的内存数据生成的命令，已经剔除了这个key。
   > 3. 多条命令可以合并为一个。如，lpush list a,lpush list b,lpush list c可以转化为lpush list a b c。为了防止单条命令过大造成客户端缓冲区溢出，对于list、set、hash、zset等类型操作，以64个元素为界拆分成多条。
   >
   > 更小的AOF文件可以更快的被redis加载。

   1. 手动触发：直接调用bgrewriteaof。

   2. 自动触发：根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage确定触发时机。

      * auto-aof-rewrite-min-size：运行AOF重写时文件最小体积，默认64M。

      * auto-aof-rewrite-percentage：代表当前AOF文件空间aof_current_size和上一次重写AOF文件空间aof_base_size的比值。

        自动触发时机=aof_current_size>auto-aof-rewrite-min-size&&(aof_current_size-aof_base_size)/aof_base_size>=auto-aof-rewrite-percentage

      ![](https://github.com/XwDai/learn/raw/master/notes/image/redisAOF%E9%87%8D%E5%86%99%E8%BF%90%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)

4. 重启加载

   ![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E6%8C%81%E4%B9%85%E5%8C%96%E6%96%87%E4%BB%B6%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B.jpg)

5. 文件校验

   > 加载损坏的AOF文件是会拒绝启动。
   >
   > 对于错误格式的AOF文件，先进行备份，然后采用redis-check-aof --fix命令进行修复。修复后使用diff -u对比数据差异，找出丢失数据，有些可以人工修改补全。

   AOF文件可能存在结尾不完整的情况，比如机器突然掉电导致AOF尾部文件命令写入不全。redis为我们提供了aof-load-truncated配置来兼容这种情况，默认开启。

#### 三.问题定位与优化

1. fork操作。

   > fork创建子进程虽然不需要拷贝父进程的物理内存，但是会复制父进程的空间内存页表。因此fork操作耗时跟进程总内存量息息相关的。

   * fork耗时问题定位
     * 正常情况fork耗时应该是GB消耗20毫秒左右。
     * info stats统计中查latest_fork_usec指标获取最近一次fork操作耗时。单位微妙。
   * 如何改善fork操作耗时
     * 优先使用物理机或高效支持fork操作的虚拟化技术，避免使用Xen。
     * 控制redis实例最大可用内存，fork耗时跟内存量成正比。线上建议控制在10GB左右。
     * 合理配置linux内存分配策略，避免物理内存不足导致fork失败。
     * 降低fork操作频率。如适度放宽AOF触发时机，避免不必要的全量复制。

2. 子进程开销监控和优化

   1. CPU
      * 子进程负责把进程内的数据分批写入文件，这个属于CPU密集型操作，通常子进程对单核CPU利用率90%
      * redis属于CPU密集型服务，不要做绑定单核CPU操作。由于子进程非常消耗CPU，会和父进程产生单核资源竞争。
      * 不要和其他CPU密集型服务部署在一起。
      * 如果部署多个redis实例，尽量保证同一时刻只有一个子进程执行重写工作。
   2. 内存
      * 子进程通过fork操作产生，占用内存大小等同于父进程。理论上需要两倍的内存完成持久化操作，但linux有写时复制机制。父子进程会共享相同的物理内存页，当父进程处理写请求时会把要修改的页创建副本，而子进程在fork操作过程中会共享整个父进程内存快照。RDB和AOF类似，只是AOF需要AOF重写缓冲区(aof_rewrite_buf)，多了写内存消耗。
      * 部署多个实例时，尽量保证同一时刻只有一个子进程工作。
      * 避免大量写入时做子进程重写操作。这样导致父进程维护大量页副本，造成内存消耗。

      > fork阻塞跟内存量和系统有关
   3. 硬盘
      * 持久化势必造成硬盘写入压力。
      * 不要和其他硬盘高负载的服务部署在一起。
      * AOF重写时会消耗大量IO，可以配置no-appendfsync-on-rewrite，默认关闭。表示AOF重写期间不做fsync操作。
      * 固态硬盘
      * 单机多个实例时，配置不同实例分盘存储AOF文件。

      > AOF追加阻塞说明硬盘资源紧张

3. AOF追加阻塞

   > 开启AOF时，常用的同步策略是everysec。这种方式，redis会使用另外一条线程每秒执行fsync同步磁盘，当系统资源繁忙时，会造成redis主线程阻塞。
   >
   > 流程：
   >
   > 1. 主线程复制写入AOF缓冲区。
   > 2. AOF线程复制每秒执行一次同步刷盘。
   > 3. 主线程负责对比上次AOF同步时间。
   >    * 距离上次成功时间在2s内，主线程直接返回。
   >    * 超过2s，主线程会阻塞，知道同步操作完成，也就是距离上次成功时间在2s内。
   >
   > 问题：
   >
   > 1. everysec最多丢失2s数据。
   > 2. fsync缓慢，将导致主线程阻塞。
   >
   > 定位：
   >
   > 1. 日志
   > 2. 每次追加阻塞事件发生，info persistence 中的aof_delayed_fsync指标会累加。
   > 3. AOF同步最多运行2s延迟。延迟说明硬盘高负载，可以通过iotop定位消耗硬盘io资源进程。



#### 四.多实例部署

![](https://github.com/XwDai/learn/raw/master/notes/image/redisInfoPersistence%E5%BA%A6%E9%87%8F%E6%8C%87%E6%A0%87.jpg)

> 单机多实例部署，并且都开启AOF，多个子进程彼此会产生CPU和IO竞争，因此，需要采用一种措施把子进程工作隔离。

解决方案：我们可以基于上面指标，通过外部程序轮询控制AOF重写操作执行。

![](https://github.com/XwDai/learn/raw/master/notes/image/redis%E8%BD%AE%E8%AF%A2%E6%8E%A7%E5%88%B6AOF%E8%AF%BB%E5%86%99.jpg)



### 八.复制

#### 一.配置

1. 建立复制

   > 参与复制的redis实例划分为主节点和从节点。默认redis都是主节点。每个从节点只能有一个主节点，主节点可以有多个从节点。复制数据的流是单向的，只能从主节点复制到从节点。
   >
   > 配置复制的三种方式：
   >
   > * 配置文件：slaveof {masterHost} {masterPort}
   > * 在redis-server启动命令中加入--slaveof {masterHost} {masterPort}
   > * 直接使用命令slaveof {masterHost} {masterPort}
   >
   > info replication 查看复制相关状态。

2. 断开复制

   >slaveof no one
   >
   >流程：
   >
   >1. 断开与主节点的复制关系
   >2. 从节点晋升为主节点

   > slaveof {newMasterHost} {newMasterPort}
   >
   > 切主操作：把当前从节点对应的主节点的复制切换到另一个主节点。
   >
   > 流程：
   >
   > 1. 断开与旧主节点复制关系
   > 2. 与新主节点建立复制关系
   > 3. **删除从节点当前所有数据**   //特别注意这点
   > 4. 对新主节点进行复制操作

3. 安全性

   对于数据比较重要的节点，主节点会通过设置requirepass参数进行密码验证，这时所有客户端访问必须使用auth命令实行校验。主从复制连接是通过一个特殊标识的客户端完成的，因此需要配置从节点的masterauth参数与主节点密码保持一致，这样从节点才可以正确连接到主节点并发起复制流程。

4. 只读

   默认情况，从节点使用slave-read-only=yes配置为只读。从节点的修改主节点无法感知，所以线上建议不要修改从节点的只读模式。

5. 传输延迟

   主从节点一般部署到不同机子上，复制时就需要考虑网络延迟的问题。

   redis提供了repl-disable-tcp-nodelay参数控制是否关闭tcp_nodelay，默认关闭。

   * 关闭时，主节点产生的命令数据无论大小都会及时的发送给从节点，主从延迟会变小，但增加了网络带宽消耗。适用于主从之间网络环境良好的场景。如同机房。
   * 开启时，主节点会合并比较小的tcp数据包从而节省带宽，默认发送时间间隔取决于linux内核，一般默认40毫秒。这种节省了带宽但增大主从延迟，适用于网络环境复杂或带宽紧张的场景，如跨机房。

   > 建议：
   >
   > 1. 要求低延迟，同机架同机房部署，关闭repl-disable-tcp-nodelay
   > 2. 考虑容灾，跨机房并开启repl-disable-tcp-nodelay

   

#### 二.拓扑

redis的复制可支持单层或多层复制关系。

1. 一主一从

   * 用于主节点出现宕机时从节点提供故障转移支持。

   * 当应用写命令并发量较高且需求持久化时，可以在从节点上开启AOF，保证数据安全性同时避免持久化对主节点的性能干扰。

     注意：主节点关闭了持久化功能，如果主节点脱机要避免自动重启操作。因为没开启持久化自动重启后数据会清空，这时从节点如果继续复制主节点会导致从节点数据也会被清空。安全做法是在从节点上执行slaveof no one 断开与主节点的复制关系，再重启主节点。

2. 一主多从

   使用多个从节点实现读写分离。

   * 对于读占比大的场景，可以把读命令发送到从节点分担主节点压力。同时一些耗时的读命令，可以在一台从节点运行，防止慢查询对主节点造成阻塞。
   * 并发较高的场景，多个从节点会导致主节点写命令的多次发送从而过度消耗网络带宽，同时也加重了主节点的负载影响服务稳定性。

3. 树状主从结构

   从节点不但可以复制主节点数据，同时可以作为其它从节点的主节点继续向下层复制。同过引入复制中间层，可以有效降低主节点负载和需要传输给从节点的数据量。

   

#### 三.原理

1. 复制过程

   从节点执行slaveof之后，复制过程便开始运作。

   

#### 四.开发与运维中的问题



### 九.redis的噩梦：阻塞

#### 一.发现阻塞

#### 二.内在原因

#### 三.外在原因



### 十.理解内存

#### 一.内存消耗

#### 二.内存管理

#### 三.内存优化



### 十一.哨兵

#### 一.基本概念

#### 二.安装部署

#### 三.API

#### 四.客户端连接

#### 五.实现原理

#### 六.开发与运维中的问题



### 十二.集群

#### 一.数据分布

#### 二.搭建集群

#### 三.节点通信

#### 四.集群伸缩

#### 五.请求路由

#### 六.故障转移

#### 七.集群运维



### 十三.缓存设计

#### 一.缓存收益与成本

#### 二.缓存更新策略

#### 三.缓存粒度控制

#### 四.穿透优化

#### 五.无底洞优化

#### 六.雪崩优化

#### 七.热key重建优化



### 十三.开发运维陷阱

#### 一.linux配置优化

#### 二.flushall/flushdb误操作

#### 三.安全的redis

#### 四.处理bigkey

#### 五.寻找热点key







​       



