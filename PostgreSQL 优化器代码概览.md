# PostgreSQL 优化器代码概览

<h3>简介</h3>

PostgreSQL 的开发源自上世纪80年代，它最初是 Michael Stonebraker 等人在美国国防部支持下创建的POSTGRE项目。上世纪末，Andrew Yu 等人在它上面搭建了第一个SQL Parser，这个版本称为Postgre95，也是加州大学伯克利分校版本的PostgreSQL的基石[1]。

我们今天看到的 PostgreSQL 的优化器代码主要是 Tom Lane 在过去的20年间贡献的，令人惊讶的是这20年的改动都是持续一以贯之的，Tom Lane 本人也无愧于“开源软件十大杰出贡献者”的称号。

但是从今天的视角，PostgreSQL 优化器不是一个好的实现，它用C语言实现，所以扩展性不好；它不是 Volcano 优化模型的[2]，所以灵活性不好；它的很多优化复杂度很高（例如Join重排是System R[3]风格的动态规划算法），所以性能不好。

无论如何，PostgreSQL 是优化器的优秀实现和创新源头（想象 Greenplum 和 ORCA 等一系列项目），它的一些优化手段和代码结构在今天仍然是值得借鉴的，包括：

参数化路径，作用于indexed lookup join
分区裁剪和并行优化
强一致的cardinality estimation保证
本文尝试快速地浏览一遍 PostgreSQL 优化器的代码，和现代优化器比较优缺点。大部分的 PostgreSQL 优化器代码来自于 https://github.com/postgres/postgres/tree/master/src/backend/optimizer 。 我们提到现代优化器主要指的是 Apache Calcite 和 ORCA。

<h4>术语解释</h4>
1.Datum
1.Qual
1.Path
<h4>关键数据结构</h4>
</h4>查询</h4>
1.__Query__: Parse Tree，优化器的输入
1.__RangeTblEntry__: Parse Tree的一个节点，它描述了一个数据集的视图，这个数据集可能来源于某个子查询、Join、Values或任何一个简单关系代数表达式。Join实现需要把它的输入都表达为 RangeTblEntry （以下简称RTE）。
<h4>执行计划</h4>
1.__PlannedStmt__: 执行计划的顶层节点
1.__PlannerInfo__: 优化器的上下文信息。它是一个树形结构，用parent_root变量指向父节点。一个Query包含一个或多个PlannerInfo，每次Join切分一次树节点。它包含RelOptInfo的指针。
1.__RelOptInfo__: 优化器的核心数据结构，包含一个子查询的Path集合等信息。这个概念对应于ORCA的Group或Calcite中的Set。
1.__Path__: 区别于Parser称Relational Expression为Node，Optimizer称优化时的关系代数为Path。Path是物理计划，它包含优化器对于单个关系代数的理解，包括并行度、PathKey和cost。
1.__PathKey__: 排序属性。这个概念相当于Volcano中的Physical Property或Calcite中的Trait。因为 PostgreSQL 是单机数据库，仅用排序属性就可以表达所有算法的需求和实现特性。对于分布式数据库，通常还需要分布属性。
<h4>主流程</h4>
<h4>子查询上拉</h4>

因为优化的单元（RelOptInfo）是子查询，合并子查询可以简化优化流程。关联的过程包括：

1.__pull_up_sublinks__: 将可转换的 ANY/EXISTS 子句转换为 (anti-)semi-join 。一些优化器称这个过程为de-correlation。
1.__pull_up_subqueries__: 将可上拉的子查询上拉到当前查询，删除原来的子查询。如果子查询是一个 Join ，这个操作相当于打平 binary join 到 multi join。
<h4>EquivalenceClass解析</h4>

Equivalence Class（EC）是 qual 的术语，它指代的是 expression 的等价性。例如，expression

```js
a = b AND b = c
```
则称 {a, b, c} 是一个EC。特别地，在 PostgreSQL 中，expression

```js
a = b AND b = 5
```
只生成简化的EC：{a = 5} {b = 5}

EC是很关键的数据结构，它的应用场景包括：

