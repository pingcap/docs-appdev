---
title: TiDB 中文开发者指南系列-->索引的使用注意
summary: 介绍索引的设计和使用。
---


# 索引的使用注意

与其他的 RDBMS 一样，TiDB 需要通过索引才能保障数据读取的性能，而TiDB 往往被用于承载单机 RDBMS 难以承载的数据量，因此索引尤其是复合索引的设计尤为关键。

## 1. TiDB 中的索引

索引也是数据，也要占用存储空间。和表中的数据一样，TiDB 中表的索引在存储引擎中也被作为 kv 来存储，一行索引是一个 kv 对。例如一张有 10 个索引的表，每插入一行数据的时候，会写入 11 个 kv 对。

TiDB 支持主键索引，唯一索引，也支持二级索引，构成以上索引的可以是单一列，也可以是多个列（复合索引）。

TiDB 目前（v5.0）还不支持反向/双向索引，全文索引，分区表的全局索引。

TiDB 中在查询的谓词是 =，`>`，`<`，`>=`，`<=`，like ‘...%’，not like ‘...%’，in，not in，`<>`，!=，is null，`<=>`，is not null，between…and … 时能够使用索引，使用与否由优化器来决策。

TiDB 中在查询的谓词是 like ‘%...’，like ‘%...%’，not like ‘%...’，not like ‘%...%’ 时，都无法使用索引。

## 2. 复合索引的设计

TiDB 中的复合索引形如 `key indexkeyname (a,b,c)` ，与其他数据库一样，设计复合索引的一般原则是尽可能的把使用频率比较高的字段放在前面。使用中需要特别注意，复合索引中前一列的范围查询会中止后续索引列的使用，可以通过下面的案例来理解这个特性。在如下的查询中：

```sql
select a,b,c from tablename where a<predicate>’<value1>’ and b<predicate>’<value2>’ and c<predicate>’<value3>’;
```

如果 a 条件的谓词（语句中的 predicate）是 = 或 in，那么在 b 的查询条件上就可以利用到组合索引 (a,b,c) 。例：`select a,b,c from tablename where a**=**1 and b\<5 and c=’abc’;`

同样的，如果 a 条件和 b 条件的谓词都是 = 或 in，那么在 c 上的查询就可以利用到组合索引 (a,b,c) 。例：`select a,b,c from tablename where a **in** (1,2,3) and b**=**5 and c=’abc’;`

如果 a 条件的谓词不是 = 也不是 in，那么 b 上的查询就无法利用到组合索引 (a,b,c)。此时 b 条件将在 a 条件筛选后的数据中进行无索引的数据扫描。例：`select a,b,c from tablename where a**\>**1 and b\<5 and c=’abc’;`

这是由于在 TiDB 中，复合索引中排在前面的列如果被用于范围查询，那么后续列的查询就会在前一列筛选后的数据范围中进行非索引的扫描。

综上，在 TiDB 中进行复合索引设计时，需要尽可能的将使用频率高的，经常被点查使用的列排在前面，将经常进行范围查询的列排在后面。

另外形如 `select c, count(*) from tabname where a=1 and b=2 group by c order by c;` 的查询可以利用到索引 (a,b,c)，同样遵循上面的原则。
