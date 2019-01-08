# Mars 算法实践——人脸识别
Mars 是一个基于矩阵的统一分布式计算框架，在之前的文章中已经介绍了 Mars 是什么， 以及 Mars 分布式执行 ，而且 Mars 已经在 GitHub 中开源。当你看完 Mars 的介绍可能会问它能做什么，这几乎取决于你想做什么，因为 Mars 作为底层运算库，实现了 numpy 70% 的常用接口。这篇文章将会介绍如何使用 Mars 完成你想做的事情。

<h4>奇异值分解 (SVD)</h4>

在处理纷繁的数据时，作为数据处理者，首先想到的就是降维，SVD 就是其中一种比较常见的降维方法，在 numpy.linalg 模块中就有 svd 方法，当我们有20000个100维的数据需要处理，调用 SVD 接口：

```js
In [1]: import numpy as np

In [2]: a = np.random.rand(20000, 100)

In [3]: %time U, s, V = np.linalg.svd(a)
CPU times: user 4min 3s, sys: 10.2 s, total: 4min 13s
Wall time: 1min 18s
```

可以看到即使 Numpy 使用了 mkl 加速，也需要1分多钟的运行时间，当数据量更大时，单机的内存已经无法处理。

Mars 也实现了 SVD ，但是它比 Numpy 有更快的速度，因为利用矩阵分块计算的算法，能够并行计算：

```js
In [1]: import mars.tensor as mt

In [2]: a = mt.random.rand(20000, 100, chunk_size=100)

In [3]: %time U, s, V = mt.linalg.svd(a).execute()
CPU times: user 5.42 s, sys: 1.49 s, total: 6.91 s
Wall time: 1.87 s
```

可以看到在相同数据量情况下，Mars 有几十倍速度的提升，仅仅需要1秒多钟就可以解决20000数据量的降维问题。想象一下淘宝用户数据做矩阵分解时，分布式的矩阵运算就显现出其价值。

主成分分析 (PCA)
提到降维，主成分分析也是一种重要的手段。PCA 会选取包含信息量最多的方向对数据进行投影，其投影方向可以从最大化方差或者最小化投影误差两个角度理解。也就是通过低维表征的向量和特征向量矩阵，可以基本重构出所对应的原始高维向量。其最主要的公式如下所示：
<div style="text-align:center" align="center">
<img src="/images/人脸识别1.png" align="center" />
</div>

为每个样本的数据，为新的投影方向，我们的目标就是使得投影方差最大化，从而找到主特征。上面式子中的矩阵在数学中可以用协方差矩阵表示，当然首先要对输入的样本做中心化调整。我们可以用随机产生的数组看一下 Numpy 是如何实现 PCA 降维操作：

```js
import numpy as np

a = np.random.randint(0, 256, size=(10000, 100))
a_mean = a.mean(axis=1, keepdims=True)
a_new = a - a_mean
cov_a = (a_new.dot(a_new.T)) / (a.shape[1] - 1)


#利用SVD求协方差矩阵前20个特征值

U, s, V = np.linalg.svd(cov_a)
V = V.T
vecs = V[:, :20]
```

#用低纬度的特征向量表示原数据
a_transformed = a.dot(vecs)
由于随机产生的数据本身就没有太强的特征，所以在100维数据中象征性的取出前20维，一般可以用特征值的比例取总和的前99%之类的数值。

再看一下 Mars 是如何实现的：

```js
import mars.tensor as mt

a = mt.random.randint(0, 256, size=(10000, 100))
a_mean = a.mean(axis=1, keepdims=True)
a_new = a - a_mean
cov_a = (a_new.dot(a_new.T)) / (a.shape[1] - 1)

#利用SVD求协方差矩阵前20个特征值
U, s, V = mt.linalg.svd(cov_a)
V = V.T
vecs = V[:, :20]

#用低纬度的特征向量表示原数据
a_transformed = a.dot(vecs).execute()

```
可以看到除了 import 的不同，再者就是对最后需要数据的变量调用 execute 方法，甚至在未来我们做完 eager 模式后， execute 都可以省去，以前用 Numpy 写的算法可以几乎无缝转化成多进程以及分布式的程序，再也不用自己手动去写MapReduce。

