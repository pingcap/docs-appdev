---
title: 创建和管理索引
category: management
---

# 创建和管理索引

## 创建和管理索引

SQL 语句参阅
[CREATE INDEX](https://docs.pingcap.com/zh/tidb/stable/sql-statement-create-index#create-index)
[ADD INDEX](https://docs.pingcap.com/zh/tidb/stable/sql-statement-add-index#add-index)
[DROP INDEX](https://docs.pingcap.com/zh/tidb/stable/sql-statement-drop-index#drop-index)
[RENAME INDEX](https://docs.pingcap.com/zh/tidb/stable/sql-statement-rename-index#rename-index)
[SHOW INDEXES [FROM|IN]](https://docs.pingcap.com/zh/tidb/stable/sql-statement-show-indexes#show-indexes-fromin)

## 索引命名规范

1. 唯一索引：uk_[表名称简写]_[字段名简写]
2. 普通索引：idx_[表名称简写]_[字段名简写]
3. 多单词组成的 column_name，取尽可能代表意义的缩写。

## 索引设计

1. 选择区分度大的列建立索引，不在低基数列上建立索引，例如：“性别”，“是否是 XXX”；
2. 单张表的索引数量控制在 5 个以内，避免冗余索引；
3. 索引中的字段数建议不超过 5 个；
4. 唯一索引建议由 3 个以下字段组成；
5. 尽量不要在频繁更新的列上创建索引；
6. 对于确定需要组成组合索引的多个字段，建议将选择性高的字段靠前放；
7. 最左前缀原则，使用联合索引时，从左向右匹配，比如索引 idx_c1_c2_c3 (c1,c2,c3)，相当于创建了 (c1)、(c1,c2)、(c1,c2,c3) 三个索引，where 条件包含上面三种情况的字段比较则可以用到索引，但像 where c1=a and c3=c 只能用到 c1 列的索引，像 c2=b and c3=c 等情况就完全用不到这个索引；
8. 不对过长的 VARCHAR 字段建立索引；
9. 定期删除一些长时间未使用过的索引；
10. ORDER BY，GROUP BY，DISTINCT 的字段需要添加在索引的后面，形成覆盖索引；
11. 新的 select,update,delete 上线，都要先 explain，确保索引的正确性；
12. 不建议在 where 条件索引列上使用函数，会导致索引失效，如 lower(email)；
13. 使用 like 模糊匹配，% 不要放首位，会导致索引失效。
