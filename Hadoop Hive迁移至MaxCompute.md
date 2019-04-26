# Hadoop Hive迁移至MaxCompute.md

本文向您详细介绍如何将 Hadoop Hive 数据迁移到阿里云MaxCompute大数据计算服务上。

<b>一、环境准备</b>
<b>1.1、Hadoop集群环境</b>
在进行 Hadoop Hive 数据迁移前，您需要保证自己的Hadoop集群环境正常。本文使用的Hadoop环境：

- HDFS 2.8.5
- YARN 2.8.5
- Hive 3.1.1
- Ganglia 3.7.2
- Spark 2.3.2
- HUE 4.1.0
- Zeppelin 0.8.0
- Tez 0.9.1
- Sqoop 1.4.7
- Pig 0.14.0
- Knox 1.1.0
- ApacheDS 2.0.0


<b>1.2、Hadoop Hive数据准备</b>
Hive脚本：

CREATE TABLE IF NOT EXISTS hive_sale(
  create_time timestamp,
  category STRING,
  brand STRING,
  buyer_id STRING,
  trans_num BIGINT,
  trans_amount DOUBLE,
  click_cnt BIGINT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' lines terminated by '\n';

insert into hive_sale values
('2019-04-14','外套','品牌A','lilei',3,500.6,7),
('2019-04-15','生鲜','品牌B','lilei',1,303,8),
('2019-04-16','外套','品牌C','hanmeimei',2,510,2),
('2019-04-17','卫浴','品牌A','hanmeimei',1,442.5,1),
('2019-04-18','生鲜','品牌D','hanmeimei',2,234,3),
('2019-04-19','外套','品牌B','jimmy',9,2000,7),
('2019-04-20','生鲜','品牌A','jimmy',5,45.1,5),
('2019-04-21','外套','品牌E','jimmy',5,100.2,4),
('2019-04-22','生鲜','品牌G','peiqi',10,5560,7),
('2019-04-23','卫浴','品牌F','peiqi',1,445.6,2),
('2019-04-24','外套','品牌A','ray',3,777,3),
('2019-04-25','卫浴','品牌G','ray',3,122,3),
('2019-04-26','外套','品牌C','ray',1,62,7);


<b>登录Hadoop集群，创建Hive SQL脚本，执行Hive命令完成脚本初始化：</b>

hive -f hive_data.sql

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute1.png" align="center" />
</div>
</br>

<b>查询Table：</b>

hive -e 'show tables';
hive -e 'select * from hive_sale';

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute2.png" align="center" />
</div>
</br>

<b>1.3、MaxCompute环境</b>
开通MaxCompute，请参考：https://help.aliyun.com/document_detail/58226.html

安装并配置MaxCompute客户端，请参考：https://help.aliyun.com/document_detail/27804.html

<b>1.4、MaxCompute Create Table</b>

CREATE TABLE IF NOT EXISTS maxcompute_sale(
  create_time STRING,
  category STRING,
  brand STRING,
  buyer_id STRING,
  trans_num BIGINT,
  trans_amount DOUBLE,
  click_cnt BIGINT
);

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute3.png" align="center" />
</div>
</br>

在建表过程中，需要考虑到Hive数据类型 与 MaxCompute数据类型的映射，可参考：
https://help.aliyun.com/document_detail/54081.html

上述通过odpscmd命令行工具完成，命令行工具安装和配置请参考：https://help.aliyun.com/document_detail/27804.html

注意：MaxCompute2.0支持的数据类型，包括基本数据类型和复杂类型，详见：
https://help.aliyun.com/document_detail/27821.html

<b>二、Hive迁移MaxCompute操作</b>
<b>2.1、通过Tunnel文件上传</b>
<b>2.1.1、生成Hive数据文件</b>

进入Hive，执行SQL语句，下面我们将导出到本地的数据按行以逗号进行分隔：

insert overwrite local directory  '/home/sixiang/' row format delimited fields terminated by ',' select * from hive_sale;

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute4.png" align="center" />
</div>
</br>

查看数据文件：

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute5.png" align="center" />
</div>
</br>

<b>2.1.2、执行Tunnel命令完成文件上传</b>

进入MaxCompute控制台，执行Tunnel upload命令完成数据上传：

tunnel upload /home/sixiang/000000_0 daniel.maxcompute_sale;

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute6.png" align="center" />
</div>
</br>

<b>2.2、通过DataWorks-数据集成上传</b>
<b>2.2.1、新建自定义资源组</b>

MaxCompute所处的网络环境与Hadoop集群中的DataNode网络通常不可达，可通过自定义资源组的方式，将DataWorks同步任务运行在Hadoop集群的Master节点上（Hadoop集群内Master和DataNode网络可通）

<b>查看Hadoop集群DataNode：执行hadoop dfsadmin –report 命令查看</b>

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute7.png" align="center" />
</div>
</br>

上图可以看到，DataNode为内网地址，很难与DataWorks默认资源组互通，所以需要设置自定义资源组，将Master Node设置为执行DataWorks数据同步任务的节点。

进入DataWorks数据集成页面，选择资源组，点击新增资源组，如下图所示：

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute8.png" align="center" />
</div>
</br>

在添加服务器步骤中，需要输入机器的UUID和IP等信息，机器IP需填写MasterNode公网IP（内网IP可能不通）。

机器的UUID需要进入MasterNode管理终端，通过命令dmidecode | grep UUID获取，如下所示：

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute9.png" align="center" />
</div>
</br>

完成添加服务器后，需保证Master Node与DataWorks网络可联通；按照提示安装自定义资源组agent，观察当前状态为可用，说明新增自定义资源组成功。

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute10.png" align="center" />
</div>
</br>

<b>2.2.3、新建数据源</b>

DataWorks新建项目后，默认设置自己为数据源odps_first；因此我们只需添加Hadoop集群数据源，在DataWorks数据集成页面，点击数据源 > 新增数据源，在弹框中选择HDFS类型的数据源。

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute11.png" align="center" />
</div>
</br>

在弹出窗口中，填写“数据源名称”、“DefaultFS”

如果Hadoop集群为HA集群，则此处地址为hdfs://IP:8020，如果Hadoop集群为非HA集群，则此处地址为hdfs://IP:9000。在本文中，Hadoop机器与DataWorks通过公网连接，因此此处填写公网IP。

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute12.png" align="center" />
</div>
</br>

完成配置后，点击测试连通性，如果提示“测试连通性成功”，则说明数据源添加成功。

<b>2.2.4、配置数据同步任务</b>

在DataWorks“数据开发”页面，选择新建数据集成节点-数据同步，在导入模板弹窗选择数据源类型如下：

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute13.png" align="center" />
</div>
</br>

具体脚本如下：

{
    "configuration": {
        "reader": {
            "plugin": "hdfs",
            "parameter": {
                "path": "/user/hive/warehouse/hive_sale/",
                "datasource": "hadoop_to_odps",
                "column": [
                    {
                        "index": 0,
                        "type": "string"
                    },
                    {
                        "index": 1,
                        "type": "string"
                    },
                    {
                        "index": 2,
                        "type": "string"
                    },
                    {
                        "index": 3,
                        "type": "string"
                    },
                    {
                        "index": 4,
                        "type": "long"
                    },
                    {
                        "index": 5,
                        "type": "double"
                    },
                    {
                        "index": 6,
                        "type": "long"
                    }
                ],
                "defaultFS": "hdfs://xxx.xxx.xxx.xxx:9000",
                "fieldDelimiter": ",",
                "encoding": "UTF-8",
                "fileType": "text"
            }
        },
        "writer": {
            "plugin": "odps",
            "parameter": {
                "partition": "",
                "truncate": false,
                "datasource": "odps_first",
                "column": [
                    "create_time",
                    "category",
                    "brand",
                    "buyer_id",
                    "trans_num",
                    "trans_amount",
                    "click_cnt"
                ],
                "table": "maxcompute_sale"
            }
        },
        "setting": {
            "errorLimit": {
                "record": "1000"
            },
            "speed": {
                "throttle": false,
                "concurrent": 1,
                "mbps": "1",
                "dmu": 1
            }
        }
    },
    "type": "job",
    "version": "1.0"
}
其中，path参数为数据在Hadoop集群中存放的位置，您可以在登录Master Node后，使用
hdfs dfs –ls /user/hive/warehouse/hive_sale 命令确认。

完成配置后，点击运行。如果提示任务运行成功，则说明同步任务已完成。

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute14.png" align="center" />
</div>
</br>

<b>2.2.5、验证结果</b>

在DataWorks数据开发/临时查询，执行select * FROM hive_sale 验证结果，如下图所示：

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute15.png" align="center" />
</div>
</br>

当然，也可以通过在odpscmd命令行工具中SQL查询表结果：

<div style="text-align:center" align="center">
<img src="/images/Hadoop Hive迁移至MaxCompute16.png" align="center" />
</div>
</br>
