# Mars 是什么、能做什么、如何做的——记 Mars 在 PyCon China 2018 上的分享
最近，在 PyCon China 2018 的北京主会场、成都和杭州分会场都分享了我们最新的工作 Mars，基于矩阵的统一计算框架。本文会以文字的形式对 PyCon 中国上的分享再进行一次阐述。

听到 Mars，很多第一次听说的同学都会灵魂三问：Mars 是什么，能做什么，怎么做的。今天我们就会从背景，以及一个例子出发，来回答这几个问题。

<h3>背景</h3>
<div style="text-align:center" align="center">
<img src="/images/Mars 是什么1.png" align="center" />
</div>

首先是 scipy 技术栈的全景图，numpy 是基础，它提供了多维数组的数据结构，并提供了它上面的各种计算。再往上，重要的有 scipy，主要面向各种科学计算的操作；pandas，其中核心的概念是 DataFrame，他提供对表类型数据的处理、清洗等功能。往上一层，比较经典的库，有 scikit-learn，它是最知名的机器学习框架之一。最上面一层，是各种垂直领域的库，如 astropy 主要面向天文，biopython 面向生物领域等。

从 scipy 技术栈可以看出，numpy 是一个核心的地位，大量上层的库都使用了 numpy 的数据结构和计算。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么2.png" align="center" />
</div>

我们真实世界的数据，并不只是表这种二维类型数据那么简单，很多时候，我们要面对的往往是多维数据，比如我们常见的图片处理，首先我们有图片的个数，然后有图片的长宽，以及 RGBA 通道，这就是四维的数据；这样的例子不胜枚举。有这样多维的处理能力，就有处理各种更加复杂，甚至是科学领域的能力；同时，由于多维数据本身包含二维数据，所以，我们也因此具备表类型数据的处理能力。

另外，如果我们需要探究数据的内在，光靠对表数据进行一些统计等操作是绝对不够的，我们需要更深层的“数学” 的方法，比如运用矩阵乘法、傅里叶变换等等的能力，来对数据进行更深层次的分析。而 numpy 由于是数值计算的库，加上各种上层的库，我们认为它们很适合用来提供这方面的能力。

<h3>为什么要做 Mars，从一个例子开始</h3>
<div style="text-align:center" align="center">
<img src="/images/Mars 是什么3.png" align="center" />
</div>

那么，为什么要做 Mars 这个项目呢？我们不妨从一个例子来看。

我们试图用蒙特卡洛方法来求解 pi，蒙特卡洛方法其实很简单，就是用随机数的方法来解决特定的问题。如图，这里我们有个半径为1的圆和边长为2的正方形，我们生成很多随机点的方式，通过右下角的公式，我们就可以计算出 pi 的值为 4 乘以落在圆里点的个数除以总的点个数。随机生成的点越多，计算出来的 pi 就越精确。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么4.png" align="center" />
</div>

用纯 Python 实现非常简单，我们只要遍历 N 次，生成 x 和 y 点，计算是不是落在圆内即可。运行1千万个点，需要超过10秒的时间。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么5.png" align="center" />
</div>

Cython 是常见加速 Python 代码的方式，Cython 定义了 Python 语言的超集，把这个语言翻译到 c/c++，然后再进行编译来加速执行。这里，我们增加了几个变量的类型，可以看到比纯 Python 提升了 40% 的性能。

Cython 现在已经成为 Python 项目的标配，核心的 Python 三方库基本都使用 Cython 来加速 Python 代码的性能。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么6.png" align="center" />
</div>

我们这个例子里的数据都是一个类型，我们可以想到用专门的数值计算的库，通过矢量化的方式，能极快加速这个任务的性能。numpy 就是当仁不让的选择了，使用 numpy，我们需要的是面向 array 的思维方式，我们应当减少使用循环。这里先用 numpy.random.uniform 来生成 N*2 的一个二维数组，然后 data ** 2 会对该数组里的所有数据做平方操作，然后 sum(axis=1) ，会对 axis=1 也就是行方向上求和，这个时候，得到的是长度为 N 的 vector，然后我们用 numpy.sqrt 来对这个 vector 的每一个值求开方，<1 会得到一个布尔值的 vector，即每个点是不是都是落在圆里，最后接一个 sum，就可以求出来总共的点的个数。初次上手 numpy 可能会不太习惯，但是用多了以后，就会发现这种写法的方便，它其实是非常符合直觉的。

