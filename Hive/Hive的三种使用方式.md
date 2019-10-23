### hive有三种使用方式：CLI、HWI、Thrift

1. **CLI（command-line-interface）命令行界面**
> hive的bin目录下的hive命令

2. **HWI（hive web interface）浏览器**
> 需要相应war包来进行安装

3. **Thrift**
> - 主要解决问题：在代码中连接Hive
- 只要程序中有客户端代码和正确的配置，可以连接到启动了HiveServer或HiveServer2的机器，从而对hive进行操作。
- 因为HiveServer或HiveServer2都是基于Thrift的，所以也被成为Thrift方式。
- HiveServer与HiveServer2：HiveServer不能处理多于一个客户端的并发请求，这是由于HiveServer使用的Thrift接口所导致的限制，不能通过修改HiveServer的代码修正；因此在Hive-0.11.0中重写了HiveServer的代码得到了HiveServer2，解决了这个问题。HiveServer2支持多客户端的并发和认证，为`开放API客户端`如JDBC，ODBC提供了更好的支持。


#### jdbc远程连接HiveServer2
> 启动源数据库：`hive --service metastore &`
开启HiveServer2服务：`hive --server hiveserver2 &` 

api操作Hive：

```java
package com.berg.hive.test1.api;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

import org.apache.log4j.Logger;

/**
 * Hive的JavaApi
 *
 * 启动hive的远程服务接口命令行执行：hive --service hiveserver &
 *
 * @author 汤高
 *
 */
public class HiveJdbcCli {
    //网上写 org.apache.hadoop.hive.jdbc.HiveDriver ,新版本不能这样写
    private static String driverName = "org.apache.hive.jdbc.HiveDriver";

  //这里是hive2，网上其他人都写hive,在高版本中会报错
    private static String url = "jdbc:hive2://master:10000/default";
    private static String user = "hive";
    private static String password = "hive";
    private static String sql = "";
    private static ResultSet res;
    private static final Logger log = Logger.getLogger(HiveJdbcCli.class);

    public static void main(String[] args) {
        Connection conn = null;
        Statement stmt = null;
        try {
            conn = getConn();
            stmt = conn.createStatement();

            // 第一步:存在就先删除
            String tableName = dropTable(stmt);

            // 第二步:不存在就创建
            createTable(stmt, tableName);

            // 第三步:查看创建的表
            showTables(stmt, tableName);

            // 执行describe table操作
            describeTables(stmt, tableName);

            // 执行load data into table操作
            loadData(stmt, tableName);

            // 执行 select * query 操作
            selectData(stmt, tableName);

            // 执行 regular hive query 统计操作
            countData(stmt, tableName);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
            log.error(driverName + " not found!", e);
            System.exit(1);
        } catch (SQLException e) {
            e.printStackTrace();
            log.error("Connection error!", e);
            System.exit(1);
        } finally {
            try {
                if (conn != null) {
                    conn.close();
                    conn = null;
                }
                if (stmt != null) {
                    stmt.close();
                    stmt = null;
                }
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

    private static void countData(Statement stmt, String tableName)
            throws SQLException {
        sql = "select count(1) from " + tableName;
        System.out.println("Running:" + sql);
        res = stmt.executeQuery(sql);
        System.out.println("执行“regular hive query”运行结果:");
        while (res.next()) {
            System.out.println("count ------>" + res.getString(1));
        }
    }

    private static void selectData(Statement stmt, String tableName)
            throws SQLException {
        sql = "select * from " + tableName;
        System.out.println("Running:" + sql);
        res = stmt.executeQuery(sql);
        System.out.println("执行 select * query 运行结果:");
        while (res.next()) {
            System.out.println(res.getInt(1) + "\t" + res.getString(2));
        }
    }

    private static void loadData(Statement stmt, String tableName)
            throws SQLException {
        //目录 ，我的是hive安装的机子的虚拟机的home目录下
        String filepath = "user.txt";
        sql = "load data local inpath '" + filepath + "' into table "
                + tableName;
        System.out.println("Running:" + sql);
         stmt.execute(sql);
    }

    private static void describeTables(Statement stmt, String tableName)
            throws SQLException {
        sql = "describe " + tableName;
        System.out.println("Running:" + sql);
        res = stmt.executeQuery(sql);
        System.out.println("执行 describe table 运行结果:");
        while (res.next()) {
            System.out.println(res.getString(1) + "\t" + res.getString(2));
        }
    }

    private static void showTables(Statement stmt, String tableName)
            throws SQLException {
        sql = "show tables '" + tableName + "'";
        System.out.println("Running:" + sql);
        res = stmt.executeQuery(sql);
        System.out.println("执行 show tables 运行结果:");
        if (res.next()) {
            System.out.println(res.getString(1));
        }
    }

    private static void createTable(Statement stmt, String tableName)
            throws SQLException {
        sql = "create table "
                + tableName
                + " (key int, value string)  row format delimited fields terminated by '\t'";
        stmt.execute(sql);
    }

    private static String dropTable(Statement stmt) throws SQLException {
        // 创建的表名
        String tableName = "testHive";
        sql = "drop table  " + tableName;
        stmt.execute(sql);
        return tableName;
    }

    private static Connection getConn() throws ClassNotFoundException,
            SQLException {
        Class.forName(driverName);
        Connection conn = DriverManager.getConnection(url, user, password);
        return conn;
    }

}
```
