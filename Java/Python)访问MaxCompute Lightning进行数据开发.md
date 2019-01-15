# 使用应用程序(Java/Python)访问MaxCompute Lightning进行数据开发
MaxCompute Lightning是MaxCompute产品的交互式查询服务，支持以PostgreSQL协议及语法连接访问Maxcompute项目，让您使用熟悉的工具以标准 SQL查询分析MaxCompute项目中的数据，快速获取查询结果。
很多开发者希望利用Lightning的特性来开发数据应用，本文将结合示例来介绍Java和Python如何连接访问Lightning进行应用开发(参考时需要替换为您项目所在region的Endpoint及用户认证信息)。
一、Java使用JDBC访问Lightning
示例如下：

```js
import java.sql.*;

public class Main {

    private static Connection connection;

    public static void main(String[] args) throws SQLException {

        String url = "jdbc:postgresql://lightning.cn-shanghai.maxcompute.aliyun.com:443/your_project_name?prepareThreshold=0&sslmode=require";
        String accessId = "<your_maxcompute_access_id>";
        String accessKey = "<your_maxcompute_access_key>";
        String sql = "select * from dual";

        try {
            Connection conn = getCon(url, accessId, accessKey);
            Statement st = conn.createStatement();
            System.out.println("Send Lightning query");
            ResultSet rs = st.executeQuery(sql);
            while (rs.next()) {
                System.out.println(rs.getString(1)+ "\t");
            }
            System.out.println("End Lightning query");
            conn.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    public static Connection getCon(String lightningsHost, String lightningUser, String lightningPwd) {
        try {
            if (connection == null || connection.isClosed()) {
                try {
                    Class.forName("org.postgresql.Driver").newInstance();
                    DriverManager.setLoginTimeout(1);
                    connection = DriverManager.getConnection(lightningsHost, lightningUser, lightningPwd);
                } catch (Exception ex) {
                    ex.printStackTrace();
                }
            }
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return connection;
    }
}
```

二、Java使用druid访问Lightning
1.pom依赖

```js
<dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.0.23</version>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>9.3-1101-jdbc4</version>
        </dependency>
```

2.spring配置
```js
    <bean id="LightningDataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <property name="url" value="jdbc:postgresql://lightning.cn-shanghai.maxcompute.aliyun.com:443/project_name?prepareThreshold=0&sslmode=require”/> <!--替换成自己project所在region的Endpoint—>
        <property name="username" value=“访问用户的Access Key ID"/>
        <property name="password" value="访问用户的Access Key Secret"/>
        <property name="driverClassName" value="org.postgresql.Driver"/>
        <property name="dbType" value="postgresql"/>
        <property name="initialSize" value="1" />  
        <property name="minIdle" value="1" />
        <property name="maxActive" value="5" />  <!—Lightning服务每个project的连接数限制20，所以不要配置过大，按需配置，否则容易出现query_wait_timeout错误 -->
 
        <!--以下两个配置，检测连接有效性，修复偶尔出现create connection holder error错误 -->
        <property name="testWhileIdle" value="true" />
        <property name="validationQuery" value="SELECT 1" />
    </bean>

  <bean class="com.xxx.xxx.LightningProvider">
    <property name="druidDataSource" ref="LightningDataSource"/>
  </bean>
```

3.代码访问

```js
public class LightningProvider {

    DruidDataSource druidDataSource;
    /**
     * 执行sql
     * @param sql
     * @return
     * @throws Exception
     */
    public void execute(String sql) throws SQLException {
        DruidPooledConnection connection = null ;
        Statement st = null;
        try{
            connection = druidDataSource.getConnection();
            st = connection.createStatement();

            ResultSet resultSet = st.executeQuery(sql);
            //对返回值的解析和处理的代码
            //按行处理，每行的数据放到一个map中
            ResultSetMetaData metaData = resultSet.getMetaData();
            int columnCount = metaData.getColumnCount();
            List<LinkedHashMap> rows = Lists.newArrayList();
            while(resultSet.next()){
            LinkedHashMap map = Maps.newLinkedHashMap();
            for(int i=1;i<=columnCount;i++){
                String label = resultSet.getMetaData().getColumnLabel(i);
                map.put(label,resultSet.getString(i));
            }
            rows.add(map);
        }   
        }catch (Exception e){
             e.printStackTrace();
        }finally {
            try {
                if(st!=null) {
                    st.close();
                }
            } catch (SQLException e) {
                e.printStackTrace();
            }

            try {
                if(connection!=null) {
                    connection.close();
                }
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}
```

三、Python使用pyscopg2访问Lightning
示例如下：

```js
#!/usr/bin/env python
# coding=utf-8

import psycopg2
import sys

def query_lightning(lightning_conf, sql):
    """Query data through Lightning by sql

    Args:
        lightning_conf: a map contains settings of 'dbname', 'user', 'password', 'host', 'port'
        sql:  query submit to Lightning

    Returns:
        result: the query result in format of list of rows
    """
    result = None
    conn = None
    conn_str = None
    try:
        conn_str = ("dbname={dbname} "
                    "user={user} "
                    "password={password} "
                    "host={host} "
                    "port={port}").format(**lightning_conf)
    except Exception, e:
        print >> sys.stderr, ("Invalid Lightning' configuration "
                       "{}".format(e))
        sys.exit(1)

    try:
        conn = psycopg2.connect(conn_str)
        conn.set_session(autocommit=True) # This will disable transaction
                                   # started with keyword BEGIN,
                                   # which is currently not
                                   # supported by Lightning’ public service

        cur = conn.cursor()
        # execute Lightning' query
        cur.execute(sql)
        # get result
        result = cur.fetchall()
    except Exception, e:
        print >> sys.stderr, ("Failed to query data through "
                       "Lightning: {}".format(e))
    finally:
        if conn:
            conn.close()

    return result

if __name__ == "__main__":
    # step1. setup configuration
    lightning_conf = {
        "dbname": “your_project_name”,
        "user": "<your_maxcompute_access_id>", 
        "password": "<your_maxcompute_access_key>", 
        "host": "lightning.cn-shanghai.maxcompute.aliyun.com",  #your region lightning endpoint
        "port": 443
    }

    # step2. issue a query
    result = query_lightning(lightning_conf, "select * from test”)
    # step3. print result
    if result:
        for i in xrange(0, len(result)):
            print "Got %d row from Lightning:%s" % (i + 1, result[i])
```

四、Python使用ODBC访问Lightning
您需要现在电脑上安装并和配置odbc驱动。代码示例如下：

```js
import pyodbc
conn_str = (
    "DRIVER={PostgreSQL Unicode};"
    "DATABASE=your_project_name;"
    "UID=your_maxcompute_access_id;"
    "PWD=your_maxcompute_access_key;"
    "SERVER=lightning.cn-shanghai.maxcompute.aliyun.com;" #your region lightning endpoint
    "PORT=443;"
    )
conn = pyodbc.connect(conn_str)
crsr = conn.execute("SELECT * from test”)
row = crsr.fetchone()
print(row)
crsr.close()
conn.close()
```

由于Lightning提供了PostgreSQL兼容的接口，您可以像开发PostgreSQL的应用一样开发Lightning应用程序。

MaxCompute产品官方地址：https://www.aliyun.com/product/odps
注：想了解更多阿里巴巴大数据计算服务MaxCompute，可以加入社群一起交流。
