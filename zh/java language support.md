因为和 MySQL 保持良好的兼容，TiDB 不需要使用专用的语言驱动。在本节中，主要介绍常见的开发语言和框架如何连接 TiDB 数据库。

## 使用 JDBC 连接 TiDB 数据库
利用 JDBC 连接 TiDB 时，连接方式与 MySQL 相同，以下是一个简单示例。

```command
package com.pingcap.test;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicLong;
public class SingleMixEngine {
  public static void main(String[] args) throws Exception {
    Class.forName("com.mysql.jdbc.Driver");
    Properties props = new Properties();
    props.setProperty("user", "pingcap");
    props.setProperty("password", "pingcap");
    SingleMixEngine eng = new SingleMixEngine();
    eng.execute(props,"jdbc:mysql://192.168.58.51:8066/testdb");
  }
  final AtomicLong tmAl = new AtomicLong();
  final String table="news_table";
  public void execute(Properties props,String url) {
    CountDownLatch cdl = new CountDownLatch(1);
    long start = System.currentTimeMillis();
    for (int i = 0; i < 1; i++) {
      TestThread insertThread = new TestThread(props,cdl, url);
      Thread t = new Thread(insertThread);
      t.start();
      System.out.println("Test start");
    }
    try {
      cdl.await();
      long end = System.currentTimeMillis();
      System.out.println("Total cost:" + (end-start) + "ms");
    } catch (Exception e) {
      e.printStackTrace();
    }
  }

  class TestThread implements Runnable {
    Properties props;
    private CountDownLatch countDownLatch;
    String url;
    public TestThread(Properties props,CountDownLatch cdl,String url) {
      this.props = props;
      this.countDownLatch = cdl;
      this.url = url;
    }
    public void run() {
      Connection connection = null;
      PreparedStatement ps = null;
      Statement st = null;
      try {
        connection = DriverManager.getConnection(url,props);
        connection.setAutoCommit(true);
        st = connection.createStatement();
        String sql = "drop table if exists " + table;
        System.out.println("SQL:\n\t"+sql);
        st.execute(sql);
                
        sql = "create table " + table + "(id int,title varchar(20))";
        System.out.println("SQL:\n\t" + sql);
        st.execute(sql);
                
        sql = "insert into " + table + " (id,title) values(?,?)";
        System.out.println("SQL:\n\t"+ sql);
        ps = connection.prepareStatement(sql);
        for (int i = 1; i <= 3; i++) {
          ps.setInt(1,i);
          ps.setString(2, "测试"+i);
          ps.execute();
          System.out.println("Insert data:\t"+i+","+"测试"+i);
        }
                
        sql = "select * from " + table + " order by id";
        System.out.println("SQL:\n\t"+ sql);
        ResultSet rs = st.executeQuery(sql);
        int colcount = rs.getMetaData().getColumnCount();
        System.out.println("Current Data:");
        while(rs.next()){
        for(int i=1;i<=colcount;i++){
          System.out.print("\t"+rs.getString(i));
        }
        System.out.println();
      }
                
        sql = "update " + table + " set title='test1' where id=1";
        System.out.println("SQL:\n\t"+ sql);
        st.execute(sql);
        sql = "select * from " + table + " order by id";
        rs = st.executeQuery(sql);
        System.out.println("Current Data:");
        while(rs.next()){
          for(int i=1;i<=colcount;i++){
            System.out.print("\t"+rs.getString(i));
          }
          System.out.println();
        }
                
        sql = "delete from " + table + " where id=2";
        System.out.println("SQL:\n\t"+ sql);
        st.execute(sql);
        sql = "select * from " + table + " order by id";
        rs = st.executeQuery(sql);
        System.out.println("Current Data:");
        while(rs.next()){
          for(int i=1;i<=colcount;i++){
            System.out.print("\t"+rs.getString(i));
          }
          System.out.println();
        }
                
        sql = "create index idx_1 on " + table + "(title)";
        System.out.println("SQL:\n\t"+ sql);
        st.execute(sql);
                
        sql = "drop index idx_1 on " + table;
        System.out.println("SQL:\n\t"+ sql);
        st.execute(sql);
      } catch (Exception e) {
        e.printStackTrace();
      } finally {
        if (ps != null)
          try {
            ps.close();
          } catch (SQLException e1) {
            e1.printStackTrace();
          }
        if (connection != null)
          try {
            connection.close();
          } catch (SQLException e1) {
                    e1.printStackTrace();
          }
        this.countDownLatch.countDown();
      }}}}
```

下文以 BenchmarkSQL 压测工具添加连接 TiDB 功能为例，介绍 Java 程序添加连接 TiDB 数据库的大致步骤。

### 添加 MySQL JDBC 驱动
添加 mysql-connector-java-8.0.13.jar 驱动文件到 lib/mysql/ 目录下。
建议使用  5.1.36 版本或以上的驱动程序文件。

### 程序中添加 MySQL 的驱动支持
在 run/funcs.sh 文件中的 function setCP() 中，添加如下的驱动描述

```command
	mysql)
	    cp="../lib/mysql/*:../lib/*"
	    ;;
```

### 配置连接属性
在 run/props.mysql 或类似的连接参数文件中，添加如下的连接信息。