<h4>人脸识别</h4>
当 Mars 实现了基础算法时，便可以使用到实际的算法场景中。PCA最著名的应用就是人脸特征提取以及人脸识别，单个人脸图片的维度很大，分类器很难处理，早起比较知名的人脸识别 Eigenface 算法就是采用PCA算法。本文以一个简单的人脸识别程序作为例子，看看 Mars 是如何实现该算法的。

本文的人脸数据库用的是ORL face database，有40个不同的人共400张人脸图片，每张图片为 92*112 像素的灰度图片。这里选取每组图片的第一张人脸图片作为测试图片，其余九张图片作为训练集。

首先利用 python 的 OpenCV 的库将所有图片读取成一个大矩阵，也就是 360*10304大小的矩阵，每一行是每个人脸的灰度值，一共有360张训练样本。利用 PCA 训练数据，data_mat 就是输入的矩阵，k 是需要保留的维度。

```js
import mars.tensor as mt
from mars.session import new_session

session = new_session()

def cov(x):
    x_new = x - x.mean(axis=1, keepdims=True)
    return x_new.dot(x_new.T) / (x_new.shape[1] - 1)

def pca_compress(data_mat, k):
    data_mean = mt.mean(data_mat, axis=0, keepdims=True)
    data_new = data_mat - data_mean
    
    cov_data = cov(data_new)

    U, s, V = mt.linalg.svd(cov_data)
    V = V.T
    vecs = V[:, :k]

    data_transformed = vecs.T.dot(data_new)
    return session.run(data_transformed, data_mean, vecs)
    
```

由于后续做预测识别，所以除了转化成低维度的数据，还需要返回平均值以及低维度空间向量。可以看到中间过程平均脸的样子，前几年比较火的各地的平均脸就可以通过这种方式获取，当然这里的维度以及样本比较少，大概只能看出个人脸的样子。

<div style="text-align:center" align="center">
<img src="/images/人脸识别2.png" align="center" />
</div>

其实 data_transformed 中保存的特征脸按照像素排列之后也能看出特征脸的形状。图中有15个特征脸，足以用来做一个人脸分类器。

<div style="text-align:center" align="center">
<img src="/images/人脸识别3.png" align="center" />
</div>

另外在函数 PCA 中用了 session.run 这个函数，这是由于三个需要返回的结果并不是相互独立的，目前的延迟执行模式下提交三次运算会增加运算量，同一次提交则不会，当然立即执行模式以及运算过的部分图的剪枝工作我们也在进行中。

当训练完成之后，就可以利用降维后的数据做人脸识别了。将之前非训练样本的图片输入，转化成降维后的维度表示，在这里我们就用简单的欧式距离判断与之前训练样本中每个人脸数据的差距，距离最小的就是识别出的人脸，当然也可以设置某个阈值，最小值超过阈值的判断为识别失败。最终在这个数据集下跑出来的准确率为 92.5%，意味着一个简单的人脸识别算法搭建完成。

```js
# 计算欧氏距离
def compare(vec1, vec2):
    distance = mt.dot(vec1, vec2) / (mt.linalg.norm(vec1) * mt.linalg.norm(vec2))
    return distance.execute()
    
```

<h4>未来</h4>
上文展示了如何利用 Mars 一步一步地完成人脸识别小算法的过程，可以看到 Mars 类 Numpy 的接口对算法开发人员十分友好，算法规模超出单机能力时，不再需要关注如果扩展到分布式环境，Mars 帮你处理背后所有的并行逻辑。

当然，Mars 还有很多可以改进的地方，比如在 PCA 中对协方差矩阵的分解，可以用特征值、特征向量计算，计算量会远小于 SVD 方法，不过目前线性代数模块还没有实现计算特征向量的方法，这些特性我们会一步步完善，包括 SciPy 里各种上层算法接口的实现。大家有需求的可以在 GitHub 上提 issue 或者帮助我们共建 Mars。

Mars 作为一个刚刚开源的项目，十分欢迎提出其他任何想法与建议，我们需要大家的加入，让 Mars 越来越好。
