# MySQL从入门到精通学习笔记

## 1. SQL语言分类

SQL语言全称为结构化查询语言，分为以下4个子类：DQMC。

1. 数据定义语言：简称DDL（Data Definition Language），用来定义数据库对象：数据库、表、列等。关键字：create、alter、drop等
2. 数据查询语言：简称DQL（Data Query Language），用来查询数据库中表的记录。关键字：select、from、where等
3. 数据操作语言：简称DML（Data Manipulation Language），用来对数据库中表的记录进行更新。关键字：insert、delete、update等
4. 数据控制语言：简称DCL（Data Control Language），用来定义数据库的访问权限和安全级别，及创建用户，关键字：grant等

## 2. SQL解析顺序（重点）

**问题1**：SQL语句的解析顺序是什么样的？

示例SQL：

```sql
SELECT DISTINCT
    < select_list >
FROM
    < left_table > < join_type >
JOIN < right_table > ON < join_condition >
WHERE
    < where_condition >
GROUP BY
    < group_by_list >
HAVING
    < having_condition >
ORDER BY
    < order_by_condition >
LIMIT < limit_number >
```

解析顺序：行过滤——列过滤——排序——LIMIT（MySQL特有）

```sql
-- 行过滤
FROM < left_table > 
ON < join_condition >
< join_type > JOIN < right_table > -- 多表连接时循环执行，构建表与表的笛卡尔积
WHERE < where_condition > -- 从左往右循环执行每个条件
GROUP BY < group_by_list >
HAVING < having_condition >
-- 列过滤
SELECT DISTINCT < select_list >
-- 排序
ORDER BY < order_by_condition >
-- MySQL特有
LIMIT < limit_number >
```

## 3. MySQL架构

### 3.1 MySQL文件结构

#### 日志文件

MySQL包含4类日志文件：错误日志、二进制日志（bin log）、通用查询日志和慢查询日志。错误日志记录了所有发生过的错误信息，可用于定位问题。

```shell
# 可以直接定义为文件路径，也可以为ON|OFF 
log_error=/var/log/mysqld.log 
# 是否将警告信息也定义至错误日志中
log_warings=1
```

二进制日志bin log记录数据库所有DDL（直接记录）和DML（必须事务提交），实现主从复制、数据备份和数据恢复。

```shell
log-bin=mysql-bin
```

通用查询日志记录用户所有操作，只用于调试数据库，平时不开启，否则会产生大量磁盘IO。慢查询日志默认也是关闭的，它记录执行时间超过long_query_time秒的所有查询，用于收集查询时间比较长的SQL语句。

#### 数据文件

InnoDB存储引擎产生3种数据文件：

1. frm文件：表结构的定义信息
2. ibd文件：使用独享表空间存储表数据和索引信息
3. ibdata文件：使用共享表空间存储表数据和索引信息

MyISAM存储引擎产生3种数据文件：

1. frm文件：表结构的定义信息
2. myi文件：表数据文件中任何索引的数据树
3. myd文件：表数据信息

### 3.2 MySQL逻辑架构

![](/Users/likejun/javaee-architect/7_MySQL从入门到精通/assets/mysql逻辑架构.png)

#### MySQL Server层（重点）

Server层涉及到几个对象：连接器、查询缓存、分析器、优化器、执行器和存储引擎。

**问题2**：SQL语句的执行流程是什么样的？

1. 客户端通过连接器连接到MySQL服务器，并向MySQL发送SQL语句
2. 服务器先在查询缓存中查找，如果缓存命中了就直接返回
3. 否则分析器就对SQL语句进行词法分析、语法分析和预处理，并生成抽象语法树
4. 优化器再对生成的抽象语法树进行优化，即选择最优执行计划
5. 执行器进行权限验证，验证通过后调用存储引擎提供的接口执行语句，有索引则直接查询索引，没有索引则全表扫描并过滤。

#### InnoDB存储引擎（重点）

**问题3**：MySQL有哪些常用的可插拔存储引擎？

1. InnoDB：默认存储引擎，支持事务和行级锁
2. MyISAM：插入查询速度快，但不支持事务
3. Memory：所有数据存储在内存中，重启服务后就丢失
4. Falcon：据说是InnoDB的替代者
5. CSV：以CSV格式存储数据

![](/Users/likejun/javaee-architect/7_MySQL从入门到精通/assets/imdb架构图.png)

