## Plan Bouquets

1. **Motivation**: 

   OLAP查询的选择率估计经常与实际相差很大 -> 错误的执行计划选择 -> 响应时间很长。

   导致选择率估计误差的原因有：过时的统计信息、复杂的谓词、算子树的误差传递放大、属性值的独立性分布假设。而且在一些远程数据源的场景中，统计信息很有可能是根本获取不到的。

2. 为应对选择率估计误差问题，前人提出的方法：

   - 改进统计信息

   - 基于反馈的调整

   - 识别对于估计误差更健壮的执行计划 (robust plan)

   - 运行时计划切换：

     - *Robust Query Processing through Progressive Optimization (2004)* 提出的POP（progressive query optimization，渐进式查询优化）

     - *Proactive Re-Optimization (2005)* 提出的Rio框架，在常规的查询处理过程中加入随机采样，以在执行期间快速准确地估计出统计信息；比起传统的Robust cardinality estimation多引入了重新优化的步骤

       ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20201116_084141.png)

   

3. 本文与相关工作的不同之处：

   1. 本文提出的Plan Bouquet方法看上去和上述两种plan-switching方法很相似（本质确实是 **plan-switching**），但有两个关键的不同之处（也是优势所在）：

      - 上述方法以优化器的估计值作为起点，如果估计值误差非常的话，会进行彻底的重新优化；但本文方法是从选择率空间的原点开始（与优化器无关），这带来的好处是保证了执行策略的可重复性

      - 上述方法都是基于启发式，无法给出性能退化的边界。

   2. 比起PQO，如 *Progressive Parametric Query Optimization (2009)*，都是在执行前 通过搜索选择率空间 来得到一个执行计划集合，但是PQO的本质是为了节省对于输入查询的优化时间（在给定了参数之后），而本文方法是强调 降低易错选择率带来的 worst case 性能下降。

   3. 比起 plan-morphing的过渡方式，如 *A pay-as-you-go framework for query execution feedback (2008)*，通过运行时 收集额外地cardinality信息（引入了较小的额外开销），来调整接下来执行计划的结构。但本文的方法不会在执行一个计划P的期间，去调整P的结构。

      另外提一句，In any system that exploits execution feedback for query optimization, there are issues such as **maintenance policy** for updates, **replacement policy** for the feedback cache etc. 不过这些问题与原文研究的无关，没有深入探讨。

4. **Contribution**:

   - 

5. 不足之处：

   - 如果可以知道估计误差比较小，那么事实上把优化器的估计值作为起始点的话 会收敛的更快；所以如果能够先验地知道 比如优化器会低估选择率，bouquet的方法也能够选择该起始点 从而更快收敛

   - 不适用于 对时延敏感的TP查询 （也就是适合AP分析），不适用事务中。所以比较适合归档数据库的决策分析之类。

   - DBA无法使用hint，不过可以参考*Optimizer with Oracle Database 12c Release 2 (2017)* 白皮书中的 Adaptive Query Optimization，毕竟在商业数据库中也引入了类似技术

     > By far the biggest change to the optimizer in Oracle Database 12c is Adaptive Query Optimization. Adaptive Query Optimization is a set of capabilities that enable the optimizer to make run-time adjustments to execution plans and discover additional information that can lead to better statistics. This new approach is extremely helpful when existing statistics are not sufficient to generate an optimal plan

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20201118_120636.png)

   - 对于属性值的分布情况有健壮性，但是如果数据集大小增大很多的话，原来的选择率空间就不再适用。因此，未来可能要考虑如何做到增量式地计算bouquet

   - 当易错选择率维度上升时，复杂度也会呈指数爆炸。不过复杂的查询也不一定代表易错的选择率也很多，比如对于  ‘属性值 op 常量’的谓词，现有技术已经能估计地比较准了；再比如对于PK-FK的join也可以考虑在PK表都能找到匹配；还可以计算 POSP代价函数的偏导数，找到影响较小的自变量 从而忽略之