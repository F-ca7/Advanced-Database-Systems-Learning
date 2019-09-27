## Optimizer I

1. 没有优化器能真正产生"最优"计划：是[NP-完全问题](http://www.matrix67.com/blog/archives/105)
   - 使用**估计(estimation)**来预测真实的开销
   - 使用**启发算法(heuristics)**来限制搜索空间

2. 逻辑代数表达式 ------optimizer-----> 最优的、等价物理代数表达式

   - 关系代数表达式等价：两个关系代数表达式 在 **任何**(不能只是存在一个)合法的数据库实例上执行时 产生的元组集合相同。如：

     (A ⨝ (B ⨝ C)) = (B ⨝ (A ⨝ C))

3. 五个设计原则
   - 最优化粒度 Optimization Granularity
   - 最优化时机 Optimization Timing
   - 预编译语句 Prepared Statements
   - 计划稳定性 Plan Stability
   - 搜索终点 Search Termination

4. Optimization Granularity：

   - 单Query：
     - 搜索空间小
     - DBMS通常不会重用query之间的结果
     - 代价模型要考虑 是什么正在运行
   - 多Query：
     - 搜索空间大
     - 当有相似查询时，效率更高
     - 对于 数据/中间结果共享的情况 很实用

5. Optimization Timing

   - 静态：
     - 在**执行前**，就选择最佳计划
     - 计划的质量 取决于 代价模型的准确度
     - 在prepared stmts的帮助下，能分摊到各个执行上
   - 动态：
     - 在query执行时，动态选择计划
     - 多执行时会有 重优化(re-optimize)
     - 难以实现
   - 适应式/混合式：
     - 用一种静态算法编译
     - 若 **估计的误差 > 阈值**，重优化

6. Prepared Statements：三种选择

   <img src="https://s2.ax1x.com/2019/09/17/nIe0HO.png" style="zoom:50%;" /> ==》<img src="https://s2.ax1x.com/2019/09/17/nIniFg.png" style="zoom:50%;" />

   - 重优化：每次调用查询时都进行优化；难以重用已经存在的计划
   - 多计划：对于 params 不同值 生成多种plan
   - 平均计划：选择一个param的平均值，每次调用都用这个平均值

7. Plan Stability：三种选择

   - Hints：允许DBA给optimizer一些优化提示
   - Fixed Optimizer Versions：
   - Backwards-Compatible Plans：

8. Search Termination：三种方式

   - 定时：超过预设时间就停止优化
   - 阈值：直到找到 有比阈值更低代价的 plan
   - 穷尽：没有更多plan的变化方法

9. **最优化搜索策略**：
   - 启发式
   - 启发式 + 基于代价的联接顺序搜索(Cost-based join search)
   - 随机化算法
   - 分层(Stratified)搜索：Planning is done in multiple stages
   - 统一(unified)搜索：Perform query planning all at once

10. 启发式：

    - 尽早执行更严格的选择

    - 在联接前执行所有选择

    - 对 **projection/predicate/limit** 下推(pushdown)

    - 基于 基数的 联接顺序排序

      -----

      优点：

      - 易实现，易调试
      - 对于简单查询很快

      缺点：

      - 在估计一个计划的预期成效时，需要依赖魔法常量
      - 当操作符有复杂的依赖时，几乎不能生成好的计划

11. 启发式 + 基于代价的联接顺序搜索：

    - 初始优化使用静态的规则实现

    - 然后使用动态规划 来决定表联接的最佳顺序： 使用 分治法搜索来 自底向上(Bottom-up) 规划

      ------

      优点：

      - 不用穷举搜索，通常也可以找到合理的执行计划

      缺点：

      - 同 启发式
      - left-deep join tree 不一定总是最优的(只能串行 join，并且由于产生了大量的中间临时表；还有另外一种join order叫做 **bushy join tree**，可以并行)
      - 需要考虑 代价模型中数据的物理性质

12. 随机化算法：(如Postgres的遗传算法)

    - 在所有可能的执行计划构成的**解空间(solution space)**中随机搜索
    - 直到达到一个cost的阈值，或者经过一定长度的时间

    ------

    模拟退火(simulated annealing)算法：

    - 起始点为 纯启发式规则
    - 计算不同操作符的排列组合(如交换两个表的联接顺序)：
      - 只要减少cost，接受之
      - 当cost增加时，**有小概率会接受**之(防止陷入到局部极优值的解，而找不到全局最优解)
      - 如果会违反结果正确性(比如排序的顺序)，拒绝之

    ------

    优点：

    - 防止局部最优解
    - 低内存占用(不需要保留历史记录)

    缺点：

    - 当DBMS选了某个方案后，难以得知它为什么要这么选
    - 需要额外的工作来验证算法的确定性
    - 需要实现 正确性的判断规则(保证每一轮变化的结果都是正确的)

13. 分层搜索：

    - 根据变换规则重写**逻辑查询(logical query)**：不考虑代价
    - 进入动态规划阶段来优化

    -------

    优点：

    - 实际表现很好

    缺点：

    - 难以指定变换的优先级
    - 规则是大问题

14. 统一搜索：(以Volcano 优化器为例)

    - 易于添加新的操作、新的等价规则
    - 把 数据的物理属性 当做第一成员
    - **自顶向下**：使用**分支定界(branch-and-bound)**搜索

    ----------

    优点：

    - 使用声明式(declarative)规则来进行变换
    - 更好地扩展性

    缺点：

    - 不易修改predicate
    - 在优化搜索前，所有等价类都被完全展开 来 生成所有的逻辑运算符

15. 查询优化很困难，甚至NoSQL一开始都没去实现它。

