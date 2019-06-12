# Kafka数据迁移MaxCompute最佳实践.md

前提条件
搭建Kafka集群
进行数据迁移前，您需要保证自己的Kafka集群环境正常。本文使用阿里云EMR服务自动化搭建Kafka集群，详细过程请参见：Kafka 快速入门。

本文使用的EMR Kafka版本信息如下：
EMR版本: EMR-3.12.1
集群类型: Kafka
软件信息: Ganglia 3.7.2 ZooKeeper 3.4.12 Kafka 2.11-1.0.1 Kafka-Manager 1.3.3.16
Kafka集群使用专有网络，区域为华东1（杭州），主实例组ECS计算资源配置公网及内网IP，具体配置如下图所示。

创建MaxCompute 项目
开通MaxCompute服务并创建好项目，本文中在华东1（杭州）区域创建项目bigdata_DOC，同时启动DataWorks相关服务，如下图所示。详情请参见开通MaxCompute。

背景信息
Kafka是一款分布式发布与订阅的消息中间件，具有高性能、高吞量的特点被广泛使用，每秒能处理上百万的消息。Kafka适用于流式数据处理，主要应用于用户行为跟踪、日志收集等场景。

一个典型的Kafka集群包含若干个生产者（Producer）、Broker、消费者（Consumer）以及一个Zookeeper集群。Kafka集群通过Zookeeper管理自身集群的配置并进行服务协同。

Topic是Kafka集群上最常用的消息的集合，是一个消息存储逻辑概念。物理磁盘不存储Topic，而是将Topic中具体的消息按分区（Partition）存储在集群中各个节点的磁盘上。每个Topic可以有多个生产者向它发送消息，也可以有多个消费者向它拉取（消费）消息。

每个消息被添加到分区时，会分配一个offset（偏移量，从0开始编号），是消息在一个分区中的唯一编号。

操作步骤
准备测试表与数据
Kafka集群创建测试数据
为保证您可以顺利登陆EMR集群Header主机及MaxCompute和DataWorks可以顺利和EMR集群Header主机通信，请您首先配置EMR集群Header主机安全组，放通TCP 22及TCP 9092端口。
登录EMR集群Header主机地址
进入EMR Hadoop控制台集群管理 > 主机列表页面，确认EMR集群Header主机地址，并通过SSH连接远程登录。

创建测试Topic
使用kafka-topics.sh --zookeeper emr-header-1:2181/kafka-1.0.1 --partitions 10 --replication-factor 3 --topic testkafka --create命令创建测试所使用的Topic testkafka。您可以使用kafka-topics.sh --list --zookeeper emr-header-1:2181/kafka-1.0.1命令查看已创建的Topic。
[root@emr-header-1 ~]# kafka-topics.sh --zookeeper emr-header-1:2181/kafka-1.0.1 --partitions 10 --replication-factor 3 --topic testkafka --create
Created topic "testkafka".
[root@emr-header-1 ~]# kafka-topics.sh --list --zookeeper emr-header-1:2181/kafka-1.0.1
__consumer_offsets
_emr-client-metrics
_schemas
connect-configs
connect-offsets
connect-status
testkafka
写入测试数据
您可以使用kafka-console-producer.sh --broker-list emr-header-1:9092 --topic testkafka命令模拟生产者向Topic testkafka中写入数据。由于Kafka用于处理流式数据，您可以持续不断的向其中写入数据。 为保证测试结果，建议您写入10条以上的数据。
[root@emr-header-1 ~]# kafka-console-producer.sh --broker-list emr-header-1:9092 --topic testkafka

123
abc

为验证写入数据生效，您可以同时再打开一个SSH窗口，使用kafka-console-consumer.sh --bootstrap-server emr-header-1:9092 --topic testkafka --from-beginning命令模拟消费者，核验数据是否已成功写入Kafka。 如下所示，当数据写入成功时，您可以看到已写入的数据。

[root@emr-header-1 ~]# kafka-console-consumer.sh --bootstrap-server emr-header-1:9092 --topic testkafka --from-beginning
123
abc
创建MaxCompute表
为保证MaxCompute可以顺利接收Kafka数据，请您首先在MaxCompute上创建表。本例中为测试便利，使用非分区表。
登陆DataWorks创建表，详情请参见表管理。

