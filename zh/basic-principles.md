---
title: SQL 开发规范及基本原则
summary: 了解开发业务及应用时需要遵守的规范和基本原则。
---

# SQL 开发规范及基本原则

本文主要介绍在开发业务及应用时，需要遵守的 SQL 规范及基本原则。

## 对象命名规范

* 命名建议使用具有意义的英文词汇，词汇中间以下划线分隔；
* 命名只能使用英文字母、数字、下划线；
* 避免用 TiDB 的保留字如 group，order 等作为单个字段名；
* 建议所有数据库对象使用小写字母。

## 创建、删除表规范

基本原则：表的建立在遵循表命名规范的前提下，建议业务应用内部封装建表删表语句来增加判断逻辑，防止业务流程异常中断。例如：`create table if not exists table_name` 或者 `drop table if exists table_name` 语句建议增加 `if` 判断，避免应用侧由于表的改动造成的异常中断。

## `SELECT *` 使用规范

基本原则：禁止使用 `SELECT *` 进行查询。建议按需求选择合适的字段列，杜绝直接 `SELECT *` 读取全部字段，减少网络带宽消耗，有效利用覆盖索引。	

## 大事务处理

基本原则：

* 按需求选择合适的字段列，杜绝直接用 `SELECT *` 读取全部字段，以减少网络带宽消耗，有效利用覆盖索引。
* 拒绝大事务，TiDB 对单个事务的大小有限制，这层限制是在 KV 层面，反映在 SQL 层面的话，一行数据会映射为一个 KV entry，每多一个索引，也会增加一个 KV entry，所以这个限制反映在 SQL 层面如下：

    * 单行数据不大于 6MB
    * 总的行数*(1 + 索引个数) < 30万
    * 一次提交的全部数据小于 100MB

> **注意：**
>
> 无论是大小限制还是行数限制，还要考虑 TiDB 做编码以及事务额外 Key 开销，在使用的时候，建议每个事务的行数不要超过 1 万行，否则有可能会超过限制，或者是性能不佳。建议无论是 `Insert`，`Update` 还是 `Delete` 语句，都通过分 Batch 或者是加 Limit 的方式限制，启用 Batch 操作步骤参考：`set @@session.tidb_distsql_scan_concurrency=5`。

> **注意：**
>
> 该参数设置过大可能导致 tidb oom，SQL 占用内存评估 5 * 4 = 20G，剩余内存至少 30G。设置参考：`set @@session.tidb_batch_insert=1`。

## Region 热点

产生热点的原因：

* 这条 SQL 涉及到的 Region 的 leader 全部在这个 TiKV 上。
* 这条 SQL 只涉及到一个 Region，并且有大量的请求使用同样或者类似的 SQL 语句。

基本原则：

* 如果表的数据量比较小，数据存储大概率只涉及到一个 region，大量请求对该表进行写入或者读取都会造成该 region 热点，可以通过手工拆分 region 方式进行调整，调整方式如下：

    {{< copyable "sql" >}}

    ```sql
    operator add split-region 1   // 将 region 1 对半拆分成两个 region
    ```

* 如果表的数据量比较大，region leader 分布不均衡，某些 tikv 节点 region leader 比较多，不均衡导致的热点需要通过某种机制平衡 leader 分布，平衡方式参考如下：

    自动均衡：

    {{< copyable "sql" >}}

    ```sql
    curl -G "host:status_port/tables/DB_NAME/TABLE_NAME/scatter"  // 打散相邻 region
    ```

    手动均衡：

    {{< copyable "sql" >}}

    ```sql
    operator add transfer-leader 1 2   // 把 region 1 的 leader 调度到 store 2
    ```

建议事项：

