## The MemSQL Query Optimizer: A modern optimizer for real-time analytics in a distributed database

1. **问题**：在大规模数据集上的实时计算，对于许多企业来说日益重要。MemSQL (最快的分布式内存关系数据库，数据通过哈希表和skip lists进行组织)，其workload通常是复杂的join、aggregation和subquery。从而要求优化器不仅需要生成**高质量的分布式执行计划**，同时**优化耗时要足够短**。

2. **Contribution:**

   - 如果基于cost的查询重写组件 不考虑分布式开销的话，优化器可能会在分布式环境中 对query rewrite做出糟糕的选择
   - 加强了Enumerator，通过广泛地剪枝搜索空间 来使enumerate过程变快；同时 针对分布式环境实现了启发式算法 来剪枝
   - 提出一种新算法 来分析Join graph 并 找出bushy pattern

3. MemSQL介绍

   分布式内存数据库，擅长 实时分析-事务处理混合型的工作，存储数据的两种方式：

   - in-memory row-oriented
   - disk-backed column-oriented

   使用MVCC以及针对内存优化的无锁数据结构，从而达到高并发读写（在事务操作的同时 进行实时分析）。

   用户数据在MemSQL的集群中有两种存储：

   - *Distributed tables*， rows are hash-partitioned, or sharded, on a given set of columns, called the shard  key,  across the leaf nodes
   - *Reference tables*,  the table data is replicated across all nodes

   Query  Optimizer的三个模块：

   - **Rewriter**

     **SQL到SQL**地对查询进行重写。

   - **Enumerator**

     优化器的中心组件，决定了 **分布式join的顺序**、**数据移动的决策**、**本地的join顺序**，以及 **access path的选择**。

   - **Planner**

     将**逻辑的执行计划** 转化为 一系列 **分布式query 以及data movement的操作**。使用扩展的SQL: *RemoteTables*和*ResultTables* 来表示数据移动的操作

