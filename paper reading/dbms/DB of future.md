## 未来的数据库

> 总结自：TiDB —— 未来的数据库是什么样的？

1. **例子**：

   创业公司一开始 MySQL+Redis缓存层，随着业务增长越来越快，发现底层MySQL容量有限，难以扩展 => 

   尝试分库分表，Redis集群（业务并发量大，单机Redis的话 网卡、内存容量扛不住） => 

   MySQL做分片扩容，每隔一段时间就要折腾一次

   **痛点**：

   - **受限的扩展能力**

     分库分表、“伪分布式”方案（看上去可以做水平扩展，但需要DBA花费大量精力去维护），Cloud-Native（表面上可水平扩展，但是是受限的）

   - **碎片化**

     后端有各种各样的数据库来面对不同场景（MongoDB、HBase、Cassandra），为什么现在Kafka很重要——作为不同数据库间Pipeline的同步方案；还有在线 与 离线分析

   - **在线业务与分析脱节**

     流式数据处理的方案（Flink、Spark Streaming）不是很灵活，当业务发生变化时，很难做匹配的修改调整；而且在线业务 和 流式处理业务很可能是两拨人在负责

2. **Real-time** HTAP

   **特点**：

   - 业务透明的无限水平扩展能力
   - 业务层不需要妥协：跟原来操作RDBMS一样
     - 支持SQL
     - 支持分布式事务
     - 支持复杂查询
   - 高可用——故障自恢复：比如主节点挂了，不需要手动重启
   - 高性能的实时分析：必须有列式存储引擎
   - Hybrid workload下，实时的OLAP不会影响OLTP事务

3. 场景：

   智能规划预算，Self-driving

   - 自动识别workload，根据workload来自动扩/缩容

     预感峰值到来（比如 快开学的教务系统），则自动启动机器，为热数据创建更多副本 并 重分布数据，提前扩容；当峰值过去之后，再自动回收机器

   - 感知业务特点，根据访问特点决定分布

     比如数据带有明显的地理特征，系统自动根据将数据的地理特征 在不同的Data Center放置

   - 感知查询类型 和 访问频率，从而决定使用**不同类型的存储介质**

     冷数据自动转移到S3这种便宜的存储介质，热数据放在高配的闪存上

     > S3: Simple Storage Service
     >
     > A simple web services interface that you can use to store and retrieve any amount of data, at any time, from anywhere on the web. It gives any developer access to the same highly scalable, reliable, fast, inexpensive data storage infrastructure

4. 数据层的智能调度能力：**Serverless**

   > 无服务器计算是指 构建和运行**不需要服务器管理**的应用程序。
   >
   > 过去是“构建一个框架运行在一台服务器上，对多个事件进行响应”，Serverless则变为“构建或使用一个微服务或微功能来响应一个事件”

   目前行业可能更多处在容器 Docker + Kubernetes，利用IaaS、PaaS和SaaS来快速搭建部署应用。

   也就是设计系统时候，不能说我这个系统是跑在Unix、Windows上之类的，而应该是把**云的基础设施**当做操作系统，未来数据库是跑在Kubernetes上、某个公有云的环境下。