# MaxCompute_JAVA_SDK_开发指南

<h3>背景及目的</h3>
方便和辅助 MaxCompute 开发人员使用 Java SDK 方式进行日常代码的开发工作。

<h3>一、MaxCompute Java SDK用法介绍</h3>
在本文档中，我们仅会对较为常用的MaxCompute核心接口做简短介绍，更多详细信息请参阅SDK Java Doc。

<div style="text-align:center" align="center">
<img src="/images/JAVA_SDK.png" align="center" />
</div>

<h4>1.1、Odps</h4>
MaxCompute SDK的入口，用户通过此类来获取项目空间下的所有对象集合，包括:
Projects、Tables、Resources、Functions、Instances。

用户可以通过传入AliyunAccount实例来构造MaxCompute对象。程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
odps.setDefaultProject("my_project");
for (Table t : odps.tables()) {
....
}
```js
<h4>1.2、Projects</h4>
MaxCompute 中所有项目空间的集合。集合中的元素为Project 。程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps  odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
Project p = odps.projects().get("my_exists");
p.reload();
Map<String, String> properties = prj.getProperties();
...

```

<h4>1.3、SQL Task</h4>
SQL Task 是SDK直接调用MaxCompute SQL的接口，能很方便得运行SQL并获得其返回结果。

从文档可以看到，SQLTask.getResult(i); 返回的是一个List。用户可以循环迭代这个List，获得完整的SQL计算返回结果。不过这个方法有个缺陷，可以参考这里这里提到的SetProject READ_TABLE_MAX_ROW的功能。

目前Select语句返回给客户端的数据条数最大可以调整到1万。也就是说如果在客户端上（包括SQLTask）直接Select，那相当于查询结果上最后加了个Limit N（如果是CREATE TABLE XX AS SELECT或者用INSERT INTO/OVERWRITE TABLE把结果固化到具体的表里就没关系）。

下面代码用于运行、处理SQL任务的接口。可以通过run接口直接运行SQL。
run接口返回Instance 实例，通过Instance获取SQL的运行状态及运行结果。
程序示例如下，仅供参考:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);

String  odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);

Instance instance = SQLTask.run(odps, "my_project", "select ...");
String id = instance.getId();
instance.waitforsuccess();

Set<String> taskNames = instance.getTaskNames();
for (String name : taskNames) {
    TaskSummary summary = instance.getTaskSummary(name);
    String s = summary.getSummaryText();
}
Map<String, String> results = instance.getTaskResults();
Map<String, TaskStatus> taskStatus = instance.getTaskStatus();

for (Entry<String, TaskStatus> status : taskStatus.entrySet()) {
    String result = results.get(status.getKey());
}
```

说明：如果用户想创建表，需要通过SQLTask接口，而不是Table 接口。用户需要将创建表(CREATE
TABLE)的语句传入SQLTask。

<h4>1.4、Instances</h4>
MaxCompute中所有实例(Instance)的集合。

集合中的元素为Instance 。程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
odps.setDefaultProject("my_project");
for (Instance i : odps.instances ()) {
....
}
```

对实例信息的描述，可以通过Instances获取相应的实例。
程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps (account);

String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);

Instance ins = odps.instances().get("instance id");
Date startTime = instance.getStartTime();
Date endTime = instance.getEndTime();
...
Status instanceStatus = instance.getStatus();
String instanceStatusStr = null;
if (instanceStatus == Status.TERMINATED) {
    instanceStatusStr = TaskStatus.Status.SUCCESS.toString();
    Map<String, TaskStatus> taskStatus = instance.getTaskStatus();
    for (Entry<String, TaskStatus> status : taskStatus.entrySet()) {
        if (status.getValue().getStatus() != TaskStatus.Status.SUCCESS) {
            instanceStatusStr = status.getValue().getStatus().toString();
            break;
        }
    }
} else {
    instanceStatusStr = instanceStatus.toString();
}
...
TaskSummary summary = instance.getTaskSummary("instance name");
String s = summary.getSummaryText();

```

<h4>1.5、Tables</h4>

MaxCompute中所有表的集合。集合中的元素为Table。

程序示例如下:

Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
odps.setDefaultProject("my_project");
for (Table t : odps.tables()) {
....
}
对表信息的描述，可以通过Tables获取相应的表。
程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
Table t = odps.tables().get("table name");
t.reload();
Partition part = t.getPartition(new PartitionSpec(tableSpec[1]));
part.reload();
...
```
<h4>1.6、Resources</h4>
MaxCompute中所有资源的集合。集合中的元素为Resource。

