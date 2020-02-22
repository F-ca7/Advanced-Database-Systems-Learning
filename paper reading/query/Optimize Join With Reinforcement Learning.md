## Learning to Optimize Join Queries With Deep Reinforcement Learning

1. **问题**：目前优化器大多数都是利用 启发式算法来剪枝搜索空间，当 代价模型cost model是线性时，启发式算法可以适用；但当cost有 非线性项时，文章指出 启发式的结果会离最优化较远。因此，需要一种由数据驱动 data-driven-way的策略 (根据特定的数据集 和 query)来缩小搜索空间。

2. **Contribution**: 
   - 提出了一种 基于强化学习的优化器 DQ
   - 实现了三种版本的DQ，很容易地集成到现有的DBMS中，包括 Apache Calcite, PostgreSQL, SparkSQL
   - 实验证明，DQ可以达到 原生查询优化器 优化后的代价 和 执行时间，但是在经过学习后，执行速度上显著更快

3. **Related Work**:

   机器学习在数据库中的应用仍然是今年的热点辩题，且这个有争议的话题仍会在接下来几年持续下去。——什么问题适合ML的解决方案？文章认为 查询优化是其中的一个sub-area，即该问题很难解决 且 性能的数量级非常关键；即使是糟糕的学习方案 也只是会让执行变慢，而不会导致错误的执行结果。

   - **Cost Function Learning**

     前人的工作 试图通过 execution的feedback 来修正优化器，如 LEO optimizer(DB2中，2003年)，通过feedback来修正cost model (其底层代价模型是基于直方图的)。Leis等人在评估了各种优化器策略的效果后，指出 challenge仍是 **feedback**和**cost estimation errors**。

     而*Cost Function Learning* 即通过基于统计的学习 来修正或替换现有的cost model。

     作者使用神经网络来预测 单关系谓词的选择率，结果很成功，但是对于一个domain大小为10k的属性，需要有1000个query作为训练集；且难适用于 字符串或其他非数值的数据类型 。

   - **Learning in Query Optimization**

     Ortiz等人使用**RL**来 学习queries的representation (2018)；也有人使用 **DNN**来进行 基数估计 cardinality estimates (2015，2018)；Marcus等人 提出了基于RL的join optimizer (2018)，但其实验的查询类型比本文少。
   
     本文将 Q-learning的动态规划 整合到标准查询优化器中，从而可以进行off-policy learning。
     
     > On-policy: The agent learned and the agent interacting with the environment is the same. —— SARSA
     >
     > Off-policy: The agent learned and the agent interacting with the environment is different. —— Q-Learning
     >
     > In SARSA, the agent learns optimal policy and behaves **using the same policy** such as ε-greedy policy. **Because the update policy is the same to the behavior policy, so SARSA is on-policy.**
     >
     > In Q-Learning, the agent learns optimal policy **using absolute greedy policy** and behaves using **other policies** such as ε-greedy policy. **Because the update policy is different to the behavior policy, so Q-Learning is off-policy.**
     
     原本Marcus等人利用On-policy的方法需要8000个query训练样本，才能达到原生的效果；而DQ 利用off-policy的Q-Learning，只需要80个query样本，降低了两个数量级。
     
   - **Adaptive Query Optimization**
   
     自适应查询处理—— *Adaptive query processing Foundations and Trends® in Databases* (2007)，执行期重优化查询——Proactive re-optimization (2005). 
   
     本文研究的优化是基于固定的数据库fixed database，而DQ提供的自适应性 只是workload级别的（无法是tuple级别的）。 
   
   - **Join Optimization At Scale**
   
     *Adaptive optimization of very large join queries* (2018)中，研究了 大规模的join查询优化。可以使用随机化randomized 的方法，如 QuickPick算法 和 遗传算法，当表到达一定数量后，商用的优化器中就会使用这些方法。但是这一类方法 对于同一个查询的性能可能有很大变化。
   
     > QuickPick performs biased sampling by selecting edges from the join graph and adding the respective joins to the query plan. 
     
     还有松弛启发式方法 是 在简化的假设下来解决问题，如 贪心搜索时 避免笛卡尔积，这是 IK-KBZ算法的前提(*Optimization of non-recursive queries*, 1986)。现有的启发式算法不能很好地解决所有非线性项，而正是机器学习 可以对此有所帮助。
   