可以看到，通过使用 numpy，我们写出了更简单的代码，但是性能确大幅提升，比纯 Python 的写法性能提升超过 10 倍。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么7.png" align="center" />
</div>

那么 numpy 的代码还能够优化么，答案是肯定的，我们通过一个叫 numexpr 的库，来将 numpy 的多个操作合并成一个操作执行，来加速 numpy 的执行。

可以看到，通过 numexpr 优化的代码，性能比纯 Python 代码提升超过 25 倍。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么8.png" align="center" />
</div>

此时的代码运行已经相当快了，如果我们手上有 GPU，那么我们可以利用硬件来加速任务执行。

这里必须要安利一个库，叫 cupy，他提供了和 numpy 一致的 API，通过简单的 import 替换，就能让 numpy 代码跑在英伟达的显卡之上。

这时可以看到，性能大幅提升超过 270 倍。真的非常夸张了。

为了让蒙特卡洛方法计算的结果更加精确，我们把计算量扩大 1000 倍。会碰到什么情况呢？

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么9.png" align="center" />
</div>

没错，这就是大家不时碰到的，OutOfMemory，内存溢出。更惨的是，在 jupyter 里，有时候内存溢出导致进程被杀，甚至会导致之前跑的全部结果都丢失。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么10.png" align="center" />
</div>

蒙特卡洛方法还是比较容易处理的，我把问题分解成 1000 个，每个求解1千万数据就好了嘛，写个循环，做个汇总。但此时，整个计算的时间来到了12分钟多，太慢了。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么11.png" align="center" />
</div>

此时我们会发现，整个运行过程中，其实只有一个 CPU 在干活，其他核都在原地吆喝。那么，我们怎么让 numpy 并行化呢？

首先，numpy 里有一些操作是能并行的，比如 tensordot 来做矩阵乘法，其他大部分操作都不能利用多核。那么，要将 numpy 并行化，我们可以：

采用多线程/多进程编写任务
分布式化
<div style="text-align:center" align="center">
<img src="/images/Mars 是什么12.png" align="center" />
</div>

蒙特卡洛方法算 pi 改写成多线程和多进程实现还是非常容易的，我们写一个函数来处理1千万的数据，我们把这个函数通过 concurrent.futures 的 ThreadPoolExecutor 和 ProcessPoolExecutor 来分别提交函数 1000 遍用多线程和多进程执行即可。可以看到性能能提升到 2倍和3倍。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么13.png" align="center" />
</div>

但是呢，蒙特卡洛求解 pi 本身就很容易手写并行，考虑更复杂的情况。


```js
import numpy as np

a = np.random.rand(100000, 100000)
(a.dot(a.T) - a).std()

```

这里创建了 10万*10万的矩阵 a，输入就有大概 75G，我们拿 a 矩阵乘 a 的转置，再减去 a 本身，最终求标准差。这个任务的输入数据就很难塞进内存，后续的手写并行就更加困难。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么14.png" align="center" />
</div>

这里问题就引出来了，我们需要什么样框架呢？

1. 提供熟悉的接口，像 cupy 这样，通过简单的 import 替换，就能让原来 numpy 写的代码并行。
1. 具备可扩展性。小到单机，也可以利用多核并行；大到一个很大的集群，支持上千台机器的规模来一起分布式处理任务。
1. 支持硬件加速，支持用 GPU 等硬件来加速任务执行。
1. 支持各种优化，比如操作合并，能利用到一些库来加速执行合并的操作。
1. 我们虽然是内存计算的，但不希望单机或者集群内存不足，任务就会失败。我们应当让暂时用不到的数据 spill 到磁盘等等存储，来保证即使内存不够，也能完成整个计算。

<h3>Mars 是什么，能做什么事</h3>

Mars 就是这样一个框架，它的目标就是解决这几个问题。目前 Mars 包括了 tensor ：分布式的多维矩阵计算。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么15.png" align="center" />
</div>

