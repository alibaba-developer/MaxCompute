# 使用MaxCompute Java SDK运行安全相关命令
使用MaxCompute Console的同学，可能都使用过MaxCompute安全相关的命令。官方文档上有详细的MaxCompute<a href="https://help.aliyun.com/document_detail/27924.html?spm=a2c4e.11153940.blogcont686985.16.5ef175734hfMQb">安全指南</a>，并给出了<a href="https://help.aliyun.com/document_detail/27939.html?spm=a2c4e.11153940.blogcont686985.17.5ef175734hfMQb">安全相关语句汇总</a>。

简而言之，<a href="https://help.aliyun.com/document_detail/27935.html?spm=a2c4e.11153940.blogcont686985.18.5ef175734hfMQb">权限管理</a>、<a href="https://help.aliyun.com/document_detail/34604.html?spm=a2c4e.11153940.blogcont686985.19.5ef175734hfMQb">列级别访问控制</a>、<a href="https://help.aliyun.com/document_detail/27937.html?spm=a2c4e.11153940.blogcont686985.20.5ef175734hfMQb">项目空间安全配置</a>以及<a href="https://help.aliyun.com/document_detail/34602.html?spm=a2c4e.11153940.blogcont686985.21.5ef175734hfMQb">跨项目空间的资源分享</a>都属于 MaxCompute 安全命令相关的范畴。

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

需要使用接口 SecurityManager.runQuery() 来运行。<a href="http://www.javadoc.io/doc/com.aliyun.odps/odps-sdk-core/0.29.11-oversea-public?spm=a2c4e.11153940.blogcont686985.22.5ef175734hfMQb&file=0.29.11-oversea-public">详细 SDK Java Doc 戳这里</a>

SecurityManager 类在 odps-sdk-core 中，因此在使用时请添加依赖：

```js
<dependency>
  <groupId>com.aliyun.odps</groupId>
  <artifactId>odps-sdk-core</artifactId>
  <version>0.29.11-oversea-public</version>
</dependency>

```

下面通过一个例子来演示如何通过 MaxCompute Java SDK 来设置表 test_label 列的访问级别为 2，也就是运行命令

```js
SET LABEL 2 TO TABLE test_label(key, value);。
```

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
<div style="text-align:center" align="center">
<img src="/images/MaxCompute Java SDK.png" align="center" />
</div>

程序运行完成后，在 MaxCompute Console 中运行 ｀desc test_lable;` 命令，可以看到 set label 已经生效了。
<div style="text-align:center" align="center">
<img src="/images/MaxCompute Java SDK1.png" align="center" />
</div>

其他安全相关的命令，都可以这样子通过 MaxCompute Java SDK 来运行呢，快来试试吧！
