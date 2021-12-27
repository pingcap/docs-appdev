---
title: TiDB 数据库开发规范 - 数据库对象设计
summary: 介绍如何设计数据库中的对象，如索引、字段类型等。
---

# 数据库对象设计

## 1. 表的设计

1. TiDB 中的一张表的 RowID 是按照主键的字节序排序的（整数类型的主键我们会使用特定的编码使其字节序和按大小排序一致）。使在 `CREATE TABLE` 语句中不显式地创建主键，TiDB 也会为该表分配一个隐式主键

2. 在进行 TiDB 表设计过程中，有以下几点需要注意

   - 为了保障主从集群复制以及增量备份的幂等性，表需要有主键或者唯一索引，需要唯一索引所有字段非空（避免出现多条空值的重复记录）。
   - 出于为性能考虑，尽量避免存储超宽表，表字段数建议不超过 60个，建议单行的总数据大小不要超过 64K，数据长度过大字段最好拆到另外的表。
   - 不推荐使用复杂的数据类型。
   - 需要 join 的字段，数据类型保障绝对一致，避免隐式类型转换

3. TiDB 字符集默认就是 UTF8 ，而且目前只支持 UTF8

   - 如果原文件是UTF8, 则只需要将编码改为 `utf8mb4`，此外不需要做其他转换。这是因为 `utf8mb4` 编码 是 utf8 的超集。

   - TiDB 中，utf8mb4 的默认排序规则为 `utf8mb4_bin`（区分大小写）

   - TiDB v4.0 及以后的版本，支持 `utf8mb4_general_ci`（不区分大小写）

## 2 字段的设计

1. 整数类型

   - TiDB 支持 MySQL 所有的整数类型，包括 INTEGER/INT、TINYINT、SMALLINT、MEDIUMINT 以及 BIGINT。

   - INT：所有整数类型的字段推荐只使用 INT 或者 BIGINT。

   - BIGINT：定义中不推荐添加长度。

   - 推荐使用 INT(10) UNSIGNED 存储 IPv4 格式 IP 地址。

   - TINYINT(1)、TINYINT(4) 都是存储一个字节，并不会因为括号里的数字改变。例如 TINYINT(4) 存储 22 则会显示 0022，因为最大宽度为4，达不到的情况下用0来补充。

2. 浮点类型

   - TiDB 支持 MySQL 所有的浮点类型，包括 FLOAT、DOUBLE、DECIMAL、NUMERIC等。

   - DECIMAL(M,D)：推荐使用 DECIMAL 类型。FLOAT 和 DOUBLE 在存储的时候，存在精度损失的问题，很可能在值的比较时，得到不正确的结果。DECIMAL 在与 VARCHAR、CHAR 类型比较时会转换为 DOUBLE 类型进行比较，也存在精度损失问题，建议在进行比较时使用显式类型转换，如 CAST 避免精度损失。

3. 日期时间类型

   - TiDB 支持 MySQL 所有的日期时间类型，包括 DATE、DATETIME、TIMESTAMP、TIME 以及 YEAR。

   - DATE：所有只需要精确到天的字段全部使用 DATE 类型，而不应该使用 TIMESTAMP 或者 DATETIME 类型；

   - DATETIME：所有需要精确到时间(时分秒)的字段均使用 DATETIME，不要使用 TIMESTAMP 类型。TIMESTAMP 支持的精度范围比较小，支持范围为 1970-2038 年，而 DATETIME 支持 1000-9999 年。

   - 时间字段使用时间日期类型，不要使用字符串类型存储。否则无法利用日期函数对日期字段进行操作，而需要使用字符串函数进行复杂的操作。

4. 处理日期和时间类型时，请记住下面这些：

   - 日期部分必须是按 年-月-日 的顺序（比如，‘1998-09-04’），而不是 月-日-年 或者 日-月-年 的顺序。
     - 如果日期中包含年份为两位数，则这个年份是有歧义的，并不显式地表示实际年份。对于 DATETIME、DATE 和 TIMESTAMP 类型，TiDB 使用如下规则来消除歧义。

         - 将 01 至 69 之间的值转换为 2001 至 2069 之间的值。
         - 将 70 至 99 之间的值转化为 1970 至 1999 之间的值。

   - 如果输出格式必须是数值类型，TiDB 会自动将日期或时间值转换为数值类型。例如

      ```sql
      SELECT NOW(), NOW()+0, NOW(3)+0;
      +---------------------+----------------+--------------------+
      | NOW()               | NOW()+0        | NOW(3)+0           |
      +---------------------+----------------+--------------------+
      | 2012-08-15 09:28:00 | 20120815092800 | 20120815092800.889 |
      +---------------------+----------------+--------------------+
      ```

   - 设置不同的 sql_mode 会改变 TiDB 的行为, 以定义 TiDB 支持哪些 SQL 语法及执行哪种数据验证检查。比如 NO_ZERO_DATE 表示 ‘在严格模式，不要将 '0000-00-00' 做为合法日期’。

