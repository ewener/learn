[TOC]

## 一.死锁

**问题**：lock wait timeout exceeded; try restarting transaction

**原因**： 今天线上环境，突然出现一个问题，追踪原因是数据库中的一条语句报错，错误内容： **lock wait timeout exceeded; try restarting transaction** ，  执行update table set status = 1 where id = 10002;是可以的。 而执行update table set status = 1 where id = 10001;这条语句执行失败。  错误提示的意思，很明显，是因为这条语句被锁住了。所以释放这个锁。 现在我们要查test库中使用情况，我们可以到information_schema中查询 

**解释**：information_schema这张数据表保存了MySQL服务器所有数据库的信息。如数据库名，数据库的表，表栏的数据类型与访问权限等。再简单点，这台MySQL服务器上，到底有哪些数据库、各个数据库有哪些表，每张表的字段类型是什么，各个数据库要什么权限才能访问，等等信息都保存在information_schema表里面。

我们可以用下面三张表来查原因： 

​       innodb_trx ## 当前运行的所有事务 

​       innodb_locks ## 当前出现的锁 

​       innodb_lock_waits ## 锁等待的对应关系

如果数据库中有锁的话，我们需要杀掉这个锁，执行 kill 线程id号。 

其他记录状态为“RUNNING” 即正在执行的事务，并没有锁，不需要关注

我们可以进一步了解一下 那三张表的表结构：

* desc information_schema.innodb_locks;

![img](D:/mkdown/qq119604AC6A460688138C5CD0E2484784/916ef2b73556485d9fafbe58da4d2d50/clipboard.png)

* desc information_schema.innodb_lock_waits

![img](D:/mkdown/qq119604AC6A460688138C5CD0E2484784/2fc9cb8c3c5e4a2a93eea4d7aca3211c/clipboard.png)

* desc information_schema.innodb_trx ;

![img](D:/mkdown/qq119604AC6A460688138C5CD0E2484784/4316e2a170c1474e85cc19053fbaf7f1/clipboard.png)

## 二.常用语句

### 1. 查看mysql版本 

* SHOW VARIABLES WHERE Variable_name = 'version'; 

## 三.名词解析

### 1.utf8和utf8mb4区别 

MySQL在5.5.3之后增加了这个utf8mb4的编码，mb4就是most bytes 4的意思，专门用来兼容四字节的unicode。好在utf8mb4是utf8的超集，除了将编码改为utf8mb4外不需要做其他转换。当然，为了节省空间，一般情况下使用utf8也就够了。

   那上面说了既然utf8能够存下大部分中文汉字,那为什么还要使用utf8mb4呢? 原来mysql支持的 utf8 编码最大字符长度为 3 字节，如果遇到 4 字节的宽字符就会插入异常了。三个字节的 UTF-8 最大能编码的 Unicode 字符是 0xffff，也就是 Unicode 中的基本多文种平面(BMP)。也就是说，任何不在基本多文本平面的 Unicode字符，都无法使用 Mysql 的 utf8 字符集存储。包括 Emoji 表情(Emoji 是一种特殊的 Unicode 编码，常见于 ios 和 android 手机上)，和很多不常用的汉字，以及任何新增的 Unicode 字符等等。

   最初的 UTF-8 格式使用一至六个字节，最大能编码 31 位字符。最新的 UTF-8 规范只使用一到四个字节，最大能编码21位，正好能够表示所有的 17个 Unicode 平面。

   utf8 是 Mysql 中的一种字符集，只支持最长三个字节的 UTF-8字符，也就是 Unicode 中的基本多文本平面。

   Mysql 中的 utf8 为什么只支持持最长三个字节的 UTF-8字符呢？我想了一下，可能是因为 Mysql 刚开始开发那会，Unicode 还没有辅助平面这一说呢。那时候，Unicode 委员会还做着 “65535 个字符足够全世界用了”的美梦。Mysql 中的字符串长度算的是字符数而非字节数，对于 CHAR 数据类型来说，需要为字符串保留足够的长。当使用 utf8 字符集时，需要保留的长度就是 utf8 最长字符长度乘以字符串长度，所以这里理所当然的限制了 utf8 最大长度为 3，比如 CHAR(100)  Mysql 会保留 300字节长度。至于后续的版本为什么不对 4 字节长度的 UTF-8 字符提供支持，我想一个是为了向后兼容性的考虑，还有就是基本多文种平面之外的字符确实很少用到。

   要在 Mysql 中保存 4 字节长度的 UTF-8 字符，需要使用 utf8mb4 字符集，但只有 5.5.3 版本以后的才支持(查看版本： select version();)。我觉得，为了获取更好的兼容性，应该总是使用 utf8mb4 而非 utf8.  对于 CHAR 类型数据，utf8mb4 会多消耗一些空间，根据 Mysql 官方建议，使用 VARCHAR  替代 CHAR。

