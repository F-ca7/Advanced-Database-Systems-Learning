## Two-Level Sampling for Join Size Estimation

1. **背景**：Join size的估计在查询优化中十分关键，因为中间结果的大小 是查询计划代价(CPU时间，I/O代价，分布式引擎的通信代价)的一个决定性因素。 *Eﬃcient join query evaluation in a parallel database system* 中指出中间结果的大小 很大程度决定了使用哪个join算法最好，<u>尤其在分布式环境中</u>。

   **Challenge**: join查询通常都包含 在**查询期间**才知道的 特定的选择谓词，因此统计信息时无法考虑这些谓词。

2. **Contribution**: 提出一种 基于**两级采样*two-level sampling***的 join size估计。且实现简单，只需要扫描一趟数据，只依赖基础的统计信息——Lk范数 ℓk-norms、**频繁命中项** heavy hitters。

3. **Conclusion**：在以下join—— 主-外键join，多对多join，多表join中，该算法比先前算法表现得都要好。

4. **Related Work**对比:

   - **Sketching based**: 对每个表在join的属性上 构建一个概述 sketch。当没有谓词时，给出的是join的精确大小；缺点有：

     1. 当**有谓词时，性能急剧下降**，因为添加一个谓词 等价于 在原基础上多一张表的join（查询时创建的虚表）
     2. 对虚表 构建sketch 只适用于<u>**谓词属性的域 domain比较小**</u>的情况

   - **Synopsis**: 通过 小波变换的信号处理技术，对数据进行有损压缩，建立一个由所有数据形成的多维数组，提取该数组的概要synopsis. 缺点有：

     1. 谓词只能是 **等于equality** 或 **有范围约束range constraint**
     2. 当属性的domain很大 或 数据维数很高时，压缩的效果很低

   - **Random sampling**: 对域大小、谓词形式/多少 不敏感。只需要在采样的样本上应用谓词，然后使用样本中符合谓词的元组 来进行join size 的估计。

     采样既可以在离线阶段进行，也可以在给定query的期间进行；前者在查询时的开销更小，后者对于选择性的谓词更有效。

5. 两种最重要的join类型：

   - PK-FK
   - many-many，比如微博中的(Follower, Followee)

6. 已有的采样方法：(sampling是并行的)

   - Independent Bernoulli sampling

     最简单的，每个元组具有独立的概率p。估计值为|S<sub>A</sub>⋈S<sub>B</sub>|/p<sup>2</sup>

     只适用于PK-FK join，且该PK-FK关系必须是有向无环图。

   - Correlated sampling

     独立采样算法忽视了两个表之间的连接关系。相关采样法Correlated sampling通过哈希函数来生成相互关联的样本。h: [u] -> [0, 1]，为随机哈希函数。当一个元组的连接属性值为*v* 且 *h(v) < p* 时，该元组就放入样本中。估计值为|S<sub>A</sub>⋈S<sub>B</sub>|/p

   - End-biased sampling

     与相关采样法类似，在其基础上引入了频率 frequency 的信息；认为一个 join value *v* 的采样概率与其频率 成正比。

   - Index-assisted sampling

     基于索引的采样，使得采样更加集中于 与query相关的元组。

7. **Future work**: 将该估计方法 整合到大型分布式查询处理引擎中。

   