在 Join 时，EC用来决定 Join Key，它决定了 Outer Join 简化和PathKey设定
在 Join 时决定 qual 穿越
决定参数化路径的参数列表
匹配主-外键约束，以便优化（Join的）cardinality estimation
EC是一个树形结构，每个节点是一个EC，并链接到它合并的父节点上。考虑a = b AND b = c的例子，最后的EC tree表达为

```js
{a, b, c}
|- {a, b}
|- {b, c}    
```   
其中，每个EC内部的expression称为EquivalenceMember（EM）。

生成 EC 的入口是 generate_base_implied_equalities ，它从 query_planner 调入。也就是说，EC是在规划Join的前一刻生成的，这个阶段解析EC的代价最小，但是也决定了EC只能应用于Join优化。

<h4>Join重排</h4>

（有关Join重排的背景知识可以参考我之前的文章 SQL优化器原理 - Join重排）

make_rel_from_joinlist是Join重排的入口，当前版本的 PostgreSQL 有三种算法：

你可以插入一个自定义的Join重排算法
GEQO： Genetic Optimization （基因算法，或遗传算法[4]），是一种非穷举的最优化算法实现
Standard：一个略微剪枝的动态规划算法。
默认在12路及以上的复杂Join中会打开GEQO。可以在postgresql.conf中修改参数
```js
geqo = on
geqo_threshold = 12
```
<h4>控制GEQO设定。</h4>

现在让我们检查 Standard 算法。它的主入口在 join_search_one_level ，每次在已生成的局部计划的基础上：

1. 按EC检查未加入的Join input，加入到生成的局部计划，这个操作仅产生 Left-deep-tree
2. 从未加入局部计划的Join input里找到有EC的两个input，生成额外的局部计划，用于生成Bushy-tree
3. 如果当前层找不到任何EC关联，生成笛卡尔积。
上述描述已经足够复杂，让我们总结一下 Standard 算法：

1. Standard 算法仍然是一个穷举的动态规划算法
2. 它对 a-b/b-a 镜像去重，同时当EC存在时不考虑笛卡尔积，这些工程上的降级有效降低了搜索复杂度
<h4>路径生成和动态规划</h4>

如上所述，优化过程集中在对子查询（RelOptInfo）的重建过程，这可以理解为逻辑优化过程，这通常是跨关系代数操作符的、比较复杂的优化。事实上 PostgreSQL 也同步在做物理优化。

物理优化就是将 Path 加入 RelOptInfo。考虑Join，物理优化的入口在 populate_joinrel_with_paths。对每个JoinRel（Join RelOptInfo），考虑：

1.sort_inner_and_outer：两边排序的MergeJoin路径
1.match_unsorted_outer：Null-generating side不排序路径，包括 MergeJoin 和 NestedLoopJoin 。
1.hash_inner_and_outer：两边哈希的HashJoin路径。
有趣的点是HashJoin路径（hash_inner_and_outer），顾名思义，它要求Join两边都计算哈希值。在生成Path过程中，需要计算两边的参数信息。例如A join B on A.x = B.y，对于A来说，x是参数，对于B是y。如果选定A作为Probe side，一旦B上有y的索引，每次x的probe将以参数的形式传递给y的索引。通过调用 get_joinrel_parampathinfo 来产生参数信息。

路径生成的入口是add_path，每次生成路径，需要更新RelOptInfo的最佳路径和最小代价以便后续动态规划选择全局最优。

<h4>流程图</h4>

```js
planner
|- subquery_planner 迭代的子查询优化
    |- pull_up_sublinks de-correlation
    |- pull_up_subqueries 子查询上拉
    |- preprocess_expression Query/PlannerInfo 结构解析，常量折叠
    |- remove_useless_groupby_columns
    |- reduce_outer_joins Outer Join退化
    |- grouping_planner
        |- plan_set_operations SetOp优化
        |- query_planner 子查询优化主入口
            |- generate_base_implied_equalities 生成/合并EC
            |- make_one_rel Join优化入口
                |- set_base_rel_pathlists 生成Join RelOptInfo列表
                |- make_rel_from_joinlist Join重排和规划
                    |- standard_join_search 标准Join重排算法
                        |- join_search_one_level
                            |- make_join_rel 生成JoinRel和对应的Path
        |- create_XXX_paths Grouping、window等其他expression优化
```
<h3>讨论</h3>
<h4>扩展性和灵活性</h4>

