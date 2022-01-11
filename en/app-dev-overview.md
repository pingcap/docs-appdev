---
title: App Development Overview
summary: Learn the overview of App development.
aliases: ['/tidb/dev/connectors-and-apis']
---

# App Development Overview

TiDB supports the MySQL protocol and most of the MySQL features. In general, to develop your application based on TiDB, you can use either native driver/connectors or popular third-party frameworks like ORM and migration tools.

This document lists some of the app development guides that show you how to build simple applications based on TiDB.

> **Note:**
>
> The support levels for connectors and ORM frameworks are described as follows:
>
> - Full support: There is no known issue using the driver or connector with TiDB.
> - Verified support: You might encounter errors when using some of the features, due to "unsupported features" of TiDB. For example, TiDB does not support Foreign Key constraints, so the foreign key related features do not work in Hibernate. However, all other features have been verified.

## Connectors

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Go</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>MySQL connector</td>
    <td></td>
    <td>Full</td>
    <td><a href="https://docs.pingcap.com/appdev/dev/for-go-sql-driver-mysql">App Development for go-sql-drive/mysql</a></td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Java</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>JDBC/MySQL connector</td>
    <td>5.1.36 or later</td>
    <td>Full</td>
    <td>NA</td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Python</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>python/mysql-connector</td>
    <td>8.0 or later</td>
    <td>Full</td>
    <td><a href="https://docs.pingcap.com/appdev/dev/for-python-mysql-connector">App Development for mysql-connector-python</a></td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Ruby</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td><a href="https://github.com/brianmario/mysql2">Mysql2</a></td>
    <td>0.5.3</td>
    <td>Full</td>
    <td>NA</td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">PHP</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td><a href="https://www.php.net/manual/en/ref.pdo-mysql.php">PDO_MYSQL</a></td>
    <td>All</td>
    <td>Full</td>
    <td>NA</td>
  </tr>
</table>

## ORM frameworks

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Go</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>GORM</td>
    <td></td>
    <td>Verified</td>
    <td><a href="https://docs.pingcap.com/appdev/dev/for-gorm">App Development for GORM</a></td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Java</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>Hibernate</td>
    <td>5.2 or later</td>
    <td>Verified</td>
    <td><a href="https://docs.pingcap.com/appdev/dev/for-hibernate-orm">App Development for Hibernate ORM</a></td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Python</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>Django</td>
    <td>3.2.x</td>
    <td>Verified</td>
    <td><a href="https://docs.pingcap.com/appdev/dev/for-django">App Development for Django</a></td>
  </tr>
  <tr>
    <td>SQLAlchemy</td>
    <td>1.4 or later</td>
    <td>Verified</td>
    <td><a href="https://docs.pingcap.com/appdev/dev/for-sqlalchemy">App Development for SQLAlchemy</a></td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">Ruby</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>ActiveRecord</td>
    <td>6.1</td>
    <td>Verified</td>
    <td><a href="https://pingcap.com/blog/build-a-ruby-on-rails-app-with-tidb">Power Up Your Rails Apps with a NewSQL Database</a></td>
  </tr>
</table>

<table style="display: table; width: 100%;">
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <col width="25%" />
  <tr>
    <th colspan="4" bgcolor="#d9d9d9" align="center">PHP</th>
  </tr>
  <tr>
    <th>Name</th>
    <th>Recommended versions</th>
    <th>Support level</th>
    <th>Example</th>
  </tr>
  <tr>
    <td>Laravel</td>
    <td>5.1 or later</td>
    <td>Verified</td>
    <td><a href="https://docs.pingcap.com/appdev/dev/for-laravel">App Development for Laravel</a></td>
  </tr>
</table>