InnoDB分为两大部分：磁盘文件和内存结构。磁盘文件部分包含redo log重做日志文件、系统表空间和用户表空间。当InnoDB数据文件发生错误时重做日志文件可以恢复数据。重做日志是两个文件循环写入的，可以通过配置文件设置日志文件大小。

系统表空间是共享的，它存储InnoDB数据字典、double write buffer、change buffer和undo logs。用户表空间存储表的数据和索引信息。InnoDB逻辑存储结构：表空间、段（数据段、索引段、回滚段）、区、页、行。

内存结构包括Buffer Pool缓冲池、额外内存池、redo log buffer重做日志缓冲。Buffer Pool缓冲池包括数据页、索引页、更新缓冲、自适应哈希索引、锁信息和数据字典信息（数据、库对象和表对象的元信息集合）。

【TODO】InnoDB工作原理：内存数据落盘、CheckPoint检查点机制和Double Write的工作原理。

## 4. MySQL集群搭建与分库分表

### 4.1 主从复制集群

**问题4**：主从复制的原理是什么？

搭建MySQL主从复制服务器集群的原理是基于binlog实现的。binlog可用于恢复本机数据或实现主从复制。在一个主从复制的MySQL集群中，主服务器负责写，从服务器负责读。主服务器上的增删改操作会记录到主服务器的binlog日志中，一旦binlog刷写到磁盘，主服务器就会发消息通知从服务器，从服务器上的IO线程会向主服务器请求binlog并将其写入relaylog。之后从服务器上的SQL线程（5.7版后不止一个）会读取relaylog并在从服务器上进行重放，从而实现主从同步。

![](/Users/likejun/javaee-architect/7_MySQL从入门到精通/assets/主从复制.png)

#### binlog三种模式

1. statement：每一条会修改数据的sql都会记录到master的bin-log中。优点是解决了row level下的缺点，不需要记录每一行数据的变化， 减少binlog日志量，节约IO，提高性能；缺点是修改数据的时候如果使用了某些特定的函数或者功能的时候会出现复制问题
2. row：记录每一行数据被修改的形式。优点是可以不记录执行的SQL语句的上下文信息，仅仅只需要记录哪一条记录被修改了。不会出现某些特定情况下的存储过程、function以及trigger的调用和触发无法被正确复制的问题；缺点是会产生大量的日志内容，binlog日志的量会很大
3. mixed：前两种模式的结合，在mixed模式下，MySQL会根据执行的每一条具体的SQL语句来区分对待记录的日志形式，也就是在statement和row之间选一种

#### binlog刷盘策略：sync_binlog参数

* 0：存储引擎不进行binlog的刷新到磁盘，而由操作系统的文件系统控制缓存刷新
* 1：每提交一次事务，存储引擎调用文件系统的sync操作进行一次缓存的刷新，这种方式最安全，但性能较低
* n：当提交的日志组为n时，存储引擎调用文件系统的sync操作进行一次缓存的刷新

#### 主从同步延迟

产生主从延迟的原因是从服务器压力过大。一个服务器开放N个链接给客户端来，有可能产生大并发的更新操作，但是从服务器读取binlog的线程仅有一个，当某个SQL在从服务器上执行的时间稍长或者由于某个SQL要进行锁表就会导致主服务器的SQL大量积压，未被同步到从服务器里。这就导致了主从不一致。

主从同步延迟只能采取措施进行缓解，没有什么一招制敌的办法。因为本质上所有的SQL都必须在从服务器里面执行一遍。主服务器如果源源不断的有更新操作，那么一旦有延迟产生，加重的可能性就会越来越大。可以采取3种措施缓解：

* 主服务器负责更新操作，对安全性的要求比从服务器高。可以设置比如sync_binlog=1，innodb_flush_log_at_trx_commit = 1。从服务器不需要这么高的数据安全，可以设置sync_binlog为0或者关闭binlog。innodb_flushlog/innodb_flush_log_at_trx_commit也可以设置为0来从很大程度上提高效率。另外就是使用比主库更好的硬件设备作为从服务器
* 把从服务器完全作为备份使用，不提供查询，负载下来了执行relay log的效率自然就高了
* 增加从服务器，分散读的压力，从而降低服务器负载

MySQL可以查看从服务器状态，可以通过`show slave status`查看Seconds_Behind_Master参数的值来判断：