### 2.执行计划

**使用explain关键字可以模拟优化器执行SQL查询语句，从而知道MySQL是如何处理你的SQL语句的，分析你的查询语句或是表结构的性能瓶颈。**

####1.explain执行计划包含的信息

![](https://github.com/XwDai/learn/raw/master/notes/image/%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E5%88%97.png)

**其中最重要的字段为：id、type、key、rows、Extra**

#### 2.各字段详解

* **id**

  select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序 

  三种情况： 

  * **id相同**：执行顺序由上至下 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/id%E7%9B%B8%E5%90%8C.png)

  * **id不同**：如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/id%E4%B8%8D%E5%90%8C.png)

  * **id相同又不同（两种情况同时存在）**：id如果相同，可以认为是一组，从上往下顺序执行；在所有组中，id值越大，优先级越高，越先执行 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/id%E7%9B%B8%E5%90%8C%E5%8F%88%E4%B8%8D%E5%90%8C.png)

* **select_type**

  查询的类型，主要是用于区分普通查询、联合查询、子查询等复杂的查询

  * **SIMPLE**：简单的select查询，查询中不包含子查询或者union 

  * **PRIMARY**：查询中包含任何复杂的子部分，最外层查询则被标记为primary 

  * **SUBQUERY**：在select 或 where列表中包含了子查询 

  * **DERIVED**：在from列表中包含的子查询被标记为derived（衍生），mysql或递归执行这些子查询，把结果放在零时表里 

  * **UNION**：若第二个select出现在union之后，则被标记为union；若union包含在from子句的子查询中，外层select将被标记为derived 

  * **UNION RESULT**：从union表获取结果的select 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/unionresult.png)

* **type**

  访问类型，sql查询优化中一个很重要的指标，结果值从好到坏依次是：system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL，一般来说，好的sql查询至少达到range级别，最好能达到ref。

  * **system**：表只有一行记录（等于系统表），这是const类型的特例，平时不会出现，可以忽略不计

  * **const**：表示通过索引一次就找到了，const用于比较primary key 或者 unique索引。因为只需匹配一行数据，所有很快。如果将主键置于where列表中，mysql就能将该查询转换为一个const 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/const_system.png)

  * **eq_ref**：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键 或 唯一索引扫描。 

    注意：ALL全表扫描的表记录最少的表如t1表

    ![](https://github.com/XwDai/learn/raw/master/notes/image/eq_ref.png)

  * **ref**：非唯一性索引扫描，返回匹配某个单独值的所有行。本质是也是一种索引访问，它返回所有匹配某个单独值的行，然而他可能会找到多个符合条件的行，所以它应该属于查找和扫描的混合体 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/ref.png)

  * **range**：只检索给定范围的行，使用一个索引来选择行。key列显示使用了那个索引。一般就是在where语句中出现了bettween、<、>、in等的查询。这种索引列上的范围扫描比全索引扫描要好。只需要开始于某个点，结束于另一个点，不用扫描全部索引 ![](https://github.com/XwDai/learn/raw/master/notes/image/range.png)

  * **index**：Full Index Scan，index与ALL区别为index类型只遍历索引树。这通常为ALL块，应为索引文件通常比数据文件小。（Index与ALL虽然都是读全表，但index是从索引中读取，而ALL是从硬盘读取） 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/index.png)

  * **ALL**：Full Table Scan，遍历全表以找到匹配的行 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/all.png)

