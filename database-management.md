---
title: 创建和管理库
summary: 创建和管理数据库的基本 SQL 操作和规范
category: management
---

# 创建和管理库

本文介绍创建和管理数据库的基本 SQL 操作和规范。

## 创建和管理库

SQL 语句参阅
- [CREATE DATABASE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-create-database#create-database)
- [DROP DATABASE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-drop-database#drop-database)
- [ALTER DATABASE](https://docs.pingcap.com/zh/tidb/stable/sql-statement-alter-database#alter-database)

## 库命名规范

建议按照业务、产品线或者其它指标进行区分，一般不要超过 20 个字符。如：临时库（tmp_crm）、测试库（test_crm）。