5. 字符串类型

   - TiDB 支持 MySQL 所有的字符串类型，包括 CHAR、VARCHAR、BINARY、VARBINARY、BLOB、TEXT、ENUM 以及 SET。

   - VARCHAR(N)：所有动态长度字符串全部使用 VARCHAR 类型，N 表示的是字符数不是字节数，比如 VARCHAR(256)，需要根据实际的宽度来选择 N，N
      尽可能小。

   - CHAR：仅仅只有单个字符的字段使用 CHAR(1) 类型，例如性别字段。

   - TEXT：仅仅当字符数量可能超过 20000 个的时候，才建议使用TEXT类型来存放字符类数据，因为所有 TiDB 数据库都会使用 UTF8 字符集。所有使 TEXT 类型的字段建议和原表进行分拆，与原表主键单独组成另外一个表进行存放。

   - 不建议使用 ENUM、SET 类型，尽量使用 TINYINT 来代替。

   - 当使用宽字段类型（如 Text、MediumBlob、MediumText）时，需注意读取并发度，以控制内存使用预防 OOM 。

## 3. 字段默认值

1. 在一个数据类型描述中的 DEFAULT value 段描述了一个列的默认值。这个默认值必须是常量，不可以是一个函数或者是表达式。但是对于时间类型，可以例外的使用NOW、CURRENT_TIMESTAMP、LOCALTIME、LOCALTIMESTAMP 等函数作为 DATETIME 或者 TIMESTAMP 的默认值。

2. BLOB、TEXT 以及 JSON 不可以设置默认值。

3. 如果一个列的定义中没有 DEFAULT 的设置。TiDB 按照如下的规则决定：

   - 如果该类型可以使用 NULL 作为值，那么这个列会在定义时添加隐式的默认值设置 DEFAULT NULL。

   - 如果该类型无法使用 NULL 作为值，那么这个列在定义时不会添加隐式的默认值设置。

4. 对于一个设置了 NOT NULL 但是没有显式设置 DEFAULT 的列，当 INSERT、REPLACE 没有涉及到该列的值时，TiDB 根据当时的 SQL_MODE 进行不同的行为：

   - 如果此时是 strict sql mode，在事务中的语句会导致事务失败并回滚，非事务中的语句会直接报错。

   - 如果此时不是 strict sql mode，TiDB 会为这列赋值为列数据类型的隐式默认值。

5. 此时隐式默认值的设置按照如下规则：

   - 对于数值类型，它们的默认值是 0。当有 AUTO_INCREMENT 参数时，默认值会按照增量情况赋予正确的值。

   - 对于除了时间戳外的日期时间类型，默认值会是该类型的“零值”。时间戳类型的默认值会是当前的时间。

   - TIMESTAMP 和 DATETIME 列可以被自动初始化或者更新为当前时间。

   - 对于表里面任意的 TIMESTAMP 或者 DATETIME 列，可以将默认值或者自动更新值指定为 current timestamp，通过在列定义时指定 DEFAULT CURRENT_TIMESTAMP 和 ON UPDATE CURRENT_TIMESTAMP 可以设置这些属性。DEFAULT 也可以指定成某个特定的值，比如 DEFAULT 0 或者 DEFAULT '2000-01-01 00:00:00'。

        ```sql
        CREATE TABLE t1 (  
        ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,  
        dt DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP);
        ```

   - 对于除枚举以外的字符串类型，默认值会是空字符串。对于枚举类型，默认值是枚举中的第一个值。

## 4. 索引的设计

1. 选择区分度大的列建立索引，不在低基数列上建立索引，例如：“性别”，“是否是 XXX”。

2. 单张表的索引数量控制在 5 个以内，避免冗余索引。

3. 索引中的字段数建议不超过 5 个。

4. 唯一索引建议由 3 个或更少的字段组成。

5. 尽量不要在频繁更新的列上创建索引。

6. 尽可能地将使用频率高的，经常被点查使用的列排在复合索引靠前的位置，将经常进行范围查询的列排在后面。