* possible_keys

  查询涉及到的字段上存在索引，则该索引将被列出，但不一定被查询实际使用

* key

  实际使用的索引，如果为NULL，则没有使用索引。 查询中如果使用了覆盖索引，则该索引仅出现在key列表中 

  ![](https://github.com/XwDai/learn/raw/master/notes/image/key1.png)

  ![](https://github.com/XwDai/learn/raw/master/notes/image/key2.png)

* key_len

  表示索引中使用的字节数，查询中使用的索引的长度（最大可能长度），并非实际使用长度，理论上长度越短越好。key_len是根据表定义计算而得的，不是通过表内检索出的

* ref

  显示索引的那一列被使用了，如果可能，是一个常量const。

* rows

  根据表统计信息及索引选用情况，大致估算出找到所需的记录所需要读取的行数

* Extra

  不适合在其他字段中显示，但是十分重要的额外信息

  * **Using filesort** ： mysql对数据使用一个外部的索引排序，而不是按照表内的索引进行排序读取。也就是说mysql无法利用索引完成的排序操作成为“文件排序” 由于索引是先按email排序、再按address排序，所以查询时如果直接按address排序，索引就不能满足要求了，mysql内部必须再实现一次“文件排序”

    ![](https://github.com/XwDai/learn/raw/master/notes/image/filesort.png)

  * **Using temporary**： 使用临时表保存中间结果，也就是说mysql在对查询结果排序时使用了临时表，常见于order by 和 group by 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/temporary.png)

  * **Using index**： 表示相应的select操作中使用了**覆盖索引**（Covering Index），避免了访问表的数据行，效率高 如果同时出现Using where，表明索引被用来执行索引键值的查找（参考上图） 如果没用同时出现Using where，表明索引用来读取数据而非执行查找动作 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/usingindex.png)

  * 覆盖索引**（Covering Index）：也叫索引覆盖。就是select列表中的字段，只用从索引中就能获取，不必根据索引再次读取数据文件，换句话说**查询列要被所建的索引覆盖**。 

    注意： 

    a、如需使用覆盖索引，select列表中的字段只取出需要的列，不要使用select * 

    b、如果将所有字段都建索引会导致索引文件过大，反而降低crud性能

  * Using where ： 使用了where过滤

  * Using join buffer ： 使用了链接缓存

  * Impossible WHERE： where子句的值总是false，不能用来获取任何元祖 

    ![](https://github.com/XwDai/learn/raw/master/notes/image/impossiblewhere.png)

  * select tables optimized away： 在没有group by子句的情况下，基于索引优化MIN/MAX操作或者对于MyISAM存储引擎优化COUNT（*）操作，不必等到执行阶段在进行计算，查询执行计划生成的阶段即可完成优化

  * distinct： 优化distinct操作，在找到第一个匹配的元祖后即停止找同样值得动作

#### 3.综合Case

![](https://github.com/XwDai/learn/raw/master/notes/image/综合case.png)

**执行顺序** 

1. （id = 4）、【select id, name from t2】：select_type 为union，说明id=4的select是union里面的第二个select。
2. （id = 3）、【select id, name from t1 where address = ‘11’】：因为是在from语句中包含的子查询所以被标记为DERIVED（衍生），where address = ‘11’ 通过复合索引idx_name_email_address就能检索到，所以type为index。
3. （id = 2）、【select id from t3】：因为是在select中包含的子查询所以被标记为SUBQUERY。
4. （id = 1）、【select d1.name, … d2 from … d1】：select_type为PRIMARY表示该查询为最外层查询，table列被标记为 “derived3”表示查询结果来自于一个衍生表（id = 3 的select结果）。
5. （id = NULL）、【 … union … 】：代表从union的临时表中读取行的阶段，table列的 “union 1, 4”表示用id=1 和 id=4 的select结果进行union操作。