100亿大小的蒙特卡洛求解 pi的问题规模是 150G，它会导致 OOM。通过 Mars tensor API，只需要将 import numpy as np 替换成 import mars.tensor as mt，后续的计算完全一致。不过有一个不同，mars tensor 需要通过 execute 触发执行，这样做的好处是能够对整个中间过程做尽量多的优化，比如操作合并等等。不过这种方式对 debug 不太友好，后续我们会提供 eager mode，来对每一步操作都触发计算，这样就和 numpy 代码完全一致了。

可以看到这个计算时间和手写并行时间相当，峰值内存使用也只是 1个多G，因此可以看到 Mars tensor 既能充分并行，又能节省内存的使用 。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么16.png" align="center" />
</div>

目前，Mars 实现了 70% 的常见 numpy 接口，完整列表见 这里。我们一致在努力提供更多 numpy 和 scipy 的接口，我们最近刚刚完成了对逆矩阵计算的支持。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么17.png" align="center" />
</div>

Mars tensor 也提供了对 GPU 和稀疏矩阵的支持。eye 是创建单位对角矩阵，它只有对角线上有值为1，如果用稠密的方式存储会浪费存储。不过目前，Mars tensor 还只支持二维稀疏矩阵。

Mars 怎么做到并行和更省内存
<div style="text-align:center" align="center">
<img src="/images/Mars 是什么18.png" align="center" />
</div>

和所有的 dataflow 的框架一样，Mars 本身也有计算图的概念，不同的是，Mars 包含粗粒度图和细粒度图的概念，用户写的代码在客户端生成粗粒度图，在提交到服务端后，会有 tile 的过程，将粗粒度图 tile 成细粒度图，然后我们会调度细粒度图执行。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么19.png" align="center" />
</div>

这里，用户写下的代码，在内存里会表达成 Tensor 和 Operand 构成的粗粒度图。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么20.png" align="center" />
</div>

当用户调用 execute 方法时，粗粒度的图会被序列化到服务端，反序列化后，我们会把这个图 tile 成细粒度图。对于输入 10002000 的矩阵，假设指定每个维度上的 chunk 大小都是 500，那它会被 tile 成 24 一共 8 个chunk。

后续，我们会对每个我们实现的 operand 也就是算子提供 tile 的操作，将一个粗粒度的图 tile 成细粒度图。这时，我们可以看到，在单机，如果有8个核，那么我们就可以并行执行整个细粒度图；另外给定 1/8 大小的内存，我们就可以完成整个图的计算。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么21.png" align="center" />
</div>

不过，我们在真正执行前，会对整个图进行 fuse 也就是操作合并的优化，这里的三个操作真正执行的时候，会被合并成一个算子。针对执行目标的不同，我们会使用 numexpr 和 cupy 的 fuse 支持来分别对 CPU 和 GPU 进行操作合并执行。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么22.png" align="center" />
</div>

上面的例子，都是我们造出来很容易并行的任务。如我们先前提到的例子，通过 tile 之后生成的细粒度图其实是非常复杂的。真实世界的计算场景，这样的任务其实是很多的。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么23.png" align="center" />
</div>

为了将这些复杂的细粒度图能够充分调度执行，我们必须要满足一些基本的准则，才能让执行足够高效。

首先，初始节点的分配非常重要。比如上图，假设我们有两个 worker，如果我们把 1和3 分配到一个 worker，而将 2和4 分配到另一个 worker，这时当 5 或者 6 被调度的时候，他们就需要触发远程数据拉取，这样执行效率会大打折扣。如果我们一开始将 1和2 分配到一个 worker，将 3和4 分配到另一个 worker，这时执行就会非常高效。初始节点的分配对整体的执行影响是很大的，这就需要我们对整个细粒度的图有个全局的掌握，我们才能做到比较好的初始节点分配。

另外，深度优先执行的策略也是相当重要的。假设这时，我们只有一个 worker，执行完 1和2 后，我们调度了 3 的话，就会导致 1和2 的内存不能释放，因为 5 此时还没有被触发执行。但是，如果我们执行完 1和2 后，调度了 5 执行，那么当 5 执行完后，1和2 的内存就可以释放，这样整个执行过程中的内存就会是最省的。

所以，初始节点分配，以及深度优先执行是两个最基本的准则，光有这两点是远远不够的，mars 的整个执行调度中有很多具有挑战的任务，这也是我们需要长期优化的对象。

