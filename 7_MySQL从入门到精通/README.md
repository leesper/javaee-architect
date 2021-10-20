# MySQL从入门到精通学习笔记

## 1. SQL语言分类

SQL语言全称为结构化查询语言，分为以下4个子类：DQMC。

1. 数据定义语言：简称DDL（Data Definition Language），用来定义数据库对象：数据库、表、列等。关键字：create、alter、drop等
2. 数据查询语言：简称DQL（Data Query Language），用来查询数据库中表的记录。关键字：select、from、where等
3. 数据操作语言：简称DML（Data Manipulation Language），用来对数据库中表的记录进行更新。关键字：insert、delete、update等
4. 数据控制语言：简称DCL（Data Control Language），用来定义数据库的访问权限和安全级别，及创建用户，关键字：grant等

## 2. SQL解析顺序

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

#### MySQL Server层

Server层涉及到几个对象：连接器、查询缓存、分析器、优化器、执行器和存储引擎。全部串起来就是SQL执行的流程：

1. 客户端通过连接器连接到MySQL服务器，并向MySQL发送SQL语句
2. 服务器先在查询缓存中查找，如果缓存命中了就直接返回
3. 否则分析器就对SQL语句进行词法分析、语法分析和预处理，并生成抽象语法树
4. 优化器再对生成的抽象语法树进行优化，即选择最优执行计划
5. 执行器进行权限验证，验证通过后调用存储引擎提供的接口执行语句，有索引则直接查询索引，没有索引则全表扫描并过滤。

#### InnoDB存储引擎

MySQL有很多可插拔的存储引擎：

1. InnoDB：默认存储引擎，支持事务和行级锁
2. MyISAM：插入查询速度快，但不支持事务
3. Memory：所有数据存储在内存中，重启服务后就丢失
4. Falcon：据说是InnoDB的替代者
5. CSV：以CSV格式存储数据

![](/Users/likejun/javaee-architect/7_MySQL从入门到精通/assets/imdb架构图.png)

InnoDB分为两大部分：磁盘文件和内存结构。磁盘文件部分包含redo log重做日志文件、系统表空间和用户表空间。当InnoDB数据文件发生错误时重做日志文件可以恢复数据。重做日志是两个文件循环写入的，可以通过配置文件设置日志文件大小。

系统表空间是共享的，它存储InnoDB数据字典、double write buffer、change buffer和undo logs。用户表空间存储表的数据和索引信息。InnoDB逻辑存储结构：表空间、段（数据段、索引段、回滚段）、区、页、行。

内存结构包括Buffer Pool缓冲池、额外内存池、redo log buffer重做日志缓冲。Buffer Pool缓冲池包括数据页、索引页、更新缓冲、自适应哈希索引、锁信息和数据字典信息（数据、库对象和表对象的元信息集合）。

InnoDB工作原理涉及到内存数据落盘、CheckPoint检查点机制和Double Write的原理，比较复杂。