1. NULL：表示io_thread或是sql_thread有任何一个发生故障，也就是该线程的Running状态是No，而非 Yes
2. 0：该值为0表示主从复制状态正常

### 4.2 读写分离集群

MySQL的主从复制只保证主服务器对外提供服务，而从服务器一般只作为备份用，不对外提供服务。在一个高可用的MySQL集群中，采取的方式是主服务器提供读写服务，而多个从服务器则提供读服务。Atlas是奇虎360在MySQL-Proxy基础上提供的读写分离解决方案。

### 4.3 分库分表

单机的存储能力、连接数是有限的，随着业务规模的扩大会变成性能瓶颈。单表数据量在百万以内时还可以通过添加从库、优化索引提升性能。一旦数据量朝着千万以上趋势增长，再怎么优化数据库，很多操作性能仍然会严重下降。为了降低数据库的负担，提升数据库响应速度，缩短查询时间，需要进行分库分表。

**问题5**：数据分片的方案和规则

分库分表需要将大量数据分散到多个数据库中，即对数据进行分片（Sharding），使每个数据库中数据量小，响应速度快，提升数据库整体性能。分片方案大致可以分为：垂直（纵向）分片和水平（横向）分片两种。

![](/Users/likejun/javaee-architect/7_MySQL从入门到精通/assets/垂直分库.png)

**垂直分片**包括垂直分库和垂直分表。垂直分库按照业务分类进行划分，每个业务有独立数据库。垂直分表就像竖着把一块蛋糕切成了两个部分：一个部分是频繁访问的热点数据（核心表），一个部分是不频繁访问到的数据（扩展表）。MySQL是行式数据库，垂直分表可以加载更多内存到数据中。

* 优点：业务间解耦，不同业务的数据进行独立的维护、监控、扩展。在高并发场景下，一定程度上缓解了数据库的压力

* 缺点：提升了开发的复杂度，由于业务的隔离性，很多表无法直接访问，必须通过代码来聚合数据。分布式事务管理难度增加。数据库还是存在单表数据量过大的问题，需要配合水平分表进行进一步分片

**水平分片**就像把一块蛋糕横着切成了两个部分，每个部分都包含整个数据的一个子集。水平分片将一张大表切分成多个表结构相同，而每个表只占原表一部分数据，然后按不同的条件分散到多个数据库中。比如一张2000万行数据的order表水平切分成四个表：order_1、order_2、order_3和order_4，每张表数据500万。

水平分片又分为**库内分表**和**分库分表**。库内分表将表拆分，每个子表都在同一个数据库实例中，解决了单一表数据量过大的问题，没有将拆分后的表分布到不同节点的库上，还在竞争同一个物理机的CPU、内存和网络IO等资源。分库分表则将切分出来的子表分散到不同的数据库中，从而使得单个表的数据量变小，达到分布式的效果。

* 优点：解决高并发时单库数据量过大的问题，提升系统稳定性和负载能力，业务系统改造的工作量不是很大

* 缺点：跨分片的事务一致性难以保证，跨库的连表查询性能较差，扩容的难度和维护量较大

分库分表后，数据在不同数据库、不同表中的分布策略有两种：（1）根据取值范围分布；（2）根据散列分布。

根据取值范围，按照时间区间或ID区间来切分。假如我们切分的是用户表，可以定义每个库的User表里只存10000条数据，第一个库userId从0~9999，第二个库10000 ~ 19999，第三个库20000 ~ 29999......以此类推。

* 优点：单表数据量是可控的。水平扩展简单只需增加节点即可，无需对其他分片的数据进行迁移。能快速定位要查询的数据在哪个库

* 缺点：连续分片可能存在数据热点。如果按时间字段分片，有些分片存储最近时间段内的数据，可能会被频繁的读写，而有些分片存储的历史数据，则很少被查询

hash取模的切分方式比较常见，还拿User表举例，对数据库从0到N-1进行编号，对User表中userId字段进行取模，得到余数i ，i=0存第一个库，i=1存第二个库，i=2存第三个库....以此类推。这样同一个用户的数据都会存在同一个库里，用userId作为条件查询就很好定位了。

* 优点：数据分片相对比较均匀，不易出现某个库并发访问的问题
* 缺点：当某一台机器宕机，本应该落在该数据库的请求就无法得到正确的处理，这时宕掉的实例会被踢出集群，此时算法变成mod N-1，用户信息可能就不再在同一个库中