```command
db=mysql
driver=com.mysql.jdbc.Driver
conn=jdbc:mysql://localhost:3306/tpcc?useSSL=false&useServerPrepStmts=true&useConfigs=maxPerformance
user=root
password=
```

### 程序中添加其他的压测业务支持内容
其他的修改属于添加对 MySQL 的压测功能的支持，如语法适配、功能特性适配等，以与添加连接的例子无关。
关于 BenchmarkSQL 压测工具添加对 TiDB 的完整支持内容请参阅：
https://github.com/pingcap/benchmarksql/compare/5.0...5.0-mysql-support?expand=1
https://github.com/pingcap/benchmarksql/compare/5.0...5.0-mysql-support-opt-2.1?expand=1

## 使用 Java 开发框架连接 TiDB 数据库
TiDB 兼容各种开发工具，开发框架和应用接口等，利用软件生态中大量的成熟工具降低了业务开发和运维团队的学习和使用成本，部分框架适配如下:
### 与 ibatis 适配
利用 ibatis 连接 TiDB 时，连接方式与 MySQL 相同，以下是一个简单示例。
#### jdbc配置：

```command
jdbc.driverClass=com.mysql.jdbc.Driver
jdbc.jdbcUrl=jdbc:mysql://192.168.58.51:8066/TESTDB?useUnicode=true&characterEncoding=utf-8
jdbc.user=root
jdbc.password=123456
```

#### 映射文件配置：

```command
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.mapper.UserMapper">
    <insert id="saveUser" parameterType="com.bean.User">
    insert into user(id,name,phone,birthday)
    values (0,#{name},#{phone},#{birthday})
    <selectKey keyProperty="id" order="after" resultType="int">
         select last_insert_id() as id
    </selectKey>
    </insert>
    <delete id="deleteUserById" parameterType="java.lang.String">
    delete from user where id=#{id}
    </delete>
    <update id="updateUser" parameterType="com.bean.User">
    update user set name=#{name}, phone=#{phone}, 
    birthday=#{birthday} where id=#{id}
    </update>
    <update id="updateUsers">
    /*!mycat: sql=select * from user;*/update users set usercount=(select count(*) from user),ts=now()
    </update>
    <select id="getUserById" parameterType="java.lang.String" resultType="com.bean.User">
    select * from user where id=#{id}
    </select>
    <select id="getUsers" resultType="com.bean.User">
    select * from user
    </select>
</mapper>
```
语句select last_insert_id() as id可用来获取新写入记录的ID。
updateUsers方法用到了 TiDB的注解，由于ibatis中的符号#具有特殊含义，因此注解中不能含有#。

### 与 hibernate 适配
利用hibernate连接TiDB时，连接方式与MySQL相同，以下是一个简单示例。
hibernate.cfg.xml：

```command
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
  "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
  "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
  <session-factory>
    <property name="hibernate.connection.driver_class">
      com.mysql.jdbc.Driver
    </property>
    <property name="hibernate.connection.url">
jdbc:mysql://192.168.58.51:8066/testdb?useUnicode=true&amp;characterEncoding=utf-8
    </property>
    <property name="hibernate.connection.username">
      root
    </property>
    <property name="hibernate.connection.password">
      123456
    </property>
    <property name="hibernate.dialect">
      org.hibernate.dialect.MySQLInnoDBDialect
    </property>
    <property name="hibernate.format_sql">
      true
    </property>
    <property name="hibernate.hbm2ddl.auto">
      update
    </property>
    <mapping resource="com/pingcap/test/News.hbm.xml"/>
  </session-factory>
</hibernate-configuration>
```

News.hbm.xml：

```command
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-mapping PUBLIC 
  "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
  "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
<hibernate-mapping>
  <class name="com.pingcap.test.News" table="news_table">
    <id name="id" type="java.lang.Integer">
      <column name="id" />
    </id>
    <property name="title" type="java.lang.String">
      <column name="title" />
    </property>
    <property name="content" type="java.lang.String">
      <column name="content" />
    </property>
  </class>
</hibernate-mapping>
```

News.java:

```command
package com.pingcap.test;
public class News {
  private Integer id;
  private String title;
  private String content;
  public Integer getId() {
    return id;
    }
  public void setId(Integer id) {
    this.id = id;
  }
  public String getTitle() {
    return title;
  }
  public void setTitle(String title) {
    this.title = title;
  }
  public String getContent() {
    return content;
  }
  public void setContent(String content) {
    this.content = content;
  }
}
```

NewsManager.java：

```command
package com.pingcap.test;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.Transaction;
import org.hibernate.cfg.Configuration;
public class NewsManager {
  public static void main(String[] args) throws Exception {
    Configuration config = new Configuration().configure();
    SessionFactory factory = config.buildSessionFactory();
    Session session = factory.openSession();
    Transaction transaction = session.beginTransaction();
    News news = new News();
    news.setId(10);
    news.setTitle("UShard示例");
    news.setContent("Hibernate 连接UShard的第一个例子");
    session.save(news);
    transaction.commit();
    session.close();
    factory.close();
    }
}
```

因为 Hibernate 无法控制 SQL 的生成，无法实现对查询 SQL 的优化，大数量下可能会出现性能问题，不建议使用 Hibernate。