* Handle ID 设计时避免连续自增主键的设计，建议采用置换算法，例如时间 12010120180601。
* TiDB Partition：  通过设置表分区方式来避免热点，支持按照 Hash 以及按照 Range 分区。
* `SHARD_ROW_ID_BITS`： 这个 TABLE OPTION 是用来设置隐式 `_tidb_rowid` 的分片数量的 bit 位数。对于 PK 非整数或没有 PK 的表，TiDB 会使用一个隐式的自增 rowid，大量 `INSERT` 时会把数据集中写入单个 region，造成写入热点。通过设置 `SHARD_ROW_ID_BITS` 可以把 rowid 打散写入多个不同的 region，缓解写入热点问题。 但是设置的过大会造成 RPC 请求数放大，增加 CPU 和网络开销。`SHARD_ROW_ID_BITS = 4` 代表 16 个分片， `SHARD_ROW_ID_BITS = 6` 表示 64 个分片，`SHARD_ROW_ID_BITS = 0` 就是默认值 1 个分片 。操作语句如下：

    `CREATE TABLE` 语句示例: 

    {{< copyable "sql" >}}

    ```sql
    CREATE TABLE t (c int) SHARD_ROW_ID_BITS = 4
    ```

    `ALTER TABLE` 语句示例：

    {{< copyable "sql" >}}

    ```sql
    ALTER TABLE t SHARD_ROW_ID_BITS = 4
    ```

## 字段上使用函数规范

基本原则：在取出字段上可以使用相关函数，但是在 `Where` 条件中的过滤条件字段上严禁使用任何函数，包括数据类型转换函数。示例如下：

- 错误的写法：

    ```sql
    select gmt_create
    from .. .
    where date_format(gmt_create，'%Y­%m­%d %H:%i:%s') = '2009­01­01 00:00:0';
    ```

- 正确的写法：

    {{< copyable "sql" >}}

    ```sql
    select date_format(gmt_create，'%Y­%m­%d %H:%i:%s')
    from .. .
    where gmt_create = str_to_date('2009­01­01 00:00:00'，'%Y­%m­%d %H:%i:s');
    ```

## 数据删除规范

基本原则：删除表中全部的数据时，使用 `TRUNCATE` 或者 `DROP` 后重建方式，不要使用 `DELETE`。

详细说明：`DELETE`，`TRUNCATE` 和 `DROP` 都不会立即释放空间，对于 `TRUNCATE` 和 `DROP` 操作，在达到 TiDB 的 GC (garbage collection) 时间后（默认 10 分钟），TiDB 的 GC 机制会删除数据并释放空间。对于 `DELETE` 操作 TiDB 的 GC 机制会删除数据，但不会释放空间，而是当后续数据写入 RocksDB 且进行 compact 时对空间重新利用。

## 分页查询规范				

基本原则：

* 分页查询语句全部都需要带有排序条件,除非业务方明确要求不要使用任何排序来随机展示数据。常规分页语句写法(`start`：起始记录数，`page_offset`：每页记录数)，例如：

    {{< copyable "sql" >}}

    ```sql
    select * from table_a t order by gmt_modified desc limit start，page_offset; 
    ```

* 多表 `Join` 的分页语句，如果过滤条件在单个表上，内查询语句必须走覆盖索引，先分页，再 `Join`。示例如下：

- 错误的写法：

    ```sql
    select a.column_a，a.column_b .. . b.column_a，b.column_b .. .
    from table_t a，table_b b
    where a.xxx.. .
    and a.column_c = b.column_d
    order by a.yyy limit
    start，page_offset;
    ```

- 正确的写法：

    {{< copyable "sql" >}}

    ```sql
    select from
    where
    a.column_a，a.column_b .. . b.column_a，b.column_b .. . (select t.column_a，t.column_b .. .
    from table_t t
    where t.xxx.. .
    order by t.yyy limit start，page_offerset) a,				
    table_b b
    a.column_c = b.column_d;
    select * from t limit 10000,10;
    select * from t order by c limit 10000,10;
    ```

## 其他规范

* `WHERE` 条件中不在索引列上进行数学运算或函数运算。
* 用 `in()` /`union` 替换 `or`，并注意 `in` 的个数小于 300。
* 禁止使用 `%` 前缀进行模糊前缀查询。