ShardingSphere是目前最好的分库分表工具。

## 5. MySQL性能优化

### 5.1 性能优化思路总览

使用慢查询日志功能，定位所有查询时间比较长的SQL语句 

EXPLAIN查看有问题的SQL的执行计划

针对查询慢的SQL语句进行优化

使用show profile[s]查看有问题的SQL的性能使用情况

操作系统调参优化

升级服务器硬件

### 5.2 慢查询日志

MySQL数据库有一个**慢查询日志**功能，用来记录查询时间超过某个设定值的SQL语句，可以帮助我们快速定位问题的症结所在。查询时间多长才算慢？每个项目或业务都有不同的要求。MySQL的慢查询日志功能默认关闭，需要手动开启，相关参数说明：

1. slow_query_log：是否开启慢查询日志，1（ON）为开启，0（OFF）为关闭。
2. log-slow-queries：5.6以下版本MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log
3. slow-query-log-file：5.6及以上版本MySQL数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件host_name-slow.log
4. long_query_time：慢查询阈值，当查询时间多于设定的阈值时，记录日志，单位为秒

了解慢查询日志格式，能读懂日志要表达的意思。mysqldumpslow是MySQL自带的慢查询日志工具。可以使用该工具搜索慢查询日志中的SQL语句。针对服务器集群，可以把所有的日志收集到一个文件夹中，然后用该工具展开分析。

### 5.3 查看SELECT语句执行计划

MySQL提供一个EXPLAIN命令，它可以对SELECT语句的执行计划进行分析，输出SELECT执行的详细信息，供开发人员针对性优化。可以通过分析SQL语句的执行计划来分析SQL语句有没有使用索引？有没有做全表扫描？可以深入了解MySQL的基于开销的优化器，还可以获得很多可能被优化器考虑到的访问策略的细节，以及当运行SQL语句时哪种策略预计会被优化器采用。在SELECT语句前加上EXPLAIN就可以查看执行计划了。各列的含义如下：

* id：SELECT查询标识符。每个 SELECT 都会自动分配一个唯一的标识符

* select_type（重点）：SELECT查询的类型

* table：查询的是哪个表

* partitions：匹配的分区

* type（重点）：join类型

* possible_keys：此次查询中可能选用的索引

* key：此次查询中确切使用到的索引.

* ref：哪个字段或常数与key一起被使用

* rows：显示此查询一共扫描了多少行，这个是一个估计值

* filtered：表示此查询条件所过滤的数据的百分比

* extra（重点）：额外的信息

#### id

每个单位查询的SELECT语句都会自动分配的一个唯一标识符，表示查询中操作表的顺序。id相同，执行顺序由上到下；id不同，如果是子查询，id号会自增，id越大，优先级越高。

#### select_type查询类型（重点）

* simple：不需要UNION操作或者不包含子查询的简单SELECT查询
* primary：一个需要UNION操作或者含有子查询的SELECT，位于最外层的单位查询的select_type即为primary
* union：UNION连接的两个SELECT查询，第一个查询是dervied派生表，除了第一个表外，第二个以后的表 select_type都是union
* dependent union：与UNION一样，出现在UNION或UNION ALL语句中，但是这个查询要受到外部查询的影响
* union result：包含UNION的结果集，在UNION和UNION ALL语句中,因为它不需要参与查询，所以id字段为null
* subquery：除了FROM字句中包含的子查询外，其他地方出现的子查询都可能是subquery
* dependent subquery：与dependent union类似，表示这个subquery的查询要受到外部表查询的影响
* derived：FROM字句中出现的子查询，也叫做派生表，其他数据库中可能叫做内联视图或嵌套SELECT

#### table

显示的单位查询的表名，有如下几种情况：

* 如果查询使用了别名，那么这里显示的是别名
* 如果不涉及对数据表的操作，那么这显示为null
* 如果显示为尖括号括起来的就表示这个是临时表，后边的N就是执行计划中的id，表示结果来自于这个查询产生
* 如果是尖括号括起来的<union M,N>，也是一个临时表，表示这个结果来自于UNION查询的id为M,N的结果集

#### partitions

使用的哪些分区。

#### type（重点）

显示单位查询的连接类型或者理解为访问类型，访问性能从好到差排列为：

