如何使用Tunnel SDK上传/下载MaxCompute复杂类型数据
基于Tunnel SDK如何上传复杂类型数据到MaxCompute？首先介绍一下MaxCompute复杂数据类型：

<h4>复杂数据类型</h4>
MaxCompute采用基于ODPS2.0的SQL引擎，丰富了对复杂数据类型类型的支持。MaxCompute支持ARRAY, MAP, STRUCT类型，并且可以任意嵌套使用并提供了配套的内建函数。

<div style="text-align:center" align="center">
<img src="/images/Tunnel SDK上传下载1.png" align="center" />
</div>

<h4>复杂类型构造与操作函数</h4>

<div style="text-align:center" align="center">
<img src="/images/Tunnel SDK上传下载2.png" align="center" />
</div>
<div style="text-align:center" align="center">
<img src="/images/Tunnel SDK上传下载3.png" align="center" />
</div>
<h4>Tunnel SDK 介绍</h4>
Tunnel 是 ODPS 的数据通道，用户可以通过 Tunnel 向 ODPS 中上传或者下载数据。</br>
TableTunnel 是访问 ODPS Tunnel 服务的入口类，仅支持表数据（非视图）的上传和下载。</br>
对一张表或 partition 上传下载的过程，称为一个session。session 由一或多个到 Tunnel RESTful API 的 HTTP Request 组成。</br>
session 用 session ID 来标识，session 的超时时间是24小时，如果大批量数据传输导致超过24小时，需要自行拆分成多个 session。</br>
数据的上传和下载分别由 TableTunnel.UploadSession 和 TableTunnel.DownloadSession 这两个会话来负责。</br>
TableTunnel 提供创建 UploadSession 对象和 DownloadSession 对象的方法.</br>


<h4>典型表数据上传流程：</h4>
1) 创建 TableTunnel</br>
2) 创建 UploadSession</br>
3) 创建 RecordWriter,写入 Record</br>
4）提交上传操作</br>

<h4>典型表数据下载流程：</h4>
1) 创建 TableTunnel</br>
2) 创建 DownloadSession</br>
3) 创建 RecordReader,读取 Record</br>

<h4>基于Tunnel SDK构造复杂类型数据</h4>
代码示例：

```js
            RecordWriter recordWriter = uploadSession.openRecordWriter(0);
      ArrayRecord record = (ArrayRecord) uploadSession.newRecord();

      // prepare data
      List arrayData = Arrays.asList(1, 2, 3);
      Map<String, Long> mapData = new HashMap<String, Long>();
      mapData.put("a", 1L);
      mapData.put("c", 2L);

      List<Object> structData = new ArrayList<Object>();
      structData.add("Lily");
      structData.add(18);

      // set data to record
      record.setArray(0, arrayData);
      record.setMap(1, mapData);
      record.setStruct(2, new SimpleStruct((StructTypeInfo) schema.getColumn(2).getTypeInfo(),
                                           structData));

      // write the record
      recordWriter.write(record);
 ```
 
<h4>从MaxCompute下载复杂类型数据</h4>
代码示例：

```js
            RecordReader recordReader = downloadSession.openRecordReader(0, 1);

      // read the record
      ArrayRecord record1 = (ArrayRecord)recordReader.read();

      // get array field data
      List field0 = record1.getArray(0);
      List<Long> longField0 = record1.getArray(Long.class, 0);

      // get map field data
      Map field1 = record1.getMap(1);
      Map<String, Long> typedField1 = record1.getMap(String.class, Long.class, 1);

      // get struct field data
      Struct field2 = record1.getStruct(2);
```

<h4>运行实例</h4>
完整代码如下：

```js
import java.io.IOException;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import com.aliyun.odps.Odps;
import com.aliyun.odps.PartitionSpec;
import com.aliyun.odps.TableSchema;
import com.aliyun.odps.account.Account;
import com.aliyun.odps.account.AliyunAccount;
import com.aliyun.odps.data.ArrayRecord;
import com.aliyun.odps.data.RecordReader;
import com.aliyun.odps.data.RecordWriter;
import com.aliyun.odps.data.SimpleStruct;
import com.aliyun.odps.data.Struct;
import com.aliyun.odps.tunnel.TableTunnel;
import com.aliyun.odps.tunnel.TableTunnel.UploadSession;
import com.aliyun.odps.tunnel.TableTunnel.DownloadSession;
import com.aliyun.odps.tunnel.TunnelException;
import com.aliyun.odps.type.StructTypeInfo;

public class TunnelComplexTypeSample {

  private static String accessId = "<your access id>";
  private static String accessKey = "<your access Key>";
  private static String odpsUrl = "<your odps endpoint>";
  private static String project = "<your project>";

  private static String table = "<your table name>";

  // partitions of a partitioned table, eg: "pt=\'1\',ds=\'2\'"
  // if the table is not a partitioned table, do not need it
  private static String partition = "<your partition spec>";

  public static void main(String args[]) {
    Account account = new AliyunAccount(accessId, accessKey);
    Odps odps = new Odps(account);
    odps.setEndpoint(odpsUrl);
    odps.setDefaultProject(project);

    try {
      TableTunnel tunnel = new TableTunnel(odps);
      PartitionSpec partitionSpec = new PartitionSpec(partition);

      // ---------- Upload Data ---------------
      // create upload session for table
      // the table schema is {"col0": ARRAY<BIGINT>, "col1": MAP<STRING, BIGINT>, "col2": STRUCT<name:STRING,age:BIGINT>}
      UploadSession uploadSession = tunnel.createUploadSession(project, table, partitionSpec);
      // get table schema
      TableSchema schema = uploadSession.getSchema();

      // open record writer
      RecordWriter recordWriter = uploadSession.openRecordWriter(0);
      ArrayRecord record = (ArrayRecord) uploadSession.newRecord();

      // prepare data
      List arrayData = Arrays.asList(1, 2, 3);
      Map<String, Long> mapData = new HashMap<String, Long>();
      mapData.put("a", 1L);
      mapData.put("c", 2L);

      List<Object> structData = new ArrayList<Object>();
      structData.add("Lily");
      structData.add(18);

      // set data to record
      record.setArray(0, arrayData);
      record.setMap(1, mapData);
      record.setStruct(2, new SimpleStruct((StructTypeInfo) schema.getColumn(2).getTypeInfo(),
                                           structData));

      // write the record
      recordWriter.write(record);

      // close writer
      recordWriter.close();

      // commit uploadSession, the upload finish
      uploadSession.commit(new Long[]{0L});
      System.out.println("upload success!");

      // ---------- Download Data ---------------
      // create download session for table
      // the table schema is {"col0": ARRAY<BIGINT>, "col1": MAP<STRING, BIGINT>, "col2": STRUCT<name:STRING,age:BIGINT>}
      DownloadSession downloadSession = tunnel.createDownloadSession(project, table, partitionSpec);
      schema = downloadSession.getSchema();

      // open record reader, read one record here for example
      RecordReader recordReader = downloadSession.openRecordReader(0, 1);

      // read the record
      ArrayRecord record1 = (ArrayRecord)recordReader.read();

      // get array field data
      List field0 = record1.getArray(0);
      List<Long> longField0 = record1.getArray(Long.class, 0);

      // get map field data
      Map field1 = record1.getMap(1);
      Map<String, Long> typedField1 = record1.getMap(String.class, Long.class, 1);

      // get struct field data
      Struct field2 = record1.getStruct(2);

      System.out.println("download success!");
    } catch (TunnelException e) {
      e.printStackTrace();
    } catch (IOException e) {
      e.printStackTrace();
    }

  }
}
```