7. 最左前缀匹配原则（ leftmost prefix ），使用联合索引时，从左向右匹配，比如索引 idx_c1_c2_c3 (c1,c2,c3)，相当于创建了 (c1)、(c1,c2)、(c1,c2,c3) 三个索引，where 条件包含上面三种情况的字段比较则可以用到索引，但像 where c1=a and c3=c 只能用到 c1 列的索引，像 c2=b and c3=c 等情况就完全用不到这个索引。

8. 很长的 VARCHAR 字段建立索引时，指定索引长度，没必要对全字段建立索引，根据实际文本区分度决定索引长度即可， 例如 idx_table_name (name(10))。

9. 定期删除一些长时间未使用过的索引。

10. ORDER BY，GROUP BY，DISTINCT 的字段需要添加在索引的后面，形成覆盖索引。

11. 新的 select,update,delete 上线，都要先执行 explain 命令，观察执行计划是否有异常情况发现，以确保索引的正确性。

12. 不建议在 where 条件索引列上使用函数，会导致索引失效，如 lower(email)。

13. 使用 like 模糊匹配，% 不要放首位，会导致索引失效。

## 5. 权限的设计

TiDB 在数据库初始化时会生成一个 'root'@'%' 的默认账户，生产环境建议调整 root 用户为强密码，且不对外开放。线上所需用户建议按照用户或者业务场景划分，根据实际情况对每个用户授予相应权限，例如：

| **序号** | **用户名** | **涵义**     | **用途**                   |
| -------- | ---------- | ------------ | -------------------------- |
| 1        | root       | 超级用户     | 全局管理，禁止对外开放     |
| 2        | dba        | 数据库管理员 | 数据库 DBA                 |
| 3        | app        | 应用开发     | 应用开发                   |
| 4        | tempuser   | 临时统计     | 线上业务临时统计，只读用户 |
| 5        | other      | 其他用户     | 第三方人员访问             |

## 6. 约束的设计

在 TiDB 中，开发时需要注意：

1. 主键：
   - 一张表只能有一个主键
   - 支持定义一个多列组合作为复合主键
   - 主键列不能允许 NULL 值
   - 对于非索引组织表，可以修改或删除主键。

2. 唯一键：
   TiDB 中的唯一键可在新建表时添加，也可以在已有表上进行添加和删除操作。

3. 外键：TiDB 不支持外键，仅做语法兼容，不会在DML中对外键进行约束检查。外键的级联操作多表数据的功能需要在应用中实现。

**注意**
TiDB 中 UNIQUE KEY 和 PRIMARY KEY 的区别：

1. Primary key 的一个或多个列必须为 NOT NULL，如果列为 NULL，在增加 PRIMARY KEY 时，列自动更改为 NOT NULL。而 UNIQUE KEY 对列没有此要求。
2. 一个表只能有一个 PRIMARY KEY，但可以有多个 UNIQUE KEY。

## 7. 分区表的设计

1. 分区类型的选择：需要按照业务场景选择

   - TiDB 当前支持的类型包括 Range 分区、List 分区和 Hash 分区 等。
   - 不同的分区 对分区键的数据类型有要求。
   - Range 分区和 List 分区可以用于解决业务中大量删除带来的性能问题，支持快速删除分区。- Hash 分区则可以用于大量写入场景下的数据打散。

2. 分区键的选择：

   TiDB 要求分区表的每个唯一键（以及主键），必须包含分区表达式中用到的所有列。例如，以下是合法的

   ```sql
   CREATE TABLE t1 (
       col1 INT NOT NULL,
       col2 DATE NOT NULL,
       col3 INT NOT NULL,
       col4 INT NOT NULL,
       UNIQUE KEY (col1, col2, col3)
   )
   PARTITION BY HASH(col3)
   PARTITIONS 4;
   ```
   
3. SQL 需要适配分区表的写法

   TiDB 的分区表支持 SQL 执行计划中的分区裁剪。分区裁剪是指通过分析查询语句中的过滤条件，只选择可能满足条件的分区，不扫描匹配不上的分区，进而显著地减少计算的数据量。为了利用此功能，需要在 SQL 的 WHERE 条件中，带上分区键。Hash 分区表 要求等值比较 才可能使用 Hash 分区表的裁剪；Range 分区可以支持 等值、in 条件、区间比较等实现分区裁剪。

   SQL 示例：

   ```sql
   WHERE
   partition_column = constant
   或者
   partition_column IN (constant1, constant2, ..., constantN)
   ```
