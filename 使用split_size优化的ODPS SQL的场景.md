# 使用split_size优化的ODPS SQL的场景
首先有两个大背景需要说明如下：
说明1：split_size，设定一个map的最大数据输入量，单位M，默认256M。用户可以通过控制这个变量，从而达到对map端输入的控制。设置语句：set odps.sql.mapper.split.size=256。一般在调整这个设置时，往往是发现一个map instance处理的数据行数太多。

说明2：小文件越多，需要instance资源也越多，MaxCompute对单个Instance可以处理的小文件数限制为120个，如此造成浪费资源，影响整体的执行性能（文件的大小小于块Block 64M的文件）。


<h4>场景一：单记录数据存储太少</h4>
<div style="text-align:center" align="center">
<img src="/images/使用split_size优化的ODPS SQL的场景1.png" align="center" />
</div>

原始Logview Detail：

<div style="text-align:center" align="center">
<img src="/images/使用split_size优化的ODPS SQL的场景2.png" align="center" />
</div>

可以发现Job只调起一个Map Instance，供处理了156M的数据，但这些数据共有5千多万的记录（单记录平均3个byte），花费了25分钟。
此外，从TimeLine看可以发现，整个Job耗费43分钟，map占用了超过60%的时间。故可对map进行优化。

优化手段：调小split_size为16M

<div style="text-align:center" align="center">
<img src="/images/使用split_size优化的ODPS SQL的场景3.png" align="center" />
</div>

优化之后的logview：

<div style="text-align:center" align="center">
<img src="/images/使用split_size优化的ODPS SQL的场景4.png" align="center" />
</div>

优化后，可以发现，Job调起了7个Map Instance，耗时4分钟；某一个Map处理了27M的数据，6百万记录。（这里可以看出set split_size只是向Job提出申请，单不会严格生效，Job还是会根据现有的资源情况等来调度Instance）因为Map的变多，Join和Reduce的instance也有增加。整个Job的执行时间也下降到7分钟。


<h4>场景二：用MapJoin实现笛卡尔积</h4>
<div style="text-align:center" align="center">
<img src="/images/使用split_size优化的ODPS SQL的场景5.png" align="center" />
</div>

原始logview：

<div style="text-align:center" align="center">
<img src="/images/使用split_size优化的ODPS SQL的场景6.png" align="center" />
</div>

可以发现，Job调起了4个Map，花费了3个小时没有跑完；查看详细Log，某一个Map因为笛卡尔的缘故，生成的数据量暴涨。
综合考虑，因为该语句使用Mapjoin生成笛卡尔积，再筛选符合条件的记录，两件事情都由map一次性完成，故对map进行优化。

策略调低split_size
优化后的logview：

<div style="text-align:center" align="center">
<img src="/images/使用split_size优化的ODPS SQL的场景7.png" align="center" />
</div>

优化后，可以看到，Job调度了38个map，单一map的生成数据量下降了，整体map阶段耗时也下降到37分钟。
回头追朔这个问题的根源，主要是因为使用mapjoin笛卡尔积的方式来实现udf条件关联的join，导致数据量暴涨。故使用这种方式来优化，看起来并不能从根本解决问题，故我们需要考虑更好的方式来实现类似逻辑。
