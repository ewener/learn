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

> 单线程架构

