## MuSQLE: Distributed SQL Query Execution Over Multiple Engine Environments

1. **问题**：过多的框架技术 带来了系统的 **异构性** 与 **复杂性**，所以多引擎的分析 在学术界和工业界都变得越来越重要，即数据存在于多个完全独立的引擎上，需要结合在一起做复杂的分析查询。

   目前的解决方案是 通过以中间件为中心来 优化多引擎的查询，但是需要 **手动地把每个基础引擎的算子、代价模型 给整合起来**。

2. **Contribution**

   - 提出了一个通用的SQL引擎API，可以用于多引擎查询优化
   - 将API整合到 基于代价的多引擎查询优化器中。优化器运行在逻辑层，物理层优化与join的执行 由引擎自身决定。
   - 系统整合了三大引擎：SparkSQL, PostgreSQL, MemSQL。

3. **相关工作**

   - SparkSQL

   - PrestoDB

   - Apache Drill

     支持基于SQL的无结构数据存储 查询。不使用独立的代价模型、统计数据，避免 在本地执行更快的情况下，仍然执行多个子查询。

   - BigDAWG

     提出 islands of information，每个岛对应一个数据模型、查询语言、数据管理系统。每个查询通过 Single-Island 或 Multi-Island Planning进行优化。

   - CloudMdsQL

     提供函数式的类SQL语言，可以使用单独一个query来查询异构的数据存储。重点关注每个数据存储的本地查询语言和引擎的用法。

   - QUEPA

     提出了  Augmented search 和 Augmented exploration两种新的查询方法。

   - SQL++
   - MISO

4. 架构

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200619015545.png)

   右上角的*Metastore* 负责存储每张表的schema以及位置。*SQL Parser*模块 与*Metastore*进行交互来验证用户查询的有效性 并 创建query graph。

   *Multi-engine Optimizer*接收query graph，找到最优的执行计划（考虑因素包括：算子的顺序、查询子图的引擎选择、中间结果的移动），其输出的 执行计划 即为一棵 由SQL与move operator(负责不同引擎中间结果的传输) 组成的树。

   Engine API则使得优化器能够真正地与不同的引擎进行交互。

5. **Engine API**

   分为两类

   - **Execution**

     有以下函数

     1. 发送一个SQL查询到指定的引擎（扩展JDBC 和 ODBC），查询结果放在Spark DataFrame中，可以传输到到其它引擎。
     2. 

   - **Estimation**

     有以下函数

     1. 类似于EXPLAIN语句，获取到 执行时间估计、结果表的统计信息。大部分引擎会返回 **执行计划**、**结果行数**、**执行开销(磁盘与CPU)**。
     2. 获取将Spark DataFrame装载到引擎的估计时间，约等于 表大小*引擎传输速率
     3. 将表与统计信息 相关联（当SQL查询需要用到中间结果，但是对应引擎没有执行的统计信息）


6. 查询优化

   DPccp、DPhyp（参考 G. Moerkotte and T. Neumann, “Dynamic programming strikes back,” in SIGMOD, 2008.），通过自底向上连接 查询子图 来减少搜索空间（每张表用顶点表示，边来记录join条件）

   > **csg-cmp-pair **(connected subgraph-connected complement subgraph)
   >
   > 设G=(V, E)为join graph，S1、S2为V的两个无交集的子集（且均连通）。如果存在边(u, v)∈E，使得u∈S1，v∈S2，则(S1, S2)为 **csg-cmp-pair**。

   每个csg-cmp对 代表了 csg与cmp的 join操作。

   论文拓展了DPhyp算法，来找到多引擎查询的最优join顺序。

7. Cost model

   | 描述                               |
   | ---------------------------------- |
   | 读取一行数据的开销                 |
   | 写入一行数据的开销                 |
   | 做一次hash的开销                   |
   | 广播发送一行数据的开销             |
   | 一次CPU比较的开销                  |
   | 集群的核心数（决定了并行的任务数） |
   | 表的partition数                    |
   | 表的行数                           |
   | Spark的shuffle partition数         |

   