程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
odps.setDefaultProject("my_project");
for (Resource r : odps.resources()) {
....
}
```
对资源信息的描述，可以通过Resources获取相应的资源。

程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
Resource r = odps.resources().get("resource name");
r.reload();
if (r.getType() == Resource.Type.TABLE) { 
  TableResource tr = new TableResource(r);
    String tableSource = tr.getSourceTable().getProject() + "." + tr.
    getSourceTable().getName();
    if (tr.getSourceTablePartition() != null) {
        tableSource += " partition(" + tr.getSourceTablePartition().toString() + ")"; 
  }
.... 
}
```

一个创建文件资源的示例:

```js
String projectName = "my_porject";
String source = "my_local_file.txt";
File file = new File(source);
InputStream is = new FileInputStream(file);
FileResource resource = new FileResource();
String name = file.getName();
resource.setName(name);
odps.resources().create(projectName, resource, is);

```

一个创建表资源的示例:

```js
TableResource resource = new TableResource(tableName, tablePrj,partitionSpec);
resource.setName("table_resource_name");
odps.resources().update(projectName, resource);

```

<h4>1.7、Functions</h4>
MaxCompute中所有函数的集合。集合中的元素为Function。

程序示例如下:
```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
odps.setDefaultProject("my_project");
for (Function f : odps.functions()) {
....
}
```
对函数信息的描述，可以通过Functions获取相应的函数。

程序示例如下:

```js
Account account = new AliyunAccount("my_access_id", "my_access_key");
Odps odps = new Odps(account);
String odpsUrl = "<your odps endpoint>";
odps.setEndpoint(odpsUrl);
Function f = odps.functions().get("function name");
List<Resource> resources = f.getResources();

```
一个创建函数的示例:

```js
String resources = "xxx:xxx";
String classType = "com.aliyun.odps.mapred.open.example.WordCount";

ArrayList<String> resourceList = new ArrayList<String>();
for (String r : resources.split(":")) {
    resourceList.add(r);
}

Function func = new Function();
func.setName(name);
func.setClassType(classType);
func.setResources(resourceList);

odps.functions().create(projectName, func);

```

<h4>1.8、Tunnel</h4>
如果需要导出的查询结果就是某张表的全部内容（或者是具体的某个分区的全部内容），可以用SDK Tunnel导出。

程序示例如下：

