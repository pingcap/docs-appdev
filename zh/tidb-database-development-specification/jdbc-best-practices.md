---
title: TiDB 中文开发者指南系列-->JDBC 最佳实践
summary: 介绍 JDBC 连接串的较好配置。
---

# JDBC 最佳实践

## 1. MySQL Connector/J 推荐版本

TiDB 服务端兼容 MySQL 5.7，客户端推荐使用 5.1.36 或更高版本 的 5.1.x jdbc 驱动 [下载链接](https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.36.tar.gz)。

## 2. JDBC 参数设置

Java 应用常用的数据库连接池包括 weblogic、c3p0、Druid 等。使用连接池配置数据源时，需要配置一系列参数，其中比较重要的包括 jdbc 的 url 配置，超时探活机制等。充分认识并理解各项参数有助于让 TiDB 发挥出更高的性能。MySQL 5.1 版本 jdbc configuration properties 参见 [链接](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html)：

一个建议的 url 配置如下：

```java
spring.datasource.url=JDBC:mysql://{TiDBIP}:{TiDBPort}/{DBName}?characterEncoding=utf8&useSSL=false&useServerPrepStmts=true&prepStmtCacheSqlLimit=10000000000&useConfigs=maxPerformance&rewriteBatchedStatements=true&defaultfetchsize=-214783648
```