首先，PostgreSQL 的优化器代码可以说非常复杂，这已经极大限制了它的扩展性和灵活性。如果看一眼这部分代码的更新日志，会发现里面的作者已经只有少数几个人。

一部分扩展性限制是由编程语言带来的，因为C语言本身不容易扩展，这意味着大部分时候想要添加一个新的Node或Path变得很不容易，你需要定义一系列的数据结构、Cardinality Estimation逻辑、并行逻辑和Path解释逻辑。并没有类似interface这样的抽象指导你该怎么做。虽然，PostgreSQL 的代码已经写得非常工整，而且也有很多的文章告诉你该怎么做（比如 Introduction to Hacking PostgreSQL 和 The Internals of PostgreSQL）。

另一部分扩展性限制是优化器本身的结构带来的。现代的优化器基本都是Volcano Model[2]的（例如SQL Server和Oracle，就像他们声称的那样），而 PostgreSQL 没有实现为 Volcano Model 这种 Generic purpose，pluggable 的形式。影响包括：

1. 无法做逻辑和物理优化的互操作。例如前文说到的，一个Join产生的EC必须和它紧跟的 RTE 结合才能产生 IndexedLookupJoin，而不像其他优化器可以把这个 EC （它在某种意义上已经是物理计划）下推到合适的逻辑计划上，指导它做物理计划转换。
1. 不容易定制优化规则。
1. 开发者关注的切片太大，开发一个优化规则除了关注优化本身，不得不学习其他优化规则的数据结构、动态规划更新、RelOptInfo新建和清理，甚至内存分配本身。
PostgreSQL 仍然提供了部分手写的 Plugin Point，包括：

1. 可定制的Join重排算法
1. 可定制的PathKey生成算法
1. 定制的Join Path生成算法
等等。

<h4>性能 </h4>

虽然没有实验，但是 PostgreSQL 在优化上的性能可以想像是比较好的，这很大程度是用灵活性交换来的。

首先，不像 Volcano Optimizer ，PostgreSQL 优化器不需要不断生成中间节点，它的 RelOptInfo 的数量是相对稳定的（约等于Join的数量）。它的最优计划搜索以 RelOptInfo 为单位，如果 Join 重排不产生大量 RelOptInfo ，搜索宽度很低。

其次，RelOptInfo 简化了大量跨 Relational Expression 优化的细节，比起 Calcite 这种按 Relational Expression 来组织等价路径集合的方案， 它的搜索宽度进一步降低了。从等价集合的数量看， PostgreSQL 的搜索宽度大概比 Calcite 要低一个数量级，当然，如上所述，这是用更多优化可能性作为交换的。

最后，PostgreSQL 在优化阶段糅合了很多业务逻辑，在提高代码阅读的难度同时，也相应加快的优化效率。在优化过程中，PostgreSQL会不间断地做常量折叠、PathKey去重、Union打平、子查询打平……这些操作不会应用在memo里。

对比 Calcite/Orca ，PostgreSQL 的优化更快，更适合事务性场景。不过我无法判断 Calcite/Orca 在做了适当的剪枝和优化规则糅合后，是否也能支持事务场景。

注释
[1] Brief History of PostgreSQL, https://www.postgresql.org/docs/current/history.html

[2] Graefe, G., & McKenna, W. J. (1993). The Volcano Optimizer Generator: Extensibility and Efficient Search. Proceedings of the Ninth International Conference on Data Engineering, (April), 209–218. https://doi.org/10.1109/ICDE.1993.344061

[3] Selinger, P. Griffiths, et al. "Access path selection in a relational database management system." Proceedings of the 1979 ACM SIGMOD international conference on Management of data. ACM, 1979.