Mars 分布式
<div style="text-align:center" align="center">
<img src="/images/Mars 是什么24.png" align="center" />
</div>

所以，Mars 本质上其实是一个细粒度的，异构图的调度系统。我们把细粒度的算子调度到各个机器上，在真正执行的时候其实是调用 numpy、cupy、numexpr 等等的库。我们充分利用了成熟的、高度优化的单机库，而不是重复在这些领域造轮子。

在这个过程中，我们会遇到一些难点：

因为我们是 master-slave 架构，我们 master 如何避免单点？
我们的 worker 如何避免 Python 的 GIL（全局解释器锁）的限制？
Master 的控制逻辑交错复杂，我们很容易写出来高耦合的，又臭又长的代码，我们如何将代码解耦？
<div style="text-align:center" align="center">
<img src="/images/Mars 是什么25.png" align="center" />
</div>

我们的解法是使用 Actor model。Actor模型定义了并行的方式，也就是一切皆 Actor，每个 Actor 维护一个内部状态，它们都持有邮箱，Actor 之间通过消息传递，消息收到会放在邮箱中，Actor 从邮箱中取消息进行处理，一个 Actor 同时只能处理一个消息。Actor 就是一个最小的并行单元，由于一个 Actor 同时只能处理一个消息，你完全不需要担心并发的问题，并发应当是 Actor 框架来处理的。而所有 Actor 是不是在同一台机器上，这在 Actor 模型里也变得不重要，Actor 在不同机器上，只要能完成消息的传递就可以了，这样 Actor 模型也天然支持分布式系统。

因为 Actor 是最小的并行单元，我们在写代码的时候，可以将整个系统分解成很多 Actor，每个 Actor 是单一职责的，这有点类似面向对象的思想，这样让我们的代码得以解耦。

另外，Master 解耦成 Actor 之后，我们可以让这些 Actor 分布在不同的机器上，这样就让 Master 不再成为单点。同时，我们让这些 Actor 根据一致性哈希来进行分配，后续如果有 scheduler 机器挂掉， Actor 可以根据一致性哈希重新分配并重新创建来达到容错的目的。

最后，我们的 actors 是跑在多进程上的，每个进程里是很多的协程，这样，我们的 worker 也不会受到 GIL 的限制。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么26.png" align="center" />
</div>

像 Scala 或者 Java 这些 JVM 语言 可以使用 akka 这个 Actor 框架，对于 Python 来说，我们并没有什么标准做法，我们认为我们只是需要一个轻量的 Actor 框架就可以满足我们使用，我们不需要 akka 里面一些高阶的功能。因此，我们开发了 Mars actors，一个轻量的 Actor 框架，我们 Mars 整个分布式的 schedulers 和 workers 都在 Mars actors 层之上。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么27.png" align="center" />
</div>

这是我们 Mars actors 的架构图，在启动 Actor pool 的时候，我们子进程会根据并发启动若干子进程。主进程上有 socket handler 来接受远程 socket 连接传递消息，另外主进程有个 Dispatcher 对象，用来根据消息的目的地来进行分发。我们所有的 Actor 都在子进程上创建，当 Actor 收到一个消息处理时，我们会通过协程调用 Actor.on_receive(message) 方法。

一个 Actor 发送消息到另一个 Actor，分三种情况。

它们在同一个进程，那么直接通过协程调用即可。
它们在一台机器不同进程，这个消息会被序列化通过管道送到主进程的 Dispatcher，dispatcher 通过解开二进制的头部信息得到目标的进程 ID，通过对应的管道送到对应子进程，子进程通过协程触发相应 Actor 的消息处理即可。
它们在不同机器，那么当前子进程会通过 socket 把序列化的消息发送到对应机器的主进程，该机器再通过 Dispatcher 把消息送到对应子进程。
由于使用协程作为子进程内的并行方式，而协程本身在 IO 处理上有很强的性能，所以，我们的 Actor 框架在 IO 方面也会有很好的性能。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么28.png" align="center" />
</div>

上图是裸用 Mars actors 来求解蒙特卡洛方法算 pi。这里定义两个 Actor，一个 Actor 是 ChunkInside，它接受一个 chunk 的大小，来计算落在圆内点的个数；另外一个 Actor 是 PiCaculator，它负责接受总的点个数，来创建 ChunkInside，这个例子就是直接创建 1000 个 ChunkInside，然后通过发送消息来触发他们计算。create_actor 时指定 address 可以让 Actor 分配在不同的机器上。

