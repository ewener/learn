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