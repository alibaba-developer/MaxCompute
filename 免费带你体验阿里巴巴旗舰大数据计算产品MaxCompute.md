# 免费带你体验阿里巴巴旗舰大数据计算产品MaxCompute
<h3>什么是MaxCompute？</h3>

众所周知，MaxCompute是阿里云推出的承载EB级的数据存储能力，百PB级的单日计算能力，公共云覆盖国内外十几个国家和地区，专有云包含城市大脑在内部署超过100+套的阿里巴巴的统一计算平台。官方地址：https://www.aliyun.com/product/odps
<div style="text-align:center" align="center">
<img src="/images/大数据计算产品1.png" align="center" />
</div>

MaxCompute是真正为大数据而生的企业级云计算产品，其核心是一项基础服务(PaaS)，用于对海量数据进行高性能的分析处理，数据规模越大，计算性能越卓越，在大规模批量计算下性能远超Hadoop Hive，甚至超越了Spark、Impala；



单纯从技术上来看，MaxCompute提供了一个在云端的SQL、MapReduce、Graph服务，提供对海量数据的批量计算能力；
<div style="text-align:center" align="center">
<img src="/images/大数据计算产品2.png" align="center" />
</div>
<div style="text-align:center" align="center">
<img src="/images/大数据计算产品3.png" align="center" />
</div>

另外，MaxCompute是基于Serverless架构实现的服务，从成本最优化、运维便利性、业务敏捷度三个方面，帮助企业升维核心竞争力；



很多开发者都想试用一下MaxCompute，但又担心这么一款旗舰产品价格不菲，今天小编带大家做一次免费试用；



<h3>准备工作</h3>

1、提前一周申请10元代金券，代金券申请地址：https://i.aliyun.com/inviteapply?agent_id=183 



2、有开发者会问到，10元代金券能干什么？买台最低配置的虚机测试还得上几百元，更别提大数据产品了。首先带大家了解一下MaxCompute计费规则：MaxCompute每月每1GB数据的存储费用是2分钱，每月每处理1GB数据SQL收费0.3元；配套的离线/实时数据同步工具如Datax、Datahub免费，提供大数据开发调度、数据质量、血缘管理的Dataworks也免费。



3、假如你有3G数据，我们看看是如何花费10元；

3.1、3G存储，5张表，每张表平均600M，MaxCompute整体压缩后约1G数据，每天存储开销2分钱；

3.2、从本地上传到MaxCompute，不收费用；

3.3、设置一个Pipeline，每天跑5个SQL

一类SQL是简单查询-单表查询，每次SQL查询开销0.2元

一类SQL是复杂查询-Join 2张表，每次SQL查询开销1.2元；

我们完成上述任务总共需要3.2元，计算方法就是：2次复杂SQL*1.2元+3次简单SQL*0.2+存储0.02分钱；同样任务我们可以通过Dataworks配置周期调度，跑三天，花销9.4元；



4、所以，我们可以通过领用10元代金券，免费完成很多数据任务。



<h3>操作流程</h3>

开通MaxCompute后付费服务->通过Dataworks创建Project->通过tunnel上传你的数据集->通过SQL运行你的Hello World。



1、开通MaxCompute后付费服务

https://help.aliyun.com/document_detail/58226.html



2、通过Dataworks创建Project

https://help.aliyun.com/document_detail/27815.html



3、通过tunnel上传你的数据集

3.1、安装并配置客户端

https://help.aliyun.com/document_detail/27804.html

3.2、创建表

https://help.aliyun.com/document_detail/27808.html

3.3、导入数据或使用公开数据集

导入数据 https://help.aliyun.com/document_detail/27809.html

使用数据集 https://yq.aliyun.com/articles/89763  注意：有些数据集上T，先通过desc 表名获取数据大小，切勿全表扫描；



4、通过SQL运行你的Hello World

https://help.aliyun.com/document_detail/27810.html



<h3>小结</h3>

通过免费试用，开发者可以感受到将所有精力都放在业务上，节省了自建平台在学习成本、开发成本、管理成本、投入机房资源和运维成本的总成本，应用开发效率有很大提高。



注：想了解更多阿里巴巴大数据计算服务MaxCompute，可以加入社群一起交流。

<div style="text-align:center" align="center">
<img src="/images/大数据计算产品4.png" align="center" />
</div>