* system：表中只有一行数据或者是空表，等同于系统表，这是const类型的特例，平时不会出现

* const：使用唯一索引或者主键，返回记录一定是1行记录的等值WHERE条件时，通常type是const

* eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描

* ref：非唯一性索引（组合索引或非唯一索引）扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，它返回所有匹配某个单

  独值的行，然而，它可能会找到多个符合条件的行，所以他应该属于查找和扫描的混合体

* fulltext：全文索引检索，若全文索引和普通索引同时存在时，mysql不管代价优先选择使用全文索引

* ref_or_null：与ref方法类似，只是增加了NULL值的比较

* unique_subquery：用于WHERE中的IN形式子查询，子查询返回不重复值唯一值

* index_subquery：用于IN形式子查询使用到了辅助索引或者IN常数列表，子查询可能返回重复值，可以使用索引将子查询 去重

* range：索引范围扫描，常见于使用>、<、IS NULL、BETWEEN、IN、LIKE等运算符的查询中

* index_merge：表示查询使用了两个以上的索引，最后取交集或者并集，常见AND，OR的条件使用了不同的索引，官方排序这个在ref_or_null之后，但是实际上由于要读取所个索引，性能可能大部分时间都不如range

* index：SELECT结果列中使用到了索引，type会显示为index。全部索引扫描，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取数据文件的查询、可以使用索引排序或者分组的查询

* ALL：全表扫描数据文件，然后再在server层进行过滤返回符合要求的记录

注意事项：

* 除了all之外，其他的type都可以使用到索引
* 除了index_merge之外，其他的type只可以用到一个索引
* 最少要使用到range级别

#### possible_keys

本次查询中可能选用的索引，一个或多个。

#### key

查询真正用到的索引。select_type为index_merge时，这里可能出现两个以上的索引。

#### key_len

key_len越小，索引效果越好。key_len只计算where条件用到的索引长度，排序和分组用到的索引不会计算在内。

#### ref

* 如果是使用的常数等值查询，这里显示const
* 如果是连接查询，被驱动表的执行计划会显示驱动表的关联字段
* 如果是条件使用了表达式或者函数，或者条件列发生了内部隐式转换，这里显示为func

#### rows

这里是执行计划中估算的扫描行数，不是精确值（InnoDB不是精确的值，MyISAM是精确的值，主要原因是InnoDB里面使用了MVCC并发机制）。

#### filtered

filtered列指示将由MySQL Server层需要对存储引擎层返回的记录进行筛选的估计百分比，也就是说存储引擎层返回的结果中包含有效记录数的百分比。最大值为100，这意味着没有对行进行筛选。值从100减小表示过滤量增加。rows显示检查的估计行数，rows×filtered显示将与下表联接的行数。

####  extra（重要）

* using filelsort：MySQL中无法利用索引完成的排序操作称为“文件排序”，说明MySQL会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取，需要优化sql
* using temporary：使用了用临时表保存中间结果，MySQL在对查询结果排序时使用临时表。常见于排序order by和分组查询group by，需要优化SQL
* using index：查询时不需要回表查询，直接通过索引就可以获取查询的结果数据。表示相应的SELECT查询中使用到了覆盖索引（Covering Index），避免访问表的数据行，效率不错；如果同时出现using where ，说明索引被用来执行查找索引键值；如果没有同时出现using where ，表明索引用来读取数据而非执行查找动作
* using where：表示MySQL将对存储引擎提取的结果进行过滤，过滤条件字段无索引
* using join buffer：表明使用了连接缓存，比如说在查询的时候多表join的次数非常多。可以将配置文件中的缓冲区的join buffer调大一些
* impossible where：where子句的值 总是false ，不能用来获取任何元组

### 5.4 show profile查看硬件性能瓶颈

#### 介绍

Query Profiler是MySQL自带的一种硬件性能瓶颈诊断分析工具。Query Profiler可以精确定位SQL语句执行的各种CPU和IO资源消耗情况，以及该SQL执行所耗费的时间。该功能默认没有打开，需要手动配置。

#### 开启

```sql
select @@profiling;
show variables like '%profil%';
setprofiling=1;
```

#### 示例

