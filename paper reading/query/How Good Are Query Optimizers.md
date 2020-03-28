## How Good Are Query Optimizers, Really?

1. **问题**：
   - Cardinality estimator有多好？什么时候 估计偏差太大 会导致query变慢？
   - 一个准确的cost model 对于整个查询优化过程有多重要？
   - 枚举的plan space需要多大才行？

2. **Contribution**:
   - 设计了一个有挑战性的workload，Join Order Benchmark(JOB)，由来自 IMDB的21个表组成，并提供了33个查询模板和113个查询。查询中的连接关系大小范围为 5到15个。
   - 第一个 使用真实的数据和查询 且end-to-end 的 join order问题研究。
   -  通过 量化 **cardinality estimation**, **cost model**,  **plan enumeration algorithm** 对查询性能的贡献，为一个查询优化器的完整设计 提供guideline（可以避免许多糟糕设计）。

3. PostgreSQL：

   PostgreSQL的优化器 遵循教科书上的架构。

   - **Join order**通过动态规划来列举（包括bushy tree，但不包括cross product）

   - **Cost model**，由于PostgreSQL是面向磁盘的，其Cost model 结合考虑了CPU 和 I/O的代价，并赋予各自一定权重。而一个Operator的cost 则是 访问磁盘页面(包括顺序和随机)数量 和 在内存中处理数据量 的加权和。

     > PostgreSQL官方文档中——
     >
     > Unfortunately, there is no well-deﬁned method for determining ideal values for the cost variables. They are best treated as averages over the entire mix of queries that a particular installation will receive. This means that changing them on the basis of just a few experiments is very risky.

   - 基表的**基数估计** 使用直方图（分位数统计），最常见的值以及它们的频率，还有domain的基数。每个属性的统计信息 都通过 *analyze*命令 使用关系的sample（参考 [Two-level Sampling](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/query/join%20size%20estimation.md)）来计算。基数估计基于以下假设：

     - 一致性 uniformity: 除了最常出现的值，其他的值都认为出现次数相同
     - 独立性 independence: 在属性上会使用的谓词 是独立的
     - 容斥原理 principle of inclusion: join key的域会有重叠，从而更小的domain能在更大的domain中找到匹配

4. 实现细节：

   - When do bad estimates lead to slow queries?

     查询优化 和数据库的物理设计 紧密联系：索引的类型和数量 严重影响了plan的搜索空间，从而影响了 系统对基数估计误差 cardinality misestimate 的敏感度。

     在实验中，我们将 (有误差的)基数估计 注入到不同的数据库系统中，最后比较 有误差和无误差的基数估计 的执行时间。有很少数的query 在使用无误差的估计后 反而更慢了，这是因为 Cost model本身的误差。

     观察那些没有在合理时间内执行完的query，它们通常都有共同问题：**当基数估计的非常小时**，PostgreSQL的优化器就会 采用 nested-loop join（因为PostgreSQL 完全基于cost来选择join算法，有可能计算出嵌套循环的cost更小，但这样风险很大 high-risk），然而实际的基数很大。
   
     另外，当我们**加入其它索引** 如外键 在属性上时，查询优化问题 会变得更困难。当然并不是说 加入索引反而查得变慢了(偶尔会有这样的情况)，整体性能一般都会显著增加，但是 **索引变多，access path就更复杂，优化器也更难优化**。
   
   - How important is an accurate cost model for the overall query optimization process?
     
   Cost model 为 从搜索空间中选择哪些plans 提供了指导。目前系统中的代价模型都很复杂，比如PostgreSQL中的代价模型 超过4000行的C代码，里面考虑到了各种因素，如：部分关联的索引访问、元组大小 等。
     
   通过替换 PostgreSQL中的cost model来实验，比如替换的cost function为 简单地只考虑query会产生的元组数。实验证明，**cost model的差别造成的影响 远小于 基数估计误差的影响**。
     
   实验中 提出了针对*main-memory*的代价函数，即 不考虑I/O，只考虑 每个operator会得到的元组数目。
     
   - How large does the enumerated plan space need to be?
   
     即 plan enumeration algorithm，有 exhaustive的 也有 heuristic的；它们考虑的 候选计划 数量(即搜索空间大小)不同。
   
     实验使用 一个独立的query optimizer，实现了 DP 以及一系列的启发式算法。实验证明，**基数估计误差 对性能的算法 大于 启发式**，bushy tree的穷举法 比起只对子搜索空间枚举法 对性能的帮助要大一点，但不显著。
   
5. **Future work**:
   - 更深入的实验，比如 研究复杂的access path 的tradeoff
   - 本文主要基于内存型的设定，可以研究磁盘型的，以及分布式的数据库
   - 对于多索引的环境，数据库系统可以结合 更多先进的估计算法(在文献中提及的)，还可以增加 运行时与优化器的interaction



