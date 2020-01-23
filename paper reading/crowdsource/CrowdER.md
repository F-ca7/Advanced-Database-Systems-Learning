## Crowdsourcing Entity Resolution

1. Entity resolution：在数据库中，找到 指向同一个实体的不同记录。（整合多数据源时很重要）比如两个产品，一个的产品名字段是"ipad 2 16GB"，另一个叫"iPad二代 16GB"，但它们代表的是现实世界中的同一种实体。
2. 提出一种人机混合的方式来处理批量的验证任务。其中，机器负责对所有数据进行粗略的处理再传递，人类只负责验证最可能匹配的（实体）对。
3. 使用了一种 两层的启发式方法（two-tiered heuristic） 来创建批处理任务。
4. 背景：
   - 在众包平台（雇佣普通人去做简单的、不需要专业知识的任务）上，去重de-duplication 是一个重要的应用。从而，Entity resolution的任务可转化为 在支持众包查询处理的系统（如CrowdDb或Qurk）中的query。参考——CrowdDB: Answering Queries with Crowdsourcing
   - CrowdDB: 使用人的输入通过众包来处理(无法被DBMS或搜索引擎回答的)查询请求，
5. 已提出基于机器的自动化算法：**Machine-based** 
   - Jaccard **similarity**：与B交集的大小与并集大小的比值。可以把两个记录视作两个集合，当相似度超过特定阈值后，即可认为是相同的实体。
   - 基于**机器学习**，将之视作分类问题，训练出 重复/不重复 的分类器。将一对记录表示为 由属性相似度组成的特征向量（部分属性）。

6. 人机混合：**Hybrid Human-Machine**
   - 先基于机器进行剪枝，再人力检查（HIT: Human Intelligence Tasks）剩余的难以分辨的pairs。
   - 关键在于 **HIT Generation**：将 记录对 结合到 HIT 的过程
   - 两种**HIT Generation**：1. 基于对pair的；2. 基于簇cluster的

7. 基于簇cluster的HIT Generation 可转化为 k-clique edge covering problem。图中每个顶点代表一条记录，每条边代表一对记录，clique是一个图中两两相连的一个点集。当且仅当 clique可以覆盖相应的边时，基于簇的HIT 才可以check一对记录。

   故问题转化为找到 可以覆盖图中所有边的、最少的k-clique(大小不超过k)。



