## Cost Prediction using RNN

1. 预测查询开销的意义：

   - **准入控制** admission control：比如*Q-Cop - Avoiding bad query mixes to minimize client timeouts under heavy loads(2010)*中指出的，在负载超过服务器容量的时候，系统需要智能地决定要处理哪些请求，防止性能的过度降级
   - **查询调度** query scheduling : 比如*iCBS: Incremental Costbased Scheduling under Piecewise Linear SLAs (2011)*中提出的 一种 感知代价的调度算法iCBS，并将 服务水平协议SLA 纳入代价的考虑中（在云环境中，云服务提供商在不同的客户之间会提供不同级别的服务，这也取决于用户自身的需求）
   - **资源管理** resource management: 更合理地分配、管理硬件资源
   - **查询监控** query monitoring: 能够观察或预测数据库系统与查询执行相关的各种指标与参数

2. 问题：

   以往的方案一般都是将代价映射为**逻辑单位**，与执行时间本身难以直接转换。且预测时可能需要一些运行时中间结果的信息，这些是**无法预先获取**到的。

3. 相关工作：

   - *Predicting Multiple Metrics for Queries: Better Decisions Enabled by Machine Learning (2009)*
   
     利用机器学习，来预测查询的多种指标，包括执行时间、访问记录数、磁盘IO。文中对比了多种机器学习方法——聚类、PCA、CCA、KCCA (Kernel Canonical Correlation Analysis)
   
   - *Database Meets Deep Learning: Challenges and Opportunities (2016)*
   
     
   
   - 