这里可以看到，我们裸用 Mars actors 的性能是要快过多进程版本的。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么29.png" align="center" />
</div>

这里我们总结一下，通过使用 Mars actors，我们能不受 GIL 限制，编写分布式代码变得非常容易，它让我们 IO 变得高效，此外，因为 Actor 解耦，代码也变得更容易维护。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么30.png" align="center" />
</div>

现在让我们看下 Mars 分布式的完整执行过程。现在有1个 client，3个 scheduler 和 5个worker。用户创建一个 session，在服务端会创建一个 SessionActor 对象，通过一致性哈希，分配到 scheduler1 上。此时，用户运行了一个 tensor，首先 SessionActor 会创建一个 GraphActor，它会 tile 粗粒度图，图上假设有三个节点，则会创建三个 OperandActor，分别分配到不同的 scheduler 上。每个 OperandActor 会控制 operand 的提交、任务状态的监督和内存的释放等操作。此时 1 和 2 的 OperandActor 发现没有依赖，并且集群资源充足，那么他们会把任务提交到相应的 worker 执行，在执行完成后，向 3 通知任务完成，3 发现 1和2 都执行完成后，因为数据在不同 worker 执行，决定好执行 worker 后，先触发数据拉取操作，然后再执行。客户端这边，通过轮询 GraphActor 得知任务完成，则会触发数据拉取到本地的操作。整个任务就完成了。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么31.png" align="center" />
</div>

我们对 Mars 分布式做了两个 benchmark，第一个是对 36 亿数据的每个元素加一再乘以2，图中红色叉是 numpy 的执行时间，可以看到，我们比 numpy 有数倍提升，蓝色的虚线是理论运行时间，可以看到我们真实加速非常接近理论时间加速。第二个 benchmark，我们增加了数据量，来到 144 亿数据，对每个元素加1乘以2后，再求和，可以看到单机 numpy 已经不能完成任务了，此时，针对这个任务，我们也可以取得不错的加速比。

未来计划
<div style="text-align:center" align="center">
<img src="/images/Mars 是什么32.png" align="center" />
</div>

Mars 已经在 Github 上源代码，让更多同学来一起参与共建 Mars：https://github.com/mars-project/mars 。

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么33.png" align="center" />
</div>

在后续 Mars 的开发计划上，如上文说，我们会支持 eager mode，让每一步触发执行，提升对性能不敏感的任务开发以及 debug 时的使用体验；我们会支持更多 numpy 和 scipy 接口；后续很重要的一个是，我们会提供 100% 兼容 pandas 的接口，由于利用了 mars tensor 作为基础，我们也可以提供 GPU 的支持；我们会提供兼容 scikit-learn 的机器学习的支持；我们还会提供在细粒度图上调度自定义函数和自定义类的功能，增强灵活性；最后，因为我们客户端其实并不依赖 Python，任意语言都可以序列化粗粒度图，所以我们完全可以提供多语言的客户端版本，不过这点，我们会视需求而定。

总之，开源对我们是很重要的，庞大的 scipy 技术栈的并行化，光靠我们的力量是不够的，需要大家来一起帮我们来共建。

现场图片
最后再 po 一点现场图片吧，现场观众对 Mars 的问题还是蛮多的。我大致总结下：

Mars 在一些特定计算的性能，比如 SVD 分解，这里我们有和用户合作项目的一些测试数据，输入数据是 8亿*32的矩阵做 SVD 分解，分解完再矩阵乘起来和原矩阵做对比，这整个计算过程使用 100个 worker（8核），用7分钟时间算完
Mars 何时开源，我们已经开源：https://github.com/mars-project/mars
Mars 开源后会不会闭源，答：不会
Mars actors 的详细工作原理
Mars 是静态图还是动态图，目前是静态图，eager mode 做完后可以支持动态图
Mars 会不会涉及深度学习，答：目前不会

<div style="text-align:center" align="center">
<img src="/images/Mars 是什么34.png" align="center" />
<img src="/images/Mars 是什么35.png" align="center" />
<img src="/images/Mars 是什么36.png" align="center" />
</div>
