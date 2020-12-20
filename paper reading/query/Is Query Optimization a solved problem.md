## IS QUERY OPTIMIZATION A “SOLVED” PROBLEM?

[原文链接](http://wp.sigmod.org/?p=1075)

首先，作者认为 查询优化的"万恶之源"在于 对中间结果大小的估计上（基数，cardinality）。通常来说，在<u>给定的基数下</u>，**代价模型**本身可能最多带来30%的误差；然而**基数估计**可以带来比这高几个数量级的误差。

尽管 学者们提出了很多改进的统计学方法（如 直方图等），并且也确实大大地提升了选择率的估计准确度，然而其他地方（如没办法用统计数据）带来的巨大误差 使得这些提升就没很大作用了。

> 所以作者说 如果审文章时看到 对于谓词直方图的改进，很可能直接把它拒了:sweat_smile:，因为现有在这方面的技术已经足够了。

基数估计的误差究竟主要在哪？

- Host variable / parameter marker，也就是Prepared statement里面的占位符参数（参考[DB2的优化器](https://www.ibm.com/developerworks/data/library/techarticle/dm-0606fechner/index.html)）：由于Prepared statement可能会在不知道参数值的情况进行预编译（通常用平均值来估计），所以会带来很大误差

  > `PreparedStatement` is a sub-interface of `Statement` and allows the usage of **parameter markers** (= placeholders; in other programming languages such placeholders are called **host variables**) -- in contrast to `Statement`. In the case of a `PreparedStatement`, the SQL statement to execute -- parameter markers included -- is first compiled, then values are bound to the parameter markers, and finally the SQL statement is executed

  在这个问题上，主要有PQO（Parametric Query Optimization）问题的探讨。

- Join谓词的选择率

  目前在join选择率估计上的创新工作很少，大部分仍然使用两个join列最大基数的倒数作为选择率（System R开创的），也就是 <u>假设一个较小列的域 是较大列的子集</u>（不过这对于有外键约束的列确实成立，但一般情况下 **join的交集很大程度 随join语义本身的不同 而变化**）。

- 把各种选择率结合起来 计算最后基数的过程

  当谓词包含了多个列的filter时，就需要考虑列与列之间的相关联程度。然而很多优化器会假设多个filter之间是相互独立的，也就是 最终选择率 等于 多个filter选择率的积。作者对这个假设的评价是：<u>大部分时候成立，偶尔不成立</u>。在错误的时候，肯定会发生基数的低估(under-estimation)，这一般比高估更严重，因为会导致优化器选择一些基数较少时的特定操作，而实际运行时却发现性能退化很大。

  所以后来很多数据库系统支持 多列统计信息 ***column group statistics***。

  相对应的，学术界也提出了不同的方法来识别schema中 列组合的关联。比如

  - *CORDS: Automatic Discovery of Correlations and Soft Functional Dependencies (2004)*，在查询执行前，会先从数据中进行采样，然后搜索任意两列的关联性。
  - *Consistent selectivity estimation via maximum entropy (2007)*，其认为*CORDS*只考虑了列与列两两之间的关联（一定程度上也减小了算法的复杂度），而没有三列、四列等作为整体之间的关联。其提出的方案是——使用CORDS的输出作为 应该维护联合统计信息的 *column groups*，从而优化器利用  *column groups*的统计信息 来避免由于独立性假设带来的误差；然后通过查询的feedback机制，来校正错误的选择性估计。

