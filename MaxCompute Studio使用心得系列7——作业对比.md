# MaxCompute Studio使用心得系列7——作业对比.md

在数据开发过程中，我们通常需要将两个作业进行对比从而定位作业运行性能或者结果有差异的问题，但是对比作业时需要同时打开两个studio 的tab页，或者两个Logview页，不停切换进行对比，使用起来非常的不方便。MaxCompute Studio从3.1.0版本开始支持作业对比，可以在一个页面同时比较两个作业，并且能自动标注出作业的差异点。

本文我以查找同个作业执行两次用时差别很大的原因为例，通过MaxCompute Studio的对比功能对两次执行的job进行对比，找出执行时间差别大的原因。

作业对比入口
MaxCompute Studio的Maxcompute 工具菜单中进入作业对比。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比1.png" align="center" />
</div>
</br>

输入两个需要对比的job的logview url 地址，点击“OK”按钮就可以开始对比：

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比2.png" align="center" />
</div>
</br>

对比基本信息
作业一运行了01:11:08 ，作业二运行了00:45:59，想知道是什么导致了相差近半个小时。先看基本信息对比：

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比3.png" align="center" />
</div>
</br>

通过基本信息可以看到studio 标注出作业一的耗时明显上升，其他项目，如IO Bytes 等相差不多， 输入输出表完全相同。基本可以断定是同一作业，为了确保是同一个作业还可以对比执行计划和脚本。

对比执行图
打开执行图 Tab ，可以一目了然看到两个作业的执行计划，执行图无法进行标注，可以通过查看text diff查看。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比4.png" align="center" />
</div>
</br>

点击text diff 后可以对比fuxi task 的执行时间， 输入输出等详细信息。可以比较绝对值，可以按比例比较，不一致的地方都会进行有效标注。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比5.png" align="center" />
</div>
</br>

这里可以发现执行计划是完全一致的。

对比脚本
点击脚本对比Tab 后，可以对比settings 和script ，settings 非常关键，不同的参数可能会导致完全不同的结果。这里需要使用text diff 功能比较sql 脚本。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比6.png" align="center" />
</div>
</br>

可以看到脚本对比功能很方便使用，即便是很复杂的sql 脚本都可以快速发现区别，这里发现只有分区日期不同，其他完全一致。

进一步分析执行计划
通过前面几个对比，确定两个作业完全一致， 再回到执行图中， 通过回放可以发现运行瓶颈在J4， 查看text diff 发现作业一的J4 用时52分， 作业二28分，由此判断作业一主要是J4用时长导致整体运行变慢。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比7.png" align="center" />
</div>
</br>

接下来重点分析J4 ，打开J4的 Operation Graph， Studio 在Operation 层新添加了Metric 信息， 可以看到每个operation 的执行时间，inner_time_ms, 这个时间指Operation 执行完所有行的平均时间（每个fuxi instance 都会用这个operation执行， 当这个operation 处理完所有分配给他的数据后就得出一个时间，这里的inner_time_ms 指的是这些fuxi instance 对应的Operation 执行的平均时间） ，通过这个时间可以发现某个Operation 执行时间是否过长，例如自定义udf 是否有性能问题。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比8.png" align="center" />
</div>
</br>

对比J4 的实际运行时间相差不多，并且执行的都比较快，由此可以考虑J4 是否存在等待资源情况， 导致fuxi instance 并没有及时开始运行。

对比分析Tab
打开作业分析tab的时序图子页面，可以明显发现作业一的J4_2_3 task 运行时间大于作业二的， 与前面看到的执行计划图一致。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比9.png" align="center" />
</div>
</br>

鼠标放到J4上点击展开作业后，可以看出fuxi instance 开始执行时间非常晚，这进一步验证了资源不足导致作业等待情况。

<div style="text-align:center" align="center">
<img src="/images/MaxCompute Studio使用心得系列7——作业对比10.png" align="center" />
</div>
</br>

<b>小结</b>

通过Studio 的作业对比功能，调查了资源等待导致的作业运行变慢情况， 并且排查的效率更高。作业对比还有很多其他功能，各位可以自行尝试。
