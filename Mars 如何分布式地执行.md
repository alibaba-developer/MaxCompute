# Mars 如何分布式地执行
先前，我们已经介绍过 Mars 是什么。如今 Mars 已在 Github 开源并对内上线试用，本文将介绍 Mars 已实现的分布式执行架构，欢迎大家提出意见。

<h3>架构</h3>
Mars 提供了一套分布式执行 Tensor 的库。该库使用 mars.actors 实现的 Actor 模型编写，包含 Scheduler、Worker 和 Web 服务。

用户向 Mars Web Service 提交的是由 Tensor 组成的 Graph。Web Service 接收这些图并提交到一台 Scheduler。在提交作业到各个 Worker 之前，Mars Scheduler 先将 Tensor 图编译成一张由 Chunk 和 Operand 组成的图，此后对图进行分析和切分。此后，Scheduler 在所有 Scheduler 中根据一致性哈希创建一系列控制单个 Operand 执行的 OperandActor。Operand 以符合拓扑序的顺序进行调度，当所有 Operand 完成执行，整张图将被标记为已完成，客户端能够从 Web 中拉取结果。整个执行过程如下图所述。

<div style="text-align:center" align="center">
<img src="/images/分布式地执行1.png" align="center" />
</div>

作业提交
用户端通过 RESTful API 向 Mars 服务提交作业。用户通过编写 Tensor 上的代码，此后通过 session.run(tensor) 将 Tensor 操作转换为 Tensor 构成的 Graph 并提交到 Web API。此后，Web API 将作业提交到 SessionActor 并在集群中创建一个 GraphActor 用于图的分析和管理。用户端则开始查询图的执行状态，直至执行结束。

在 GraphActor 中，我们首先根据 chunks 设置将 Tensor 图转换为 Operand 和 Chunk 组成的图，这一过程使得图可以被进一步拆分并能够并行执行。此后，我们在图上进行一系列的分析以获得 Operand 的优先级，同时向起始 Operand 指派 Worker，关于这一部分的细节可以参考 准备执行图 章节。此后，每个 Operand 均建立一个 OperandActor 用于控制该 Operand 的具体执行。当 Operand 处于 READY状态（如同在 Operand 状态 章节描述的那样），Scheduler 将会为 Operand 选择目标 Worker，随后作业被提交 Worker 进行实际的执行。

执行控制
当一个 Operand 被提交到 Worker，OperandActor 等待 Worker 上的回调。如果 Operand 执行成功，Operand 的后继将被调度。如果 Operand 执行失败，OperandActor 将会尝试数次，如果仍失败则将此次执行标记为失败。

取消作业
用户端可以使用 RESTful API 取消运行中的作业。取消请求将被写入 Graph 的状态存储中，同时 GraphActor 上的取消接口将被调用。如果作业在准备阶段，它将在检测到停止请求后立即结束，否则请求将被下发到每个 OperandActor，并设置状态为 CANCELLING。如果此时 Operand 没有运行，Operand 状态将被直接置为 CANCELLED。如果 Operand 正在运行，停止请求将被下发到 Worker 中并导致一个 ExecutionInterrupted 错误，该错误将返回给 OperandActor，此时 Operand 的状态将被标记为 CANCELLED。

<h3>准备执行图</h3>
当一个 Tensor 图被提交到 Mars Scheduler，一张包含更细粒度的，由 Operand 和 Chunk 构成的图将根据数据源中包含的 chunks 参数被生成。

图压缩
当完成 Chunk 图的生成后，我们将会通过合并图中相邻的节点来减小图的规模，这一合并也能让我们充分利用 numexpr 这样的加速库来加速计算过程。目前 Mars 仅会合并形成单条链的 Operand。例如，当执行下面的代码

```js
import mars.tensor as mt
a = mt.random.rand(100, chunks=100)
b = mt.random.rand(100, chunks=100)
c = (a + b).sum()

```
Mars 将会合并 Operand ADD 和 SUM 成为 FUSE 节点。RAND Operand 不会被合并，因为它们并没有和 ADD 及 SUM 组成一条简单的直线。

<div style="text-align:center" align="center">
<img src="/images/分布式地执行2.png" align="center" />
</div>

<h4>初始 Worker 分配</h4>
为 Operand 分配 Worker 对于图执行的性能而言至关重要。随机分配初始 Operand 可能导致巨大的网络开销，并有可能导致不同 Worker 间作业分配的不平衡。因为非初始节点的分配能够根据其前驱生成数据的物理分布及各个 Worker 的空闲情况方便地确定，在执行图准备阶段，我们只考虑初始 Operand 的分配问题。

初始 Worker 分配需要遵循几个准则。首先，分配给每个 Worker 执行的 Operand 需要尽量保持平衡满，这能够使计算集群在整个执行阶段都有较高的利用率，这在执行的最后阶段显得尤其重要。其次，初始节点分配需要使后续节点执行时的网络”传输尽量小。也就是说，初始点分配需要充分遵循局部性原则。