4. **实现细节**：

   - **Join Order Benchmark** (JOB)

     参考 *How good are query optimizers, really* (2015)。由来自 IMDB 的 21 个表组成，并提供了 33个查询模板 和 113个查询。查询中的连接关系大小范围为 5~15。

   - **Query Graph**
   
     无向图，关系*R*为顶点，连接谓词*p*为边，记*k<sub>G</sub>*为G的连通分量数。则 join 两个子计划 subplan，即 选择两个连接的顶点，并将合并为一个顶点。
   
   - **Markov Decision Process** 马尔可夫决策过程
   
     每个join query都可以用 Query Graph来描述。
   
     | 符号            | 定义                                    |
     | --------------- | --------------------------------------- |
     | 状态 *G*        | Query graph，是决策过程的一个状态 state |
     | 动作 *c*        | 即 join                                 |
     | 下一个状态 *G’* | 经过一次join的query graph               |
     | *J(c)*          | 评估join的代价模型                      |
   
     状态转移 state transition *P(G, c)*即为顶点合并的过程，而奖励 reward 定义为负的代价 即 -J。
   
     MDP的输出为 **将给定query graph 映射到 最佳的下一次join 的函数**。
   
   - **Greedy Operator Optimization**
   
     朴素的优化算法，主流程分三步 (1)从query graph开始；(2)找到最低代价的join；(3)更新query graph；重复以上直到最后只有一个顶点。但是贪婪算法 **不会考虑局部决策对未来的影响**，而有的时候 需要牺牲短期收益 来获取长期利益。
     $$
     \frac{\partial J(\theta)}{\partial \theta_i} = \frac{1}{m}\sum_{k=1}^{m}[x^{(k)T}\theta - y^{(k)} ](x_i^{(k)}) + \frac{1}{m}\lambda\theta_i
     $$
   
   - **Long Term Reward**
   
     考虑最优化问题
     $$
     V(G)=min\sum_{i = 0}^{t}J(c_{i})
     $$
     Q函数的递归定义
     $$
     Q(G,c)=J(c)+min_{c'}Q(G',C')
     $$
     使用 强化学习来接近全局的Q-function，可以得到join优化的多项式时间算法。
   
   - 应用 RL
   
     选择Q-Learning（而不是其他强化学习）的三个理由：
   
     1. Q-Learning允许我们 在训练时 利用最优的子结构 optimal substructure，从而大大减少需要的数据
     2. 比起Policy-Learning，Q-Learning 在每个subplan中会输出 每个join的分数 *score*，而不是直接输出最优的join选择
     3. 评分模型支持 TopK 排序，不是简单地得到最优的plan
   
     *Deep reinforcement learning with double q-learning (2016)* 中提出了Q-Learning的变种，留给Future work
   
   - 模型训练
   
     使用 多层感知器神经网络(MLP) 来表示 Q-function。基于经验，使用 **两层MLP** 可以在适中的时间内训练得到最好的效果（使用SGD算法进行训练）
   
   - 模型使用
   
     在完成训练后，DQ 可以接受纯文本的 SQL 查询语句，将其解析为抽象语法树，对树进行特征化，并在每次候选连接获得评分时调用神经网络。最后，可以使用来自实际执行的反馈定期重新调整 DQ。
   
   - Fine-tuning
   
     在深度学习中，实践中由于数据集不够大，很少有人从头开始训练网络。常见的做法是使用预训练的网络，来重新fine-tuning（也叫微调）。
   
     而DQ可以在执行的过程，通过fine-tuning来 再训练自己，从而更适合运行环境。
   
   - 三种Cost Model
     - CM1: 适合内存型数据库
     - CM2: 额外考虑 有限内存内的hash join（超出阈值，需要考虑 partition的开销）
     - CM3: 额外考虑 复用 已建好的hash表
   - 进行对比的baseline
     - QuickPick1000: 选择1000个随机join计划中最好的
     - IK-KBZ: 多项式时间启发式方法，将query graph拆解为链状，再进行排序
     - Right-deep
     - Left-deep
     - Zig-zag
     - Exhaustive with the indicated plan shapes
   
   

