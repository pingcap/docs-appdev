---
title: SQL 开发规范及基本原则
summary: 了解开发业务及应用时需要遵守的规范和基本原则。
---

# SQL 开发规范及基本原则

本文主要介绍在开发业务及应用时，需要遵守的 SQL 规范及基本原则。

## 对象命名规范

* 命名只能使用英文字母、数字、下划线；
* 建议使用具有意义的英文词汇，词汇中间以下划线分隔；
* 建议所有的数据库对象名使用小写字母；
* 避免用 TiDB 的[关键字](https://docs.pingcap.com/zh/tidb/dev/keywords)，如：`group，error，rank` 等作为对象名；
* 数据库对象名长度请注意[标识符长度硬限制](https://docs.pingcap.com/zh/tidb/dev/tidb-limitations#%E6%A0%87%E8%AF%86%E7%AC%A6%E9%95%BF%E5%BA%A6%E9%99%90%E5%88%B6) ；

## 数据库命名规范

* 数据库相当于对象的集合，按照业务模块、产品线和访问权限隔离需求来创建数据库；
* 建议数据库名称不要超过 16 个字符，如：基础信息库（basicinfo_db）、认证中心库（certifycenter_db）；

## 表命名规范

* 表名命名尽可能使用实际含义的英文单词或简写，同一业务或者模块的表尽可能使用相同的前缀；
* 建议表的名称不要超过 32 个字符；
* 建议对表的用途进行注释说明方便维护；
* 只支持将 [lower-case-table-names](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_lower_case_table_names) 值设为 2，使用 `SHOW TABLES` 等命令查看数据字典的表名区分大小写，使用 `CREATE TABLE` 命令建表时会按照小写比较表名是否重复；

## 字段命名规范

* 字段命名尽可能使用实际含义的英文单词或简写；
* 字段名不建议超过 32 个字符，多单词组成的字段名，使用能代表意义的缩写；
* 不同表之间相同业务意义的字段应使用相同的字段类型，建议使用相同的字段名；
* 建议对字段的用途进行注释说明方便维护，枚举型需指明主要值的含义，如”0 - 离线，1 - 在线”；
* 布尔值列命名为 [is\_描述]。如 member 表上表示为 enabled 的会员的列命名为 is_enabled；


## 索引命名规范

* 主键索引命名建议：pk\_[表名称简写]\_[字段名简写] ；
* 唯一索引命名建议：uk\_[表名称简写]\_[字段名简写] ；
* 普通索引命名建议：idx\_[表名称简写]\_[字段名简写] ；
* 组合索引名中的字段名顺序应与字段序保持一致 ；

## 表的设计

* 建表时指定主键或者非空唯一索引，能与各项日志复制工具更好地兼容；
* 主键不建议带有业务含义，如果带有业务含义，需要确认不存在写入热点；
* 业务表使用自增主键时，字段类型推荐使用 `bigint unsigned`，最大值可达 18446744073709551615；
* 出于为性能考虑，尽量避免存储超宽表，数据长度过大的字段最好拆到独立的表存储，建议表字段数不超过 60 个，建议单行的总数据大小不要超过 64K，TiDB 硬限制单行的总数据大小不超过 6 MB；
* 不推荐使用复杂的数据类型，如 blob 或者 json；
* 进行 join 的关联字段，数据类型保证一致，避免隐式转换造成精度等问题；
* 不能以范式作为唯一标准或者指导，在设计过程中，需要从实际需求出发，以性能提升为根本目标来展开设计工作。基于减少表关联和提升性能考虑，可以适当保留冗余数据字段的反范式设计；
* 表对象的设计请注意[单个 Table 的限制](https://docs.pingcap.com/zh/tidb/dev/tidb-limitations#%E5%8D%95%E4%B8%AA-table-%E7%9A%84%E9%99%90%E5%88%B6) ；

## 字符集

* 建表时不指定字段集，使用默认的 `utf8mb4` 编码；
* `utf8mb4` 的默认排序规则为 `utf8mb4_bin`（区分大小写）；
* 使用 `utf8mb4_general_ci`（不区分大小写）排序规则，需要在集群环境部署时配置 [新的排序规则框架](https://docs.pingcap.com/zh/tidb/dev/character-set-and-collation#%E6%8E%92%E5%BA%8F%E8%A7%84%E5%88%99%E6%94%AF%E6%8C%81)；
* 其他的排序规则仅为语法支持，实际不支持；

## 列的自增属性

* 列的自增属性仅保证唯一，仅保证在单个计算节点中自增；
* 自增值按可变长度区间分配给不同的计算节点，不保证多个节点并发插入时字段值自增，不保证单台计算节点多次取得自增值区间的连续性，带有自增属性的列出现空洞和值跳跃是正常现象；
* 业务不应该依赖自增属性的连续性和有序性，如记录插入顺序的排序应按照记录的创建时间；
* 不应在插入语句中显式指定具有自增属性的列的值，显式指定的值可能已经分配给其他节点，可能出现值重复冲突的报错；
* 允许移除列的 `AUTO_INCREMENT` 属性，但请谨慎评估，移除该属性后不可恢复；

## 创建、删除表规范

* 表的建立在遵循表命名规范前提下，如果业务应用内部封装建表删表语句，需要增加判断逻辑，如 `CREATE TABLE IF NOT EXISTS table_name` 防止业务流程异常中断。
* 不支持 `CREATE TABLE AS SELECT` 语法，需要改写为表结构复制 `CREATE TABLE LIKE` 和将数据写入的 `INSERT INTO SELECT` 的组合语句，同时需要注意事务大小限制；

## 变更表规范

* 不支持单条 `ALTER TABLE` 语句中完成多个表结构变更操作，不能在单个语句中添加或删除多个列或索引，需要更换成多个单个列或索引的操作；
* 不支持对字段类型的有损修改或修改为超集，请参考[DDL 的限制](https://docs.pingcap.com/zh/tidb/stable/mysql-compatibility#ddl-%E7%9A%84%E9%99%90%E5%88%B6)；

## 创建、删除索引规范

* 在记录数过亿的表上创建索引，需要平衡执行时间和集群性能之间的关系，创建索引会[影响业务响应时间](https://docs.pingcap.com/zh/tidb/stable/online-workloads-and-add-index-operations#tidb_ddl_reorg_batch_size--32)；
* 创建组合索引时，过滤条件最好的等值条件应该在最前面，范围查询条件应该在等值条件后面，再根据 `SELECT ` 投影字段的数量评估选择覆盖索引；
* 组合索引的最后一个有序字段是主键时，创建时应显式写出主键字段名；
* 删除索引可能会对线上 SQL 执行计划的造成影响，需要经过测试环境的谨慎评估；

## 大事务处理

* TiDB 单个事务的大小存在限制，请参考[事务限制](https://docs.pingcap.com/zh/tidb/stable/transaction-overview#%E4%BA%8B%E5%8A%A1%E9%99%90%E5%88%B6)；
* 事务限制设置过大，或者 Batch 数量过多，可能导致 tidb-server OOM，需要评估好内存容量；
* 为了使性能达到最优，需要对大事务按某个业务维度进行拆分，每 1000～5000 行提交一个事务；

## `SELECT *` 使用规范

* 禁止使用 `SELECT *` 进行查询。建议按需求选择合适的字段列，杜绝直接 `SELECT *` 读取全部字段，减少网络带宽消耗，有效利用覆盖索引；

## 防范业务热点操作规范

* 对于写入量非常大的表，应当通过应用性能压测等方法在测试环境模拟表的热点情况;
* 根据行数据存储为 KV pair 时 key 的构造方式，使用不同的方法规避表的主键写入热点,通过 `SELECT   TIDB_ROW_ID_SHARDING_INFO FROM information_schema.tables WHERE TABLE_SCHEMA='db_name' and TABLE_NAME='table_name'` 查看表的 key 的构造方式；
* `NOT_SHARDED(PK_IS_HANDLE)` 说明使用主键构成 KV pair 的 key，建议采用[雪花算法](https://github.com/Meituan-Dianping/Leaf)生成 UUID 主键，或者由业务生成非连续的主键，防范写入热点；
* `NOT_SHARDED` 说明由 TiDB 分配 rowid 构成 KV pair 的 key，使用[SHARD_ROW_ID_BITS](https://docs.pingcap.com/zh/tidb/stable/troubleshoot-hot-spot-issues#%E4%BD%BF%E7%94%A8-shard_row_id_bits-%E5%A4%84%E7%90%86%E7%83%AD%E7%82%B9%E8%A1%A8) 语法修改为 `SHARD_BITS=N` 的随机打散方式;
* `SHARD_BITS=N` 说明使用 rowid 方式，并且在 rowid 的 N bit 上随机变化，产生业务连续写入时会在 2 的 N 次幂个位置上连续插入，热点已经进行打散；
* 如果 `NOT_SHARDED(PK_IS_HANDLE)` 的表业务无法改造使用雪花算法，需要修改集群的 `alt-primary-key` 参数项为 true 后，重新建表；
* 使用主键做为分区字段创建 Hash 或 Range 分区表，也可以避免热点，但业务不能有其他字段的索引查找需求；

## 数据删除规范

* 删除表中全部的数据时，使用 `TRUNCATE` 重建表或者 `DROP TABLE` 后重建方式，不要使用 `DELETE FROM` 语句 ；
* 数据删除后，磁盘空间不会立即释放，需要等待 TiDB 后台的 GC (garbage collection)  和 Compaction 机制清除数据历史版本；
* 对于按范围进行部分数据的删除，如果超过大事务的限制，可以参考以下窗口函数的方法，分成批量小任务进行数据的删除；

  a. 将数据按照主键排序，然后调用窗口函数 row_number() 为每一行数据生成行号，接着调用聚合函数按照设置好的页大小对行号进行分组，最终计算出每个分组的行号的最小值和最大值。

  ```
  MySQL [demo]> select min(t.serial_no) as start_key, max(t.serial_no) as end_key, count(*) as page_size from ( select *, row_number () over (order by serial_no) as row_num from tmp_loan ) t group by floor((t.row_num - 1) / 50000) order by start_key;
  ```
  
  ```
  +-----------+-----------+-----------+
  | start_key | end_key   | page_size |
  +-----------+-----------+-----------+
  | 200000000 | 200050001 |     50000 |
  | 200050002 | 200100007 |     50000 |
  |  ........ |.......... | ........  |
  | 201900019 | 201950018 |     50000 |
  | 201950019 | 201999003 |     48985 |
  +-----------+-----------+-----------+
  40 rows in set (1.51 sec)
  ```

  b. 借助计算好的分组信息，使用 `where serial_no between start_key and end_key`  操作每个分组的数据，实现高效数据删除或者更新。

## 分页查询 order by 语法使用规范

* 分页查询语句需要带有排序条件，除业务排序条件外还应包含主键或者其他唯一键以保证分页稳定，避免没有业务排序字段或者一个业务排序字段值匹配多条记录导致结果集不稳定；
* 常规分页语句如下, start 是起始记录数，page_offset 是每页记录数；

  ```
  MySQL [demo]>   SELECT * FROM table_a t ORDER BY gmt_modified DESC,pk LIMIT start,page_offset;
  ```

## GROUP BY 语法使用规范

* SELECT 字段中不得引用未在 GROUP BY 子句中声明的非聚集字段，即不得使用 MySQL non-full group by 语法;
* `SELECT class,stdt_name,MAX(score) as max_score FROM score GROUP BY class` 是不被允许的;

## 模糊匹配 LIKE 语法使用规范

* 使用 LIKE 查找字符串，通配符 % 放首位会导致无法使用索引，业务语句中不能直接使用 % 放首位，如 '%keyword%',或者使用时结合其他有效的约束条件在小结果集内模糊匹配;

## 视图使用规范

* 可以使用不同的视图对应不同的应用程序查询需求，防范应用程序直接访问数据表的操作风险；
* 视图不可更新，不支持 `UPDATE、INSERT、DELETE` 等写入操作；

## 多表关联查询规范

* 多表关联应该显式使用 `JOIN` 子句，避免漏掉关联条件，造成笛卡尔积；
* 嵌套 SQL 语句应该为不同表指定不同别名；
* 高并发交易场景，单条语句关联表不超过两张，使用执行计划为 IndexJoin 的多表关联语句，外表应建立正确的条件过滤索引，内表应建立正确的关联和条件过滤索引；
* 低并发的分析场景，单条语句关联表不超过 10 张，其中亿级表不超过 2 张，注意 `tidb_mem_quota_query` 变量指定的单条语句内存使用限制，默认是 1GB，生产建议不超过 16 GB；

## 分区表使用规范

* 当数据需要按时间进行归档清理时，可按某个业务时间段对表进行分区，对分区进行 `TRUNCATE` 操作满足数据清理要求；
* 按时间进行分区表的粒度应该将分区记录数控制在十亿级别，不应该配置太小，如日分区，也不应太大，如年分区；
* 分区表的二级索引属于分区内索引在分区表上进行查找时，查找条件必须包含分区字段的查找条件，定位到具体分区后再经过二级索引回表查找；