4. 实现细节：

   - 查询优化
     1. query首先解析为*operator tree*，再送入优化器
     2. 首先**Rewriter**分析*operator tree*，若重写后有提升，则apply this change，输出为*operator tree*
     3. 接着送到**Enumerator**，使用 带剪枝的搜索算法，结合 **表的统计数据** 以及 **分布式操作**(如 *broadcasting*和*partitioning*)**的代价** 来生成最优的join order，输出为 *带标注的operator tree*
     4. 最后给**Planner**，生成 *分布式查询执行计划DQEP*(由 类SQL语言描述的步骤)，在数据库不同的partition会并发地执行

   - DQEP - distributed query execution plan

     以TPC-H为例，假设 *customer*表、*orders*表是分布式的表，它们的**shard key**分别为 *c_custkey* 和 *o_orderkey*。Shard key即分片字段，其选择直接决定了集群中数据分布是否均衡、集群性能是否合理。

     > The **shard key** determines the distribution of the collection's documents among the cluster's **shards**. The **shard key** is either an indexed field or indexed compound fields that exists in every document in the collection. MongoDB partitions data in the collection using ranges of **shard key** values.

     假设有如下query

     ```sql
     SELECT c_custkey, o_orderdate 
     FROM orders, customer 
     WHERE o_custkey = c_custkey  
       AND o_totalprice < 1000; 
     ```

     该查询 由一个简单join 和 一个过滤条件组成，故<u>Rewriter无法直接对它应用任何改写规则</u>；接着送到 Enumerator，由于 两个shard keys不与join keys相同，所以需要 data movement才能完成这个join。一种可能的计划是 基于*o_custkey*对*orders*表重新划分，来匹配*customer*表的*c_custkey*。最后，Planner 将逻辑计划转化为DQEP:

     ```sql
     (1) CREATE RESULT TABLE r0 
           PARTITION BY (o_custkey) 
         AS 
           SELECT orders.o_orderdate as o_orderdate, 
                  orders.o_custkey as o_custkey 
           FROM   orders 
           WHERE  orders.o_totalprices < 1000; 
      
     (2) SELECT customer.c_custkey as c_custkey, 
                r0.o_orderdate as o_orderdate 
         FROM   REMOTE(r0(p)) JOIN customer 
         WHERE  r0.o_custkey = customer.c_custkey 
     ```

     上述的两个DQEP步骤 使用了 ResultTable和RemoteTable的SQL扩展。第一步 在本地操作*orders*表，基于连接键 *o_custkey*进行过滤和划分，然后将结果流式地保存到ResultTable *r0*。第二步 根据REMOTE 关键字从分布式数据表中 提取数据，将第一步准备好的数据 通过网络进行移动。

   - **Rewriter**

     - **Heuristic and Cost-Based Rewrites** 

       如 **移除没有被使用的列的投影**(always beneficial)，从而节省I/O和网络资源。类似的启发式规则还有 **Group-By Pushdown**、 **Sub-Query Merging**等。

     - **Interleaving of Rewrites** 重写的交错使用

       一些情况下需要交错地使用rewrite，**Outer Join to Inner Join**(比如 接下来有谓词把Outer表的NULL行都给去除) 、**Predicate  Pushdown**(找到衍生表的谓词，将它们下推到子选择查询中) ，这两者需要考虑使用的先后顺序。

     - **Costing Rewrites**

       Estimate the cost of a candidate query transformation by calling the Enumerator. Enumerator **only needs to re-cost those select blocks which are changed**, as we can reuse the saved costing annotations for any unchanged select blocks. 

   - **Bushy Join**

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200303101247.png)

     参考*Of snowstorms and bushy trees*(2014)，

   - **Enumerator**

     - **Search Space Analysis**

       只考虑每个select块里的join优化，不考虑在select块之间移动join（该操作由Rewriter完成）。

       从最小的表达式 开始优化，渐进式地向更大的父表达式（自底向上）。利用了***sharding  distribution***的性质， **predicate columns of equality joins** 和 **grouping columns**被认为是感兴趣的shard keys；在动态规划的过程中，记录 **每个能使数据分布在感兴趣的分片上 的候选join集合** 的最优代价，

     - **Distributed Costing**

       本地的关系型操作（如 join、grouping等，以及数据移动的操作）；分布式查询引擎支持的Data Movement Operation有 (*R*为移动数据的行数；*D*为移动每条数据的平均开销，取决于一行的大小 与 网络因素；*H*为计算每行的划分hash 的开销；*N*为节点数)

       - **Broadcast**:

         Tuples are broadcasted from each leaf node to all other leaf nodes. 

         其开销为 R\*D

       - **Partition** (also called **Reshuffle**): 

         Tuples are moved from each leaf node to a target leaf node based on a hash of a chosen set of distribution columns.

         其开销为 (R\*D + R\*H)÷N

     - **Fast Enumeration**

       参考 *Query optimization time: The new bottleneck in real-time analytics*，2015

   - **Planner**

     - **Remote Tables**

       允许 叶子结点 像aggregator一样 查询所有的partitions（在简单查询中，只有叶子结点 到 aggregator结点之间 是必要的结点间通信；当query越来越复杂时，就需要 叶子结点之间的数据移动）

     - **Result Tables** 

       A result table is a temporary result set, which stores one partition of a distributed intermediate result.

5. **Related Work**

   - **Massively Parallel Processing** Database

     - SAP HANA

       *SAP HANA database: data management for modern business applications*，2012

     - Teradata/Aster

     - SQL Server PDW

       *SAP HANA database: data management for modern* 
       *business applications*，2012

     - Oracle Exadata

     - Pivotal GreenPlum

       使用*Orca*，模块化的优化器架构(*Orca: a modular query optimizer* 
       *architecture for big data*, 2014)。其 自顶向下的优化器 可以作为standalone optimizer，可以支持不同的计算架构 如Hadoop等。

     - Vertica

       implements an industry strength optimizer for its column storage data that is organized into projections. 实现了 Rewrite、cost-base join order，针对 star/snowflake schemas的优化。

   - GPU co-processor加速查询优化

     *A first step towards GPU-assisted query optimization*，2012

   - 并行化Cascade风格的枚举器

     *Parallelizing extensible query optimizers*，2009

     