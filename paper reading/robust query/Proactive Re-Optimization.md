## Proactive Re-Optimization

<small>SIGMOD 2005</small>

1. **Motivation**: 传统优化器采用plan-first execute-next的方式执行查询， 这很大依赖中间结果估计的准确度 来选出好的执行计划。 而随着大数据时代，优化器没有跟上数据库系统对非常大的数据集执行复杂查询的能力，很容易选择次优的计划，导致运行时性能显著下降；学界对应的方法也无非 更好的统计数据、新的优化算法、自适应的执行架构。

   作者认为比较有前景的办法是 *re-optimization*——优化阶段与执行阶段是相互交织的。然而前人提出的re-optimizer都是采用的 被动方式(reactive)，即 先使用传统优化器来生成计划，然后记录运行时统计信息，并根据执行期间在计划中检测到的估计错误 以及 因此产生的次优度 做出相应调整；这样会带来以下缺点：

   - 优化器可能选择 那些性能很大依赖于不确定统计信息 的执行计划，导致很可能发生 重优化
   - 当触发重优化 并 切换计划时，流水线中完成的部分工作将丢失
   - 在查询执行期间快速准确地收集统计信息的能力是有限的。因此，当重优化被触发时，优化器可能会犯新的错误，导致潜在的性能退化

2. **Contribution**: 

   - 提出一种主动的(proactive) 重优化方案来解决以上问题，这个框架原型称为 **Rio**
   - 根据统计估计值计算出 *Bounding Boxes*，来表示这些估计值中的不确定性
   - 在优化阶段，通过*Bounding Boxes* 来生成**健壮和可切换的计划**，以<u>最小化重新优化的需要 和 流水线工作的损失</u>
   - 把 **随机样本处理** 与 **常规查询执行**相结合，在运行时快速、准确、高效地收集统计信息

3. **相关工作**

   - **被动的重优化**：*Efficient  Mid-Query  Re-Optimization  of  Sub-Optimal  Query  Execution  Plans (1998)* 提出的**ReOpt**，*Robust Query Processing through Progressive Optimization (2004)*提出的**POP**框架
   - **用区间的估计来代替确切点估计**：*Least Expected Cost Query Optimization (1999)*提出的最小期望代价（Least Expected Cost），*Novel Query Optimization and Evaluation Techniques (2003)*提出的误差感知优化（Error-Aware Optimization），*Parametric Query Optimization (1992)*、*Distributed Query Adaptation and Its Trade-offs (2003)*等研究的参数化查询
   - **健壮的基数估计** RCE：*Towards a Robust Query Optimizer: A Principled and Practical Approach (2005)* 使用随机采样作为基数估计来应对不确定性，并探索了性能的可预测性。但是，RCE没有考虑将 随机采样和常规的查询执行合在一起，或者在join之间传播随机采样的结果。

4. **核心流程**

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20201222_010028.png)

   1. Bounding Box的生成：

      Bounding Box本质是用来表示统计数据的不确定性。  

   2. 

5. 