---
title: TiDB 中文开发者指南系列-->SQL 开发规范
summary: 介绍 TiDB 上 SQL的推荐写法。
---

# SQL 开发规范

## 1. 建表删表规范

1. 基本原则：表的建立在遵循表命名规范前提下，建议业务应用内部封装建表删表语句增加判断逻辑，防止业务流程异常中断。

2. 详细说明：`create table if not exists table_name` 或者 `drop table if exists table_name` 语句建议增加 if 判断，避免应用侧由于SQL 命令运行异常造成的异常中断。

## 2. select \* 使用规范

1. 基本原则：避免使用 select \* 进行查询。

2. 详细说明：按需求选择合适的字段列，避免盲目地 SELECT \* 读取全部字段，因为其会消耗网络带宽。考虑将被查询的字段也加入到索引中，以有效利用覆盖索引功能。

## 3. 大事务处理

基本原则：避免大事务，TiDB 对单个事务的大小有限制，这层限制是在 KV 层面。反映在 SQL 层面的话，简单来说一行数据会映射为一个 KV entry，每多一个索引，也会增加一个 KV entry。所以这个限制反映在 SQL 层面是：

- 最大单行记录容量为 120MB（TiDB v5.0 及更高的版本可通过 tidb-server 配置项 `performance.txn-entry-size-limit` 调整，低于 v5.0 的版本支持的单行容量为 6MB. 

- 支持的最大单个事务容量为 10GB（TiDB v4.0 及更高版本可通过 tidb-server 配置项 `performance.txn-total-size-limit` 调整，低于 TiDB v4.0 的版本支持的最大单个事务容量为 100MB. 

另外注意，无论是大小限制还是行数限制，还要考虑 事务执行过程中，TiDB 做编码以及事务额外 Key 的开销，在使用的时候，为了使性能达到最优，建议每 100～500 行写入一个事务。

## 4. 字段上使用函数规范

1. 基本原则：在取出字段上可以使用相关函数,但是在 Where 条件中的过滤条件字段上避免使用任何函数,包括数据类型转换函数，以避免索引失效。或者可以考虑使用表达式索引功能。

2. 详细说明：

    不推荐的写法：
    
    ```sql
    select gmt_create
    from ...
    where date_format(gmt_create，'%Y%m%d %H:%i:%s') = '20090101 00:00:0'
    ```
    
    推荐的写法：
    
    ```sql
    select date_format(gmt_create，'%Y%m%d %H:%i:%s')
    from .. .
    where gmt_create = str_to_date('20090101 00:00:00'，'%Y%m%d %H:%i:s')
    ```

## 5. 数据删除规范

1. 基本原则：

    删除表中全部的数据时，推荐使用 TRUNCATE 或者 重建（`DROP then CREATE` 的方式，或者使用分区表中的`DROP/TRUNCATE PARTITON` 的方式，
    或者带有条件的`DELETE where condition ..limit N`的循环方式。

2. 详细说明：

    `DELETE`，`TRUNCATE` 和 `DROP` 都不会立即释放空间。对于 `TRUNCATE` 和 `DROP` 操作，在达到 TiDB 的 GC (garbage collection) 时间后（默认 10 分钟. ，TiDB 的 GC 机制会删除数据并释放空间。对于 DELETE 操作 TiDB 的 GC 机制会删除数据，但不会释放空间，而是当后续数据写入 RocksDB 且进行 compact 时对空间重新利用。

## 6. 其他规范

1. WHERE 条件中不要在索引列上进行数学运算或函数运算；

2. 用 in/union 替换 or，并注意 in 的个数小于 300；

3. 避免使用 %前缀进行模糊前缀查询；

4. 如应用使用 Multi Statements 执行 SQL，即将多个 SQL 使用分号连接，一次性地发给客户端执行，TiDB 只会返回第一个 SQL 的执行结果。
