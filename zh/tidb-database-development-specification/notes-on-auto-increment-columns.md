---
title: TiDB 数据库开发规范 - 自增列的使用注意
summary: 介绍自增列的注意事项。
---

# 自增列的使用注意

## 1. 原理

TiDB 的自增 ID (auto_increment) 只保证自增且唯一，并不保证连续分配。TiDB 目前采用批量分配的方式，所以如果在多台 TiDB 上同时插入数据，分配的自增 ID 会不连续。当多个线程并发往不同的 tidb-server 插入数据的时候，有可能会出现后插入的数据自增 ID 小的情况。此外，TiDB 允许给数值类型的列指定 auto_increment，且一个表只允许一个属性为 auto_increment 的列。

## 2. 最佳实践

设置自增 ID 的目的一般是将它作为表内数据的唯一性约束，因此被设计为主键或唯一索引，此类列属性应带有 `not null`。

自增列可以定义在整数、FLOAT 或 DOUBLE 的列上；但是最佳实践是使用整型。

在几种整型类型中，我们建议使用 BIGINT，这是由于即使在单机数据库中也屡见 INT 类型的自增 ID 被耗光的情况，而 TiDB 被用于处理比单机数据大得多的数据量。此外 TiDB 采用多线程的方式分配自增 ID，因此 INT 类型无法满足需求。另外自增 ID 一般不需要存储负值，为列增加 unsigned 属性可以扩充一倍的 id 存储容量。INT 无符号的范围是 0 到 4294967295，BIGINT 无符号的范围是 0 到 18446744073709551615。

综上，自增 ID 设计的最佳实践如下：

```sql
`auto_inc_id` bigint unsigned not null unique key auto_increment comment '自增 ID'
```

## 3. 显式赋值的后果及挽救措施

TiDB 实现自增 ID 的原理是每个 tidb-server 实例缓存一段 ID 值用于分配（目前会缓存30000 个 ID），用完这段值再去取下一段。

在集群中有多个 tidb-server 时，人为向自增列写入值之后，可能会导致 TiDB 分配自增值冲突而报 “Duplicate entry” 错误：

假设有这样一个带有自增 ID 的表：`create table t(id int unique key auto_increment,c int);`

假设集群中有两个 tidb-server 实例 A 和 B（A 缓存 [1,30000] 的自增 ID，B 缓存[30001,60000] 的自增 ID），依次执行如下操作：

客户端向 B 插入一条将 id 设置为 1 的语句 `insert into t values (1,1)`，并执行成功。

客户端向 A 发送 Insert 语句 `insert into t (c) (1)`，这条语句中没有指定 id 的值，所以会由 A 分配，当前 A 缓存了 [1, 30000] 这段 ID，所以会分配 1 为自增 ID 的值，并把本地计数器加 1。而此时数据库中已经存在 id 为 1 的数据，最终返回 `Duplicated Entry` 错误。

处理该问题只需要调大表上的 `AUTO_INCREMENT` 属性值即可让所有 tidb-server 重新获取一段自增 ID：

1. 确认表上自增值的最大值：`show create table t;`

2. 修改表上的自增值最大值到一个更大的值：`alter table t AUTO_INCREMENT=120000;`

自增列的另外一个问题是：因为每个显式赋值，都需要 reset auto_increment 动作，这会体现在SQL 的 prepare 时间变长。这一点 尤其在批量数据操作中 体现明显，需要避免这样的使用方式。