需要注意的是，上述准则在某些情况下会彼此冲突。一个网络传输量最小的分配方案可能会非常偏斜。我们开发了一套启发式算法来获取两个目标的平衡，该算法描述如下：

1. 表中的第一个初始节点和第一台机器；
1. 从 Operand 图转换出的无向图中自该点开始进行深度优先搜索；
1. 如果另一个未被分配的初始节点被访问到，我们将其分配给步骤1中选择的机器；
1. 当访问到的 Operand 总数大于平均每个 Worker 接受的 Operand 个数时，停止分配；

前往步骤1，如果仍有 Worker 未被分配 Operand，否则结束。

<h3>调度策略</h3>
当一个 Operand 组成的 Graph 执行时，合适的执行顺序会减少集群中暂存的数据总量，从而减小数据被 Spill 到磁盘的可能性。合适的 Worker 能够减少执行时网络传输的总量。

<h3>Operand 选择策略</h3>
合适的执行顺序能够显著减小集群中暂存的数据总量。下图中展示了 Tree Reduction 的例子，圆形代表 Operand，方形代表 Chunk，红色代表 Operand 正在执行，蓝色代表 Operand 可被执行，绿色代表 Operand 产生的 Chunk 已被存储，灰色代表 Operand 及其相关数据已被释放。假设我们有两台 Worker，并且每个 Operand 的资源使用量均相等，每张图展示的是不同策略下经过5个时间单元的执行后的状态。左图展示的是节点依照层次分别执行，而右图展示的是依照接近深度优先的顺序执行。左图中，有6个 Chunk 的数据需要暂存，右图只有2个。

<div style="text-align:center" align="center">
<img src="/images/分布式地执行3.png" align="center" />
</div>

因为我们的目标是减少存储在集群中的数据总数，我们为进入 READY 状态的 Operand 设定了一套优先级策略：

1. 大的 Operand 需要被优先执行；
1. 被更深的 Operand 依赖的 Operand 需要被优先执行；
1. 输出规模更小的节点需要被优先执行。

<h4>Worker 选择策略</h4>

当 Scheduler 准备执行图时，初始 Operand 的 Worker 已被确定。我们选择后续 Operand 分配 Worker 的依据是输入数据所在的 Worker。如果某个 Worker 拥有的输入数据大小最大，则该 Worker 将被选择用于执行后续 Operand。如果这样的 Worker 有多个，则各个候选 Worker 的资源状况将起到决定作用。

<h4>Operand 状态</h4>
Mars 中的每一个操作符都被一个 OperandActor 单独调度。执行的过程是一个状态转移的过程。在 OperandActor 中，我们为每一个状态的进入过程定义一个状态转移函数。起始 Operand 在初始化时位于 READY 状态，非起始 Operand 在初始化时则位于 UNSCHEDULED 状态。当给定的条件满足，Operand 将转移到另一个状态并执行相应的操作。状态转移的流程可以参考下图：

<div style="text-align:center" align="center">
<img src="/images/分布式地执行4.png" align="center" />
</div>

我们在下面描述每个状态的含义及 Mats 在这些状态下执行的操作。

- SCHEDUED：一个 Operand 位于此状态，当它的上游数据没有准备好。
- READY：一个 Operand 位于此状态，当所有上游输入数据均已准备完毕。在进入这一状态时，OperandActor 向 AssignerActor 中选择的所有 Worker 提交作业。如果某一 Worker 准备运行作业，它将向 Scheduler 发送消息，Scheduler 将向其他 Worker 发送停止运行的消息，此后向该 Worker 发送消息以启动作业执行。
- RUNNING：一个 Operand 位于此状态，当它的执行已经启动。在进入此状态时，OperandActor 会检查作业是否已经提交。如果尚未提交，OperandActor 将构造一个由 FetchChunk Operand 和当前 Operand 组成的图，并将其提交到 Worker 中。此后，OperandActor 会在 Worker 中注册一个回调来获取作业执行完成的消息。
- FINISHED：一个 Operand 位于此状态，当作业执行已完成。当 Operand 进入此状态，且 Operand 无后继，一个消息将被发送到 GraphActor 以决定是否整个 Graph 的执行都已结束。与此同时，OperandActor 向它的前驱和后继发送执行完成的消息。如果一个前驱收到此消息，它将检查是否所有的后继都已执行完成。如是，当前 Operand 上的数据可以被释放。如果一个后继收到此消息，它将检查是否所有的前驱已完成。如是，该后继的状态可以转移到 READY。
- FREED：一个 Operand 位于此状态，当其上所有数据都已被释放。
- CANCELLED：一个 Operand 位于此状态，当所有重新执行的尝试均告失败。当 Operand 进入此状态，它将把相同状态传递到后继节点。
- CANCELLING：一个 Operand 位于此状态，当它正在被取消执行。如果此前作业正在执行，一个取消执行的请求会被发送到 Worker 上。
- CANCELLED：一个 Operand 位于此状态，当执行已被取消并停止运行。如果执行进入这一状态，OperandActor 会尝试将书友的后继都转为 CANCELLING。
Worker 中的执行细节

