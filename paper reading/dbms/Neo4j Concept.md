## Graph Database Applications and Concepts with Neo4j 

1. Core Concepts: 
   - **Node**
     - 通过**Relationship**连接到其他节点
     - 可以具有一个或多个**Property**
   - **Relationship**
     - 连接两个**Node**
     - 有方向
     - 可以具有一个或多个**Property**
   - **Property**
     - 可存储为 Key-Value对
2. 应用：
   - **Social Graph**
   - **Recommender System**
   - **Bioinformatics**

3. Query: 

   图查询取出数据的一个基本操作是 **traversal**，其与 SQL的主要区别是 traversal是一个局部的操作；图中没有全局邻接的索引，而是每个顶点与边 存储与之相邻对象的索引 （只有在找起始点时，才会使用全局索引）。故整个图的大小 不会影响traversal的性能，而SQL的join操作就会受表大小的影响。

   目前图数据库 没有用于 traversal和insertion操作的 标准语言。目前Neo4j支持的有 Java API, REST接口, Gremlin语言 和 Cypher语言。

   - Gremlin: 是领域特定语言 domain-Specific Language，基于Groovy实现
   - Cypher: 是声明式图查询语言 declarative graph query language 

4. Transaction Management:
   - 所有对Neo4j数据的改变都包含在事务中
   - Query可能可以看到 其他事务的修改
   - 事务中 只有写锁才会被获取并持有
   - 锁是在 node和relationship级别 获取的
   - 核心事务管理系统中有死锁检测

5. 高可用性：

   通过coordination和replication实现master-slave模式，依赖于Zookeeper进行结点间的协调。

6. **Conclusion**: 

   图数据库的使用不是为了取代关系型数据库，只有当需要 高度连接的动态数据模型 a dynamic data model that represents highly connected data，才有用图数据库的必要。

