---
title: 创建和管理表
summary: 创建和管理数据表的基本 SQL 操作和规范
category: management
---

# 创建和管理表

本文介绍创建和管理数据表的基本 SQL 操作和规范。

## 创建和管理表

SQL 语句参阅：

- [CREATE TABLE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-create-table)
- [DROP TABLE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-drop-table#drop-table)
- [FLASHBACK TABLE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-flashback-table#flashback-table)
- [RENAME TABLE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-rename-table#rename-table)
- [ALTER TABLE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-alter-table#alter-table)
- [RECOVER TABLE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-recover-table#recover-table)

## 表命名规范

1. 同一业务或者模块的表尽可能使用相同的前缀，表名称尽可能表达含义；
2. 多个单词以下划线分隔，不推荐超过32个字符。
3. 建议对表的用途进行注释说明，以便于统一认识，例如：

    * 临时表（`tmp_t_crm_relation_0425`）。
    * 备份表（`bak_t_crm_relation_20170425`）。
    * 业务运营临时统计表（`tmp_st_[业务代码]_[创建人缩写]_[日期]`）。
    * 账期归档表（`t_crm_ec_record_YYYY[MM][DD]`）。
    
4. 不同业务模块的表单独建立 `DATABASE`，并增加相应注释。 

## 表设计

1. TiDB 中的一张表的行（Rows）是按照主键的字节序排序的（整数类型的主键我们会使用特定的编码使其字节序和按大小排序一致），即使在 `CREATE TABLE` 语句中不显式的创建主键，TiDB 也会为该表分配一个隐式主键
2. 在进行 TiDB 表设计过程中，有以下几点需要注意：

    * 表需要有主键或者唯一索引。
    * 尽量选择有意义的列作为主键。
    * 出于为性能考虑，尽量避免存储超宽表，表字段数不建议超过 60 个，建议单行的总数据大小不要超过 64K，数据长度过大字段最好拆到另外的表。
    * 不推荐使用复杂的数据类型。
    * 需要 `JOIN` 的字段，数据类型保障绝对一致，避免隐式转换。