```js
package daniel.sixiang;

import java.io.IOException;
import java.util.List;

import org.apache.commons.lang3.StringUtils;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import com.aliyun.odps.Column;
import com.aliyun.odps.Odps;
import com.aliyun.odps.PartitionSpec;
import com.aliyun.odps.TableSchema;
import com.aliyun.odps.account.Account;
import com.aliyun.odps.account.AliyunAccount;
import com.aliyun.odps.data.Record;
import com.aliyun.odps.data.RecordReader;
import com.aliyun.odps.tunnel.TableTunnel;
import com.aliyun.odps.tunnel.TableTunnel.DownloadSession;
import com.aliyun.odps.tunnel.TunnelException;
import com.google.common.collect.Lists;

public class ChimaeraCustomerODPSUtils {

    private static Log logger = LogFactory.getLog(ChimaeraCustomerODPSUtils.class);

    private String partition;
    private static final String ACCESS_ID = "---";
    private static final String ACCESS_KEY = "---";
    private static final String project = "---";
    private static final String table = "---";

    private static final String FIELD_admin_mbr_seq = "---";
    private static final String FIELD_state = "---";
    private static final String FIELD_level = "---";
    private static final String FIELD_org_level = "---";
    private static final String FIELD_identity = "---";

    private static String ODPS_URL = "---";
    private static String TUNNEL_URL = "---";


    public static void main(String[] args) {
        ChimaeraCustomerODPSUtils customerODPSUtils = new ChimaeraCustomerODPSUtils("20190115");
        System.out.println(customerODPSUtils.getCustomerFromODPS(1L, 10));
    }

    public ChimaeraCustomerODPSUtils(String dString) {
        this.partition = "ds =" + dString;
    }

    public List<CustomerOdpsDTO> getCustomerFromODPS(Long start, int pageSize) {
        List<CustomerOdpsDTO> custList = Lists.newArrayListWithExpectedSize(pageSize);

        Account account = new AliyunAccount(ACCESS_ID, ACCESS_KEY);
        Odps odps = new Odps(account);
        odps.setEndpoint(ODPS_URL);
        odps.setDefaultProject(project);
        TableTunnel tunnel = new TableTunnel(odps);
        tunnel.setEndpoint(TUNNEL_URL);

        RecordReader recordReader = null;
        try {
            DownloadSession downloadSession;
            PartitionSpec partitionSpec = new PartitionSpec(partition);
            downloadSession = tunnel.createDownloadSession(project, table, partitionSpec);
            long count = downloadSession.getRecordCount();
            logger.warn("Session Status is : " + downloadSession.getStatus().toString());
            if (count != 0) {
                recordReader = downloadSession.openRecordReader(start, pageSize);
                Record record;
                while ((record = recordReader.read()) != null) {
                    CustomerOdpsDTO cust = buildCustomer(record, downloadSession.getSchema());
                    if (cust != null) {
                        custList.add(cust);
                    }
                }
            }
        } catch (Exception e) {
            logger.error("error ", e);
        } finally {
            if (recordReader != null) {
                try {
                    recordReader.close();
                } catch (IOException e) {
                    logger.error("odps_record_reader close error, e=", e);
                }
            }
        }
        return custList;
    }

    private CustomerOdpsDTO buildCustomer(Record record, TableSchema schema) {
        CustomerOdpsDTO cust = new CustomerOdpsDTO();
        for (int i = 0; i < schema.getColumns().size(); i++) {
            Column column = schema.getColumn(i);
            try {
                if (StringUtils.equalsIgnoreCase(column.getName(), FIELD_admin_mbr_seq)) {
                    cust.setAliMemberId(record.getString(i));
                } else if (StringUtils.equalsIgnoreCase(column.getName(), FIELD_state)) {
                    cust.setOper(record.getString(i));
                } else if (StringUtils.equalsIgnoreCase(column.getName(), FIELD_level)) {
                    cust.setNewLevel(record.getString(i));
                } else if (StringUtils.equalsIgnoreCase(column.getName(), FIELD_org_level)) {
                    cust.setOrgLevel(record.getString(i));
                } else if (StringUtils.equalsIgnoreCase(column.getName(), FIELD_identity)) {
                    cust.setIdentity(record.getString(i));
                }
            } catch (Exception e) {
                logger.error("read from column exception.", e);
                return null;
            }
        }
        return cust;
    }

    public Long getCustCountFromODPS() {
        Account account = new AliyunAccount(ACCESS_ID, ACCESS_KEY);
        Odps odps = new Odps(account);
        odps.setEndpoint(ODPS_URL);
        odps.setDefaultProject(project);
        TableTunnel tunnel = new TableTunnel(odps);
        tunnel.setEndpoint(TUNNEL_URL);
        DownloadSession downloadSession;
        try {
            PartitionSpec partitionSpec = new PartitionSpec(partition);
            downloadSession = tunnel.createDownloadSession(project, table, partitionSpec);
            return downloadSession.getRecordCount();
        } catch (TunnelException e) {
            logger.error("TunnelException ,partition = " + partition, e);
            return 0L;
        } finally {
        }
    }
}
package daniel.sixiang;

import java.io.Serializable;

public class CustomerOdpsDTO implements Serializable {

    private static final long serialVersionUID = -9075685189945269963L;

    private String aliMemberId;

    private String oper;

    private String newLevel;

    private String orgLevel;

    private String identity;

    public String getAliMemberId() {
        return aliMemberId;
    }

    public void setAliMemberId(String aliMemberId) {
        this.aliMemberId = aliMemberId;
    }

    public String getOper() {
        return oper;
    }

    public void setOper(String oper) {
        this.oper = oper;
    }

    public String getNewLevel() {
        return newLevel;
    }

    public void setNewLevel(String newLevel) {
        this.newLevel = newLevel;
    }

    public String getOrgLevel() {
        return orgLevel;
    }

    public void setOrgLevel(String orgLevel) {
        this.orgLevel = orgLevel;
    }

    public String getIdentity() {
        return identity;
    }

    public void setIdentity(String identity) {
        this.identity = identity;
    }

    @Override
    public String toString() {
        return "CustomerOdpsDTO [aliMemberId=" + aliMemberId + ", oper=" + oper + ", newLevel=" + newLevel
                + ", orgLevel=" + orgLevel + ", identity=" + identity + "]";
    }
}
```

