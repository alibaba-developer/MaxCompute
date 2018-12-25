# MaxCompute小文件问题优化方案
<h3>小文件背景知识</h3>
<h4>小文件定义</h4>
分布式文件系统按块Block存放，文件大小比块大小小的文件（默认块大小为64M），叫做小文件。

如何判断存在小文件数量多的问题
<h4>查看文件数量</h4>

```js
desc extended + 表名
image
```
<div style="text-align:center" align="center">
<img src="/images/小文件1.png" align="center" />
</div>

<h4>判断小文件数量多的标准</h4>
1、非分区表，表文件数达到1000个，文件平均大小小于64M</br>
2、分区表: a) 单个分区文件数达到1000个，文件平均大小小于64M，</br>
               b) 整个非分区表分区数达到五万 （系统限制为6万）</br>

<h4>产生小文件数量多的主要原因</h4>
1、表设计不合理导致：分区多导致文件多，比如按天按小时按业务单元（假如有6个业务单元BU）分区，那么一年下来，分区数将会达到365246=52560。</br>
2、在使用Tunnel、Datahub、Console等数据集成工具上传上传数据时，频繁Commit，写入表（表分区）使用不合理导致：每个分区存在多个文件，文件数达到几百上千，其中大多数是大小只有几 k 的小文件。</br>
3、在使用insert into写入数据时过，几条数据就写入一次，并且频繁的写入。</br>
4、Reduce过程中产生小文件过多。</br>
5、Job执行过程中生成的各种临时文件、回收站保留的过期的文件过多。</br>

注意：虽然在MaxCompute系统侧会自动做小文件合并的优化，但对于原因1、2、3需要客户采用合理的表分区设计和上传数据的方法才可以避免。

<h4>小文件数量过多产生的影响</h4>
MaxCompute处理单个大文件比处理多个小文件更有效率，小文件过多会影响整体的执行性能;小文件过多会给文件系统带来一定的压力，且影响空间的有效利用。MaxCompute对单个fuxi Instance可以处理的小文件数限制为120个，文件数过多影响fuxi instance数目，影响整体性能。

<h4>合并小文件命令</h4>

```js
set odps.merge.max.filenumber.per.job=50000; --值默认为50000个；当分区数大于50000时需要调整，最大可到1000000万，大于1000000的提交多次merge
ALTER TABLE 表名[partition] MERGE SMALLFILES;
```
<h3>如何合并小文件</h3>
<h4>分区表：</h4>
如果您的表已经是分区表，请检查您的分区字段是否是可收敛的，如果分区数过多同样会影响计算性能，建议用日期做分区。
1、定期执行合并小文件命令；
2、如果是按日期建的分区，可以每天对前一天的分区数据用insert overwrite重新覆盖写入。
例如：

insert overwrite table tableA partition (ds='20181220')
select * from tableA where ds='20181220';
非分区表：
如果您的表是非分区表，您可以定期执行合并小文件命令来优化小文件问题，但强烈建议您设计成分区表：</br>
1、先创建一个新的分区表，建议按日期做分区，合理设置生命周期，以方便进行历史数据回收；</br>
2、把原非分区表的数据导入新的分区表；（建议先暂停原非分区表的实时写入业务）</br>
例如：
```js
create table sale_detail_patition like sale_detail;
alter table sale_detail_insert add partition(sale_date='201812120', region='china');
insert overwrite table sale_detail_patition partition (sale_date='20181220', region='china')
select * from sale_detail;
```

3、修改上下游业务：入库程序改成写入新分区表，查询作业改成从新分区表中查询；</br>
4、新分区表完成数据迁移和验证后，删除原分区表。</br>

注意：如果您使用insert overwrite重新写入全量数据合并小文件时，请注意一定不要同时存在insert overwrite和insert into同时存在的情况，否则有丢失数据的风险。

<h3>如何避免产生小文件</h3>
<h4>优化表设计</h4>
合理设计表分区，分区字段是尽量是可收敛或可管理的，如果分区数过多同样会影响计算性能，建议用日期做分区，并合理设置表的生命周期，以方便对历史数据回收，也可控制您的存储成本。
参考文章：《MaxCompute 表(Table)设计规范》、《MaxCompute表设计最佳实践》</br>

<h4>避免使用各种数据集成工具产生小文件</h4>
1、Tunnel->MaxCompute</br>
使用Tunnel上传数据时避免频繁commit，尽量保证每次提交的DataSize大于64M，请参考《离线批量数据通道Tunnel的最佳实践及常见问题》

2、Datahub->MaxCompute</br>
如果用Datahub产生小文件，建议合理申请shard，可以根据topic的Throughput合理做shard合并，减少shard数量。可以根据topic的Throughput观察数据流量变化，适当调大数据写入的间隔时间。

申请Datahub shard数目的策略（申请过多的datahub shard将会产生小文件问题）</br>
1）默认吞吐量单个shard是1MB/s，可以按照这个分配实际的shard数目（可以在此基础上多加几个）；</br>
2）同步MaxCompute的逻辑是每个shard有一个单独的task（满足5分钟或者64MB会commit一次），默认设置5分钟是为了尽快能在MaxCompute查到数据。如果是按照小时建partition，那个一个shard每个小时有12个文件。如果这个时候数据量很少，但是shard很多，在MaxCompute里面就会很多小文件（shard*12/hour）。所以不要过多的分配shard，按需分配。</br>

参考建议：​​如果流量是5M/s，那么就申请5个shard，为预防流量峰值预留20%的Buffer，可以申请6个shard。

3、DataX->MaxCompute
因为datax也是封装了tunnel的SDK来写入MaxCompute的，因此，建议您在配置ODPSWriter的时候，把blockSizeInMB这个参数不要设置太小，最好是64M以上。
