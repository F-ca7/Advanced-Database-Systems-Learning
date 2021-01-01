## Exact Cardinality Query Optimization with Bounded Execution Cost

SIGMOD, 2019

1. Motivation: 还是优化器估计不准确，导致的次优计划问题。

2. 相关术语:

   - **ECQO**(Exact Cardinality Query Optimization)

     在*Exact cardinality query optimization for optimizer testing (2009)*中被提出。通过一个probe阶段来得到 精确基数大小，来生成最优计划。

     给定查询q，代价模型c，如何在没有可靠的基数估计模型

     这里基于了一个假设：<u>当知道了精确的基数大小后，代价模型c的估计本身是可靠的。</u>这个假设不一定总成立，但*Predicting query execution time: Are optimizer cost models really unusable? (2013)*中指出了，比起代价模型本身，**基数估计的问题更大**。

     比如在*How good are query optimizers, really? (2015)*中，就比较了不同优化器在精确基数的条件下 产出的计划。

     前人提出的ECQO算法大部分都是需要 遍历得到计划搜索空间出现过的所有中间表的大小，但缺少运行时性能的保障，比如执行到最差计划可能会导致占用大量系统资源。

   -  **Probe Query** 和  **Probe Order**

     试探查询的作用是 获取中间结果的精确基数。试探顺序是指 执行试探查询时的Join顺序，比如可能对应bushy tree的顺序。这个过程的额外开销很大！所以得考虑离线执行。

3. Contribution: 

   - 提出了一种ECQO算法，通过部分生成 大的中间结果表，来减少执行的开销
   - 证明了算法执行代价上界 为最有代价的函数（类似于SpillBound的MSO）
   - 通过实验证明了比前人工作的优越性（使用了Join Order Benchmark）

4. **核心流程**：

   为了避免直接跑完大的中间表，肯定要给其一个限定的执行预算。所以，这里的思想是：只生成部分的中间表（文章中通过`limit`子句来实现），得到一个基数的下界，也能计算出对应计划的代价下界。

   实际流程是**迭代式**的：

   1. 从最小的代价假设出发，逐渐增加代价预算来执行Probe Query（这一点和SpillBound一样）
   2. 每一轮迭代，都能通过执行Probe Query获取到基数信息（文章使用的是Pg的Explain Analyze功能 来实现获取中间结果的大小）
   3. 根据基数信息，自底向上地计算代价下界；然后再自顶向下地计算 每张表的*剩余部分代价*，即生成最终结果的代价。这两步的结果构成了整体代价的下界
   4. 每张表都有对应的状态*status*
      - pending
      - 
   5. 

   伪代码

   ```c
   输入：查询q, 代价预算增长系数t
   SafeECQO(q, a)
   	// 初始化表的元数据
     R = getRelInfo(q)
     // 初始化每个表的代价预算
   	b = getBudget(q)
     while r.status
       // 基数试探的join顺序
       p = calcProbeOrder(q, R, b)
       if p != null
         // 在限定的代价内执行p
         card = safeExecute(p, b)
         // 更新基数下界
         updateCard(p, b, card, R)
         updateCost(q, R)
         // 更新表状态
         updateStatus(q, R, b ,false)
       else
         // 增加预算
         b = b * a
         // 更新表状态
         updateStatus(q, R, b ,true)
       end if
     end while 
   end func
   ```

   

5. updateCard() 更新基数的过程

6. updateCost() 更新代价的过程