<h3>二、MaxCompute Java SDK实用小技巧</h3>
<h4>2.1、如何使用 SDK 运行安全相关命令</h4>
使用MaxCompute Console的同学，可能都使用过MaxCompute安全相关的命令。官方文档上有详细的MaxCompute安全指南，并给出了安全相关语句汇总。

简而言之，权限管理、列级别访问控制、项目空间安全配置以及跨项目空间的资源分享都属于MaxCompute安全命令相关的范畴。

再直白一点，以下列关键字开头的命令，都是MaxCompute安全相关操作命令：

```js
GRANT/REVOKE ...
SHOW  GRANTS/ACL/PACKAGE/LABEL/ROLE/PRINCIPALS
SHOW  PRIV/PRIVILEGES
LIST/ADD/REOVE  USERS/ROLES/TRUSTEDPROJECTS
DROP/CREATE   ROLE
CLEAR EXPIRED  GRANTS
DESC/DESCRIBE   ROLE/PACKAGE
CREATE/DELETE/DROP  PACKAGE
ADD ... TO  PACKAGE
REMOVE ... FROM  PACKAGE
ALLOW/DISALLOW  PROJECT
INSTALL/UNINSTALL  PACKAGE
LIST/ADD/REMOVE   ACCOUNTPROVIDERS
SET  LABLE  ...

```

那么，这些能在 MaxCompute Console 上运行的命令，该如何使用 MaxCompute Java SDK 运行呢？它们是与 SQL 一样通过创建 instance 的方式来运行吗？

答案：不可以，这些命令不是 SQL ， 不可以通过 SQL Task 来运行。

需要使用接口 SecurityManager.runQuery() 来运行。详细 SDK Java Doc 戳这里

```js
SecurityManager 类在 odps-sdk-core 中，因此在使用时请添加依赖：

<dependency>
  <groupId>com.aliyun.odps</groupId>
  <artifactId>odps-sdk-core</artifactId>
  <version>0.29.11-oversea-public</version>
</dependency>

```

下面通过一个例子来演示如何通过 MaxCompute Java SDK 来设置表 test_label 列的访问级别为 2，也就是运行命令：SET LABEL 2 TO TABLE test_label(key, value);

```js
import com.aliyun.odps.Column;
import com.aliyun.odps.Odps;
import com.aliyun.odps.OdpsException;
import com.aliyun.odps.OdpsType;
import com.aliyun.odps.TableSchema;
import com.aliyun.odps.account.Account;
import com.aliyun.odps.account.AliyunAccount;
import com.aliyun.odps.security.SecurityManager;

public class test {
  public static void main(String [] args) throws OdpsException {
    try {
      // init odps
      Account account = new AliyunAccount("<your_accessid>", "<your_accesskey>");
      Odps odps = new Odps(account);
      odps.setEndpoint("http://service-corp.odps.aliyun-inc.com/api");
      odps.setDefaultProject("<your_project>");

      // create test table
      // if u already have a table, skip this
      TableSchema schema = new TableSchema();
      schema.addColumn(new Column("key", OdpsType.STRING));
      schema.addColumn(new Column("value", OdpsType.BIGINT));
      odps.tables().create("test_label", schema);

      // set label 2 to table columns
      SecurityManager securityManager = odps.projects().get().getSecurityManager();
      String res = securityManager.runQuery("SET LABEL 2 TO TABLE test_label(key, value);", false);
      System.out.println(res);
    } catch (OdpsException e) {
      e.printStackTrace();
    }
  }
}

```
运行结果：

```js
undefined

```
程序运行完成后，在 MaxCompute Console 中运行 ‘desc test_lable;’ 命令，可以看到 set label 已经生效了。

```js
undefined

```
其他安全相关的命令，都可以这样通过 MaxCompute Java SDK 来运行！

<h4>2.2、如何使用 SDK 生成 instance Logview</h4>

<h4>场景</h4>

A: “用 MaxCompute Java SDK 跑作业，为什么卡住不动了？”

ME: “有 Logview 吗？发来看下。”

A: “没有，我用的是SDK，没Logview。”

用户 A 的问题在于没有 instance 的 logview，导致无法追踪 instance 的运行过程。

通常用户在创建 instance 后会调用 instance.waitForSuccess() 来等待作业运行完成，一旦作业耗时巨大，程序就卡在这一步了，此时如果有 logview ，就能查看追踪查看作业等待的具体原因了。