* **show profile** 和 **show profiles**语句可以展示**当前会话**（退出session后profiling重置为0）中执行语句的资源使用情况
* **show profiles**：以列表形式显示最近发送到服务器上执行的语句的**资源使用情况**，显示的记录数由变量profiling_history_size控制，默认15条
* **show profile**：展示最近一条语句执行的详细资源占用信息，默认显示Status和Duration两列
* **show profile**还可根据show profiles列表中的 **Query_ID**选择显示某条记录的性能分析信息

```sql
show profile for query 2;
show profile cpu,swaps for query 2;
```

### 5.5 SQL语句优化

#### 索引优化

* 考虑数据的业务场景：读多还是写多？如果读多，则为搜索字段（where中的条件）、分组字段、排序字段、select查询列创建合适的索引；如果写多，索引更新不能频繁，更新非常频繁的数据不适宜建索引，因为维护索引的成本高

* 尽量建立**组合索引**并注意组合索引的创建顺序，按照顺序组织查询条件、尽量将筛选粒度大的查询条件放到最左边（粒度从大到小）

* SELECT语句中尽量不要使用*，尽量使用覆盖索引
* 索引长度尽量短，短索引可以节省索引空间，使查找的速度得到提升，同时内存中也可以装载更多的索引键值。太长的列，可以选择建立前缀索引
* order by排序应该遵循最佳左前缀查询，如果是使用多个索引字段进行排序，那么排序的规则必须相同（同是升序或者降序），否则索引同样会失效

#### LIMIT优化

* 如果预计SELECT语句的查询结果是一条，最好使用 **LIMIT 1**，避免全表扫描
* 处理分页会使用到LIMIT ，当翻页到非常靠后的页面的时候，偏移量会非常大，它会导致MySQL扫描大量不需要的行然后再抛弃掉，这时LIMIT的效率会非常差。解决方案：单表分页时，使用自增主键排序，然后where id > offset做筛选，limit后面只写rows，例如`SELECT * FROM (SELECT * FROM tuser2 WHERE id > 1000000 AND id < 1000500 ORDER BY id) t limit 0, 20`

#### 其他查询优化

* 针对JOIN连表查询：做到（1）小表驱动大表，因为使用JOIN的话，第一张表是必须全扫描的，以少关联多就可以减少这个扫描次数；（2）两张表的关联字段最好都建立索引，而且最好字段类型是一样的
* 避免全表扫描：（1）MySQL在使用不等于的时候无法使用索引导致全表扫描；（2）避免MySQL放弃索引查询，如果MySQL估计使用全表扫描要比使用索引快，则不使用索引（最典型的场景就是数据量少的时候）
* WHERE条件中尽量不要使用NOT IN语句（用NOT EXISTS替代）
* 合理利用慢查询日志、EXPLAIN执行计划、show profile查看SQL执行时的资源使用情况

### 5.6 MySQL服务器优化

#### 内存优化

* innodb_buffer_pool_size表示缓冲池字节大小，设置足够大才能将数据读取到内存中。建议innodb_buffer_pool_size设置为物理内存的50%~80%（SHOW GLOBAL STATUS LIKE 'innodb_buffer_pool_pages%'，查看Innodb_buffer_pool_pages_free，为0表示耗尽）

* 使用足够大的写入缓存 **innodb_log_buffer_size**，推荐 innodb_log_buffer_size设置为 0.25 * innodb_buffer_pool_size

* innodb_log_file_size推荐设置为64~512M。过大，实例恢复时间长；过小，造成日志切换频繁

#### 磁盘优化

* 对于生产环境来说通用查询日志、慢查询日志、错误日志不需要开启的。分析查询性能时慢查询日志的阈值建议设置为long_query_time=0.3（单位秒）

* 设置合适的**innodb_flush_log_at_trx_commit**，与redo log和undo log日志磁盘刷新策略有关系，innodb_flush_log_at_trx_commit=1

* 每提交1次事务同步写binlog到磁盘中，可以设置为n，sync_binlog=1

* innodb_max_dirty_pages_pct控制脏页占innodb_buffer_pool_size的多大比例时触发刷脏页到磁盘，推荐值为25%~50%

* innodb_io_capacity：后台进程最大IO性能指标。默认200，如果SSD调整为5000~20000

* 指定多个innodb共享表空间文件的路径innodb_data_file_path

* MySQL主从复制格式，binlog_format=row

#### 操作系统优化

内核参数、资源限制、磁盘调度策略

#### 服务器硬件优化

加内存、提升网络带宽、使用SSD硬盘、提升CPU性能









