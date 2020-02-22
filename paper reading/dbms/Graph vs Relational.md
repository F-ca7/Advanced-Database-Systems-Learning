## A Comparison of a Graph Database and a Relational Database

1. **Motivation**: 对比 如MySQL的关系型数据库 和 如Neo4j的图数据库 哪一种更适合数据起源追踪系统data provenance system

   即 在最终决定数据的存储方式前，通过一系列基准测试来比较 关系型模型 和 图模型。

2. **Related Work**:

   - *Should you go beyond relational databases*(2009)中提出，一个DBA如何决定 是否要从关系型转到NoSQL。满足以下条件的数据 可能更适合NoSQL：(数据起源 就符合下述条件)
     1. 表有很多列，但是有几列 只有少数row才用得到
     2. 有属性表 attribute tables
     3. 有很多 多对多关系
     4. 具有 树状的特征
     5. 表模式经常变动

3. **Conclusion**:

   对于数据起源追踪系统来说，在生产环境的图数据库还有些尚未成熟，尽管它搜索速度很快，但缺乏支持 是其最大的缺点。

4. 实验设计
  - **客观基准测试** Objective Benchmarks

    包括 给定query的处理速度、磁盘空间需求、可扩展性。

    每个query都模拟 数据起源追踪的过程，query的 **参数随机生成**；在每个数据库中执行10次，去掉最大和最小值，取平均（保证 cache或系统进程活动 不会影响时间）

  - **主观评测** Subjective Measures

    包括以下：

    - 成熟度/支持的级别 Maturity/Level of Support

      成熟度即 该系统有多老，经过了多彻底完整的测试。系统越成熟，使用的人越多，工业界、学术界也越多人讨论。

    - 编程友好性 Ease of Programming

      友好性取决于任务类型，当需要Graph traversal时，图数据库更方便，而MySQL可能需要包括循环或递归 来完成。

      通常关系型数据库 可以存储图数据，但图数据库 通常会让关系型数据更难理解。

    - 弹性 Flexibility

      Neo4j是轻量且高效的，通常能在服务器上保持良好的性能；MySQL则是大型的服务器应用，存在于大规模的多用户环境中。

      Neo4j的schema可变性很强；MySQL的则较弱。

    - 安全性 Security

      Neo4j没有内置的安全支持，假设运行环境是可信的，用户管理需要在应用层实现。

  

  