【 怎么使用 MaxCompute Java SDK 生成 instance Logview 】

答案很简单， MaxCompute Java SDK 提供了 logview 接口，详情可查看 SDK Java Doc

```js
RunningJob rj = JobClient.runJob(job);
com.aliyun.odps.Instance instance = SessionState.get().getOdps().instances().get(rj.getInstanceID());
String logview = SessionState.get().getOdps().logview().generateLogView(instance, 7 * 24);
System.out.println(logview);

```

两个参数： instance 对象，logview token 超时时间 (单位：小时)

再次提醒用户，在使用 SDK 的时候，请为每个 instance 记录 Logview，一旦遇到问题可快速追踪。

当然如果改代码很麻烦，那还有一个绝招：在 MaxCompute Console 中使用 wait 命令也可以得到Logview。

<h4>2.3、如何使用 SDK 输出错误日志</h4>

<h4>场景</h4>

B ：“用 MaxCompute Java SDK 访问 Table，为什么卡住半天没反应？”

ME：“卡在哪一行了？”

B："就 RestClient retry 然后卡住了。"

用户 B 的问题在于 sdk 的 Restclient 本身有重试机制，从表面来看就是卡住了，没有任何输出。

如果在每次重试的时候都输出错误，就可以快速定位问题节约时间了。我已经遇到好几个公共云用户因为缺包导致一直卡住几分钟才丢出异常，严重影响了工作效率。

【 能不能在每次重试的时候，都把错误输出呢？】

当然可以。MaxCompute Java SDK 提供了抽象类 RetryLogger 详情可查看 SDK Java Doc

```js
public static abstract class RetryLogger {
    /**
     * 当 RestClent 发生重试前的回调函数
     *
     * @param e
     *     错误异常
     * @param retryCount
     *     重试计数
     * @param retrySleepTime
     *     下次需要的重试时间
     */
    public abstract void onRetryLog(Throwable e, long retryCount, long retrySleepTime);
}
```

只需实现一个自己的 RetryLogger 子类，然后在初始化 odps 对象的时候使用 odps.getRestClient().setRetryLogger(new UserRetryLogger()); 就可以将日志输出。

一个典型的实现如下：

```js
// init odps
odps.getRestClient().setRetryLogger(new UserRetryLogger());

// your retry logger
public class UserRetryLogger extends RetryLogger {

    @Override
    public void onRetryLog(Throwable e, long retryCount, long sleepTime) {
      if (e != null && e instanceof OdpsException) {
        String requestId = ((OdpsException) e).getRequestId();
        if (requestId != null) {
          System.err.println(String.format(
              "Warning: ODPS request failed, requestID:%s, retryCount:%d, will retry in %d seconds.",
              requestId, retryCount, sleepTime));
          return;
        }
      }
      System.err.println(String.format(
          "Warning: ODPS request failed:%s, retryCount:%d, will retry in %d seconds.", e.getMessage(),retryCount,
          sleepTime));
    }
  }
  
```
<h4>2.4、如何设置SQL参数</h4>
使用 DataWorks 或者 MaxCompute Console 提交SQL 经常需要设置一些flag，比如：
```js
set odps.sql.type.system.odps2=true;
```
那么用SDK 提交 SQL的时候，要怎么设置呢？ 

特别注意，当使用SDK提交SQL，把set直接放到sql query中是不生效的，会报错的。

下面以Java SDK为例说明：

```js
// 构造 SQLTask 对象
SQLTask task = new SQLTask();
task.setName("foobar");
task.setQuery("select ...");

// 设置flag
Map<String, String> settings = new HashMap<>();
settings.put("odps.sql.type.system.odps2", "true");
...  // set other flags
task.setProperty("settings", new JSONObject(settings).toString());  // 这里是关键：将flags对应的json string设置到settings property中

// 执行
Instance instance = odps.instances().create(task);

```

<h4>2.5、SDK如何提交多条SQL</h4>
我们知道DataWorks中一个node可以包含多条SQL，而MaxCompute Console也可以通过odpscmd -f 来提交一个包含多条语句的文件。实际上这些提交方式，都是通过一个预处理，来把语句按照分号来拆分成多条语句，然后再一条一条运行的。

这个拆分的过程是在MaxCompute Console做的，而用户通过SDK提交作业，就需要自己做这个拆分。

一次提交一条SQL，如果一次提交了多句SQL，是会报错的。