您可以单击DDL模式进行建表，建表语句举例如下。
CREATE TABLE testkafka (

`key` string,
`value` string,
`partition1` string,
`timestamp1` string,
`offset` string,
`t123` string,
`event_id` string,
`tag` string
) ;
其中的每一列，对应于DataWorks数据集成Kafka Reader的默认列，您可以自主命名。详情请参见配置Kafka Reader：
__key__表示消息的key。
__value__表示消息的完整内容 。
__partition__表示当前消息所在分区。
__headers__表示当前消息headers信息。
__offset__表示当前消息的偏移量。
__timestamp__表示当前消息的时间戳。
数据同步
新建自定义资源组
由于当前DataWorks的默认资源组无法完美支持Kafka插件，您需要使用自定义资源组完成数据同步。自定义资源组详情请参见新增任务资源。

在本文中，为节省资源，直接使用EMR集群Header主机作为自定义资源组。完成后，请等待服务器状态变为可用。

新建并运行同步任务
在您的业务流程中右键单击数据集成，选择新建数据集成节点 > 数据同步。

新建数据同步节点后，您需要选择数据来源的数据源为Kafka，数据去向的数据源为ODPS，并且使用默认数据源odps_first。选择数据去向表为您新建的testkafka。完成上述配置后，请点击下图框中的按钮，转换为脚本模式。

脚本配置如下，代码释义请参见配置Kafka Reader。
{

"type": "job",
"steps": [
    {
        "stepType": "kafka",
        "parameter": {
            "server": "47.xxx.xxx.xxx:9092",
            "kafkaConfig": {
                "group.id": "console-consumer-83505"
            },
            "valueType": "ByteArray",
            "column": [
                "__key__",
                "__value__",
                "__partition__",
                "__timestamp__",
                "__offset__",
                "'123'",
                "event_id",
                "tag.desc"
            ],
            "topic": "testkafka",
            "keyType": "ByteArray",
            "waitTime": "10",
            "beginOffset": "0",
            "endOffset": "3"
        },
        "name": "Reader",
        "category": "reader"
    },
    {
        "stepType": "odps",
        "parameter": {
            "partition": "",
            "truncate": true,
            "compress": false,
            "datasource": "odps_first",
            "column": [
                "key",
                "value",
                "partition1",
                "timestamp1",
                "offset",
                "t123",
                "event_id",
                "tag"
            ],
            "emptyAsNull": false,
            "table": "testkafka"
        },
        "name": "Writer",
        "category": "writer"
    }
],
"version": "2.0",
"order": {
    "hops": [
        {
            "from": "Reader",
            "to": "Writer"
        }
    ]
},
"setting": {
    "errorLimit": {
        "record": ""
    },
    "speed": {
        "throttle": false,
        "concurrent": 1,
        "dmu": 1
    }
}
}
您可以通过在Header主机上使用kafka-consumer-groups.sh --bootstrap-server emr-header-1:9092 --list命令查看group.id参数，及消费者的Group名称。
[root@emr-header-1 ~]# kafka-consumer-groups.sh --bootstrap-server emr-header-1:9092 --list
Note: This will not show information about old Zookeeper-based consumers.

_emr-client-metrics-handler-group
console-consumer-69493
console-consumer-83505
console-consumer-21030
console-consumer-45322
console-consumer-14773
以console-consumer-83505为例，您可以根据该参数在Header主机上使用kafka-consumer-groups.sh --bootstrap-server emr-header-1:9092 --describe --group console-consumer-83505命令确认beginOffset及endOffset参数。
[root@emr-header-1 ~]# kafka-consumer-groups.sh --bootstrap-server emr-header-1:9092 --describe --group console-consumer-83505
Note: This will not show information about old Zookeeper-based consumers.
Consumer group 'console-consumer-83505' has no active members.
TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG CONSUMER-ID HOST CLIENT-ID
testkafka 6 0 0 0 - - -
test 6 3 3 0 - - -
testkafka 0 0 0 0 - - -
testkafka 1 1 1 0 - - -
testkafka 5 0 0 0 - - -
完成脚本配置后，请首先切换任务资源组为您刚创建的资源组，然后点击运行。

完成运行后，您可以在运行日志中查看运行结果，如下为成功运行的日志。

结果验证
您可以通过新建一个数据开发任务运行SQL语句，查看当前表中是否已存在从Kafka同步过来的数据。本例中使用select * from testkafka；语句，完成后点击运行即可。

执行结果如下，本例中为保证结果，在testkafka Topic中输入了多条数据，您可以查验是否和您输入的数据一致。
