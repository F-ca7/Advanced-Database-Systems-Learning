## Robust query processing Mission possible

1. 问题：数据库查询可能面临的性能降级是巨大的（比起 在优化过程知道所有精确的输入值，较差的情况可能达到几个数量级的差距）。[比较常见的](https://www.dagstuhl.de/en/program/calendar/semhp/?semnr=10381)原因包括

   - 基数估计误差
   - workload波动大
   - 并发控制冲突
   - 严格的资源管理

   因此提出了Robust Query Processing，近年来的工作考虑到以下环境因素：

   - 内存使用量：buffer pool、heap
   - 并发的磁盘操作导致IO带宽的变化
   - 并发控制导致的锁
   - 检测当前执行计划的问题
   - CPU可用核心数
   - Access path的变化
   - 网络拥堵情况
   - Query之间相互影响，比如热点行、事务日志冲突

2. Robust不同粒度的分类：
   - **Operator**：算子级别，比如以下几篇论文
     - **Smooth Scan**: 自适应的access path选择，可以平滑地在**顺序扫描** 和 **索引扫描**之间转化。
     - **G-join**: 将常见的join算法（NL, hash, sort-merge）给合并到一个统一的框架中，从而优化器不用决定选择哪个
     - **[Flow-join](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/robust%20query/Flow-join.md)**: 用于解决分布式join下的data skew现象。思想是：trade off communication for computation；在初始的运行阶段中，会找到高命中的元组（通过小数据量的近似直方图），再通过**广播**这些元组 来避免失衡
     - **Eddies framework**: 见下篇论文的相关工作
   - **Plan**：粒度提升到整个计划上。当输入的参数值只有运行期才知道时，目前的优化器通常都是 使用有代表性的值（如平均数、中位数）来估计这个参数的分布情况，这样做很容易误差太大
     - **Least Expected Cost**(LEC) plan: 会计算输入参数全分布的期望值，其在整个参数的分布空间上计算代价的期望值，而非只计算输入固定参数后计划的代价，从而在计划层面上分摊了基数估计误差带来的开销。
     - **SEER**
   - **Execution**：主要是保证Maximum Sub-Optimality(**MSO**)，即整个选择率空间中，最差情况下的性能下降
     - **Plan Bouquet**
     - **SpillBoud**
   - **Cost Model**：要注意和基数本身关注点不同——cardinality本身是用来衡量数据的分布和关联特征；而代价模型本身是衡量底层硬件和物理操作的指标

3. 未来方向：
   - **Plan cost function**: 前人的工作中，一般都认为代价函数应该与selectivity呈单调关系（比如，选择出的元组越多，cost越高）。参考*A Concave Path to Low-overhead Robust Query Processing (2018)*
   - **Query-graph sensitive robustness**: 查询图可能呈现出不同结构，比如chain, cycle, star, clique；比如 链状的就比星状的 **优化选择更少**，更容易保证高性能地执行
   - **Robustness benchmark**: 像TPC-DS的是只测性能；而最近的一些benchmark如**OptMark**, **JOB**, **OTT**也没有考虑robustness的方面
   - **ML for component selection**: 在不同运行的场景下，可以使用不同的组件；但如何根据运行环境去做出选择判断——可以结合机器学习。参考 *F. Hueske. Specification and Optimization of Analytical Data Flows (2016)*.
   - **Graceful** **performance** **degradation**: 现有问题是，当运行环境发生微小的改变时（通常是硬件的资源上，文章没给出具体例子，我的理解是比如可用内存突然变小了，或参考概述前文的environments），性能会急剧退化。如何设计出一个算法，对于其中所有性能相关的参数，都可以优雅地进行降级（该算法需要可以被理论证明的）。

4. 来自 Dagstuhl Seminar (2012)，在当时提出 值得探索的方向
   - Smooth operations: 可参考 [Smooth Scan](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/robust%20query/Smooth%20Scan.md)
   - Opportunistic query processing: 不是单一地执行一个计划，而是选出多个不同计划并行执行、
   - Run-time join order selection: 主要是通过利用执行期间得到的中间结果，从而使优化、执行两阶段交织进行，并减小技术估计误差
   - Robust plans: 通过smooth operator让整个计划能有健壮性的保障（不过这里说的robust plan和我理解的不太相同，有另一种理解是 计划本身具备健壮性，这里是通过smooth operation来进行运行时计划的平滑过渡）
   - Assessing robustness of query execution plans: 比较执行计划健壮性的指标
   - Testing adaptive execution: 试着比较 自适应执行方法对性能的影响 以及 基数估计误差带来的性能影响
   - Pro-active physical design: 通过持续的workload分析，来逐渐地调整数据库的物理设计（核心假设是 workload有共性、周期性）
   - Adaptive partitioning: 根据当前的workload来调整底层的partition（当然也属于physical design的一部分）
   - Adaptive resource allocation: 比如运行时的动态内存分配、动态workload调节、自适应地控制并发执行
   - Physical database design without workload knowledge: 在数据批量导入期间来决定 数据库的物理设计
   - Weather prediction: 通过分析当前workload，以及当前系统状态的参数，来控制当前系统性能预期
   -  Lazy parallelization: 静态的优化阶段 给出的并行执行计划是有不小风险的，因为(1) 确切的总工作量是无法预知的；(2) 数据倾斜问题对并行有影响；(3) 运行时的可用资源也是不确定的。所以，可以考虑将是否并行给延到执行阶段决定
   - Pause and resume: 通过pause与resume，使得 重复或浪费的工作最少（甚至可以通过undo一些操作 来恢复到可以resume的阶段）
   - Physical database design in query optimization: 将一部分数据库的物理设计 放在查询优化阶段来做。比如把索引创建给延迟到优化阶段，即只有当创建索引有好处时，才会相应地创建