MaxCompute是不是支持一次性提交多条SQL来执行呢？答案是肯定的，我们提供了脚本模式的功能。

脚本模式下，所有SQL是一次性提交，一次性执行的。注意用脚本模式提交，相当于让不同语句之间产生了关系，所有SQL作为一个整体执行，这个和MaxCompute Console 一句一句顺序执行的模式是完全不同的。同时也会引入一些限制，比如最多只有一个create table as 语句，settings和ddl必须放在开头等。如果不能满足这个条件，那么还是需要用户自己一条一条地拆分后提交。

脚本模式的正常脚本经常需要涉及到多条语句之间的关联操作（比如 table variable定义 和 table variable 引用是在不同的SQL语句中）。这时候，将SQL一句句拆开执行是不行的，因为定义variable的语句是在前一条语句执行的，和引用variable的语句完全是两个上下文，会报错 "variable xx cannot be resolved" 。 这时候，如果用别的客户端提交查询，也需要以脚本模式的方式提交

- MaxCompute Console 通过 -f 参数提交的查询是会将脚本一句一句拆分开的，如果也想要使用脚本模式来提交查询，那么需要使用-s 参数，如 odpscmd -s foo.sql 。注意目前MaxCompute Console 的交互式模式是不能提交脚本模式的查询的。
- DataWorks里面用"ODPS SQL"节点提交作业，是一句一句顺序执行的。按照脚本模式，需要在创建节点的时候，选择类型为 "ODPS Script"
- MaxCompute Studio 对脚本模式有非常强大的支持，推荐使用。
<h4>2.6、如何使用SQLTask+Tunnel实现大数据量导出</h4>
SQLTask不能处理超过1万条记录，但是Tunnel刚好可以，两者存在互补。所以可以基于两者完成大数据的导出。

以下用一个代码的例子来实现：

```js
 private static final String accessId = "userAccessId";
    private static final String accessKey = "userAccessKey";
    private static final String endPoint = "http://service.odps.aliyun.com/api";
    private static final String project = "userProject";
    private static final String sql = "userSQL";
    private static final String table = "Tmp_"+UUID.randomUUID().toString().replace("-", "_");//其实也就是随便找了个随机字符串作为临时表的名字
    private static final Odps odps = getOdps();

    public static void main(String[] args) {
        System.out.println(table);
        runSql();
        tunnel();
    }

    /*
     * 把SQLTask的结果下载过来
     * */
    private static void tunnel() {
        TableTunnel tunnel = new TableTunnel(odps);
        try {
            DownloadSession downloadSession = tunnel.createDownloadSession(
                    project, table);
            System.out.println("Session Status is : "
                    + downloadSession.getStatus().toString());
            long count = downloadSession.getRecordCount();
            System.out.println("RecordCount is: " + count);
            RecordReader recordReader = downloadSession.openRecordReader(0,
                    count);
            Record record;
            while ((record = recordReader.read()) != null) {
                consumeRecord(record, downloadSession.getSchema());
            }
            recordReader.close();
        } catch (TunnelException e) {
            e.printStackTrace();
        } catch (IOException e1) {
            e1.printStackTrace();
        }
    }

    /*
     * 保存这条数据
     * 数据量少的话直接打印后拷贝走也是一种取巧的方法。实际场景可以用Java.io写到本地文件，或者写到远端数据等各种目标保存起来。
     * */
    private static void consumeRecord(Record record, TableSchema schema) {
        System.out.println(record.getString("username")+","+record.getBigint("cnt"));
    }

    /*
     * 运行SQL，把查询结果保存成临时表，方便后面用Tunnel下载
     * 这里保存数据的lifecycle为1天，所以哪怕删除步骤出了问题，也不会太浪费存储空间
     * */
    private static void runSql() {
        Instance i;
        StringBuilder sb = new StringBuilder("Create Table ").append(table)
                .append(" lifecycle 1 as ").append(sql);
        try {
            System.out.println(sb.toString());
            i = SQLTask.run(getOdps(), sb.toString());
            i.waitForSuccess();

        } catch (OdpsException e) {
            e.printStackTrace();
        }
    }

    /*
     * 初始化MaxCompute连接信息
     * */
    private static Odps getOdps() {
        Account account = new AliyunAccount(accessId, accessKey);
        Odps odps = new Odps(account);
        odps.setEndpoint(endPoint);
        odps.setDefaultProject(project);
        return odps;
    }
```