一个 Mars Worker 包含多个进程，以减少全局解释器锁（GIL）对执行的影响。具体的执行在独立的进程中完成。为减少不必要的内存拷贝和进程间通讯，Mars Worker 使用共享内存来存储执行结果。

当一个作业被提交到 Worker，它将首先被置于队列中等待分配内存。当内存被分配后，其他 Worker 上的数据，或者当前 Worker 上已被 spill 到磁盘的数据将会被重新载入内存中。此时，所有计算需要的数据已经都在内存中，真正的计算过程将启动。当计算完成，Worker 将会把作业放到共享存储空间中。这四种执行状态的转换关系见下图。

<div style="text-align:center" align="center">
<img src="/images/分布式地执行5.png" align="center" />
</div>

<h4>执行控制</h4>
Mars Worker 通过 ExecutionActor 控制所有 Operand 在 Worker 中的执行。该 Actor 本身并不参与实际运算或者数据传输，只是向其他 Actor 提交任务。

Scheduler 中的 OperandActor 通过 ExecutionActor 上的 enqueue_graph 调用向 Worker 提交作业。Worker 接受 Operand 提交并且将其换存在队列中。当作业可以执行时，ExecutionActor 将会向 Scheduler 发送消息，Scheduler 将确定是否将执行该操作。当 Scheduler 确定在当前 Worker 上执行 Operand，它将调用 start_execution 方法，并通过 add_finish_callback注册一个回调。这一设计允许执行结果被多个位置接收，这对故障恢复有价值。

ExecutionActor 使用 mars.promise 模块来同时处理多个 Operand 的执行请求。具体的执行步骤通过 Promise 类的 then 方法相串联。当最终的执行结果被存储，之前注册的回调将被触发。如果在之前的任意执行步骤中发生错误，该错误会被传导到最后 catch 方法注册的处理函数中并得到处理。

<h4>Operand 的排序</h4>
所有在 READY 状态的 Operand 都被提交到 Scheduler 选择的 Worker 中。因此，在执行的绝大多数时间里，提交到 Operand 的 Worker 个数通常都高于单个 Worker 能够处理的 Operand 总数。因此，Worker 需要对 Operand 进行排序，此后选择一部分 Worker 来执行。这一排序过程在 TaskQueueActor 中进行，该 Actor 中维护一个优先队列，其中存储 Operand 的相关信息。与此同时，TaskQueueActor 定时运行一个作业分配任务，对处于优先队列头部的 Operand 分配执行资源直至没有多余的资源来运行 Operand，这一分配过程也会在新 Operand 提交或者 Operand 执行完成时触发。

<h4>内存管理</h4>
Mars Worker 管理两部分内存。第一部分是每个 Worker 进程私有的内存空间，由每个进程自己持有。第二部分是所有进程共享的内存空间，由 Apache Arrow 中的 plasma_store 持有。

为了避免进程内存溢出，我们引入了 Worker 级别的 QuotaActor，用于分配进程内存。当一个 Operand 开始执行前，它将为输入和输出 Chunk 向 QuotaActor 发送批量内存请求。如果剩余的内存空间可以满足请求，该请求会被 QuotaActor 接受。否则，请求将排队等待空闲资源。当相关内存使用被释放，请求的资源会被释放，此时，QuotaActor 能够为其他 Operand 分配资源。

共享内存由 plasma_store 管理，通常会占据整个内存的 50%。由于不存在溢出的可能，这部分内存无需经过 QuotaActor 而是直接通过 plasma_store 的相关方法进行分配。当共享内存使用殆尽，Mars Worker 会尝试将一部分不在使用的 Chunk spill 到磁盘中，以腾出空间容纳新的 Chunk。

从共享内存 spill 到磁盘的 Chunk 数据可能会被未来的 Operand 重新使用，而从磁盘重新载入共享内存的操作可能会非常耗费 IO 资源，尤其在共享内存已经耗尽，需要 spill 其他 Chunk 到磁盘以容纳载入的 Chunk 时。因此，当数据共享并不需要时，例如该 Chunk 只会被一个 Operand 使用，我们会将 Chunk 直接载入进程私有内存中，而不是共享内存，这可以显著减少作业总执行时间。

<h4>未来工作</h4>
Mars 目前正在快速迭代，近期将考虑实现 Worker 级别的 failover 及 shuffle 支持，Scheduler 级别的 failover 也在计划中。
