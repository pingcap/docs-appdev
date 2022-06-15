
TiDB 兼容MySQL 协议、常用的功能及语法，可以使用支持 MySQL 的命令行和图形客户端程序连接 TiDB 数据库。

## 安装 MariaDB 分支的 MySQL 客户端
在 Redhat Enterprise Linux 7 操作系统或其兼容发行版中，默认包含由社区维护的 MariaDB 分支的 MySQL。以 RHEL 7.6 版本为例，内含 MariaDB 5.5。在操作系统 REPO 源配置正确的情况下，通过 yum 命令快速安装 mariadb 客户端，再通过 mysql 客户端连接 TiDB。

{{< copyable "command" >}}

```command
yum install -y mysql
mysql -uroot -h172.16.4.66 -P4000 -prootpassword --comments
```

注意：
MySQL 命令行客户端在 5.7.7 版本之前默认清除 Optimizer Hints。如果需要在这些早期版本的客户端中使用 Hint 语法，需要在启动客户端时加上 --comments 选项，例如 mysql -h 127.0.0.1 -P 4000 -uroot --comments。 

通过 --help 子命令也可以看到具体的版本信息。

{{< copyable "command" >}}

```command
mysql --help
```

```
mysql  Ver 15.1 Distrib 5.5.65-MariaDB, for Linux (x86_64) using readline 5.1
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
```

## 安装 Oracle 的 MySQL 5.7 客户端
通过配置 Oracle 官方的 REPO 源，在 Redhat Enterprise Linux 7 操作系统或兼容发行版中可以通过 yum 命令快速安装 mariadb 客户端，再通过 mysql 客户端连接 TiDB。

注意：
从 MySQL 8.0.4开始，MySQL 8.0 的默认身份验证方式改用 caching_sha2_password 验证，与 TiDB 不兼容，需要使用 5.7 版本的客户端。

{{< copyable "command" >}}

```command
vi /etc/yum.repos.d/mysql-community.repo
```

```
[mysql57-community]
name=MySQL 5.7 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/6/$basearch/
enabled=1
gpgcheck=0
```

{{< copyable "command" >}}

```command
yum repolist all | grep mysql
```

```
mysql57-community/x86_64      MySQL 5.7 Community Server            启用:    414
```

{{< copyable "command" >}}

```command
yum install mysql
```

通过 --help 子命令也可以看到具体的版本信息。

{{< copyable "command" >}}

```command
mysql --help
```

```
mysql  Ver 14.14 Distrib 5.7.31, for Linux (x86_64) using  EditLine wrapper
Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
```

## 安装 Navicat 图形化工具
Navicat 是一款强大的图形化管理数据库工具，在 MySQL DBA 群体中较为流行，产品发展较快，最新的 15 版本支持 MySQL 8.0。Navicat 是一款商业产品，官方提供 14 天下载试用，并支持订阅销售方式。

- Navicat 程序主界面连接菜单

![连接菜单](/media/ch1-1-navicat1.png)

- Navicat 新建连接对话框

![连接对话框](/media/ch1-1-navicat2.png)

- Navicat 连接成功操作界面

![成功操作界面](/media/ch1-1-navicat3.png)

- Navicat 活动数据库下拉菜单

![下拉菜单](/media/ch1-1-navicat4.png)

- Navicat 命令语句输入框

![语句对话框](/media/ch1-1-navicat5.png)

## 安装 DBeaver 图形化工具
DBeaver 是一款为开发人员和数据库管理员准备的通用数据库工具，支持多种主流的数据库，属于开源产品，在金融等行业中流行较广。

- DBeaver 程序主界面连接菜单

![连接菜单](/media/ch1-1-dbeaver1.png)

- DBeaver 新建连接对话框

![连接对话框](/media/ch1-1-dbeaver2.png)

- DBeaver 连接成功操作界面

![成功操作界面](/media/ch1-1-dbeaver3.png)

- DBeaver 命令语句输入框

![下拉菜单](/media/ch1-1-dbeaver4.png)