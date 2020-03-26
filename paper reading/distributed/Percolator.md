## Large-scale Incremental Processing Using Distributed Transactions and Notiﬁcations

1. 问题：此前谷歌的网页索引系统采用的是基于MapReduce的全量批处理流程，所有的网页内容更新，都需要将新增数据和历史全量数据一起通过MapReduce进行批处理，带来的问题就是网页索引的更新频率不够快

2. Percolator**为 大规模增量处理 提供了两个主要的abstraction : 

   - 基于可随机访问仓库 的ACID事务
   - 管理增量式计算的 观察者

   每个节点上会运行三个进程：

   - Percolator worker
   - Bigtable tablet server
   - GFS chunkserver

3. Background: 

   - **分布式事务**

     事务的操作位于不同的节点上，需要保证事务的 AICD 特性。四种解决方案：

     - **2PC** 两阶段提交 —— *强一致性*

       通过 **分阶段提交** 和 **日志** 的方式，记录下事务提交所处的阶段状态。宕机重启后，可通过日志恢复事务提交的阶段状态。

       Coordinator会询问参与者 事务是否**执行成功**，参与者返回事物的执行结果。

       如果事务在每个参与者上都执行成功，Coordinator发送通知让参与者提交事务；若未能都成功，协调者发送通知让参与者回滚事务（commit / abort）

       2PC存在如下问题：

       1. **同步阻塞**：所有事务参与者在等待其它参与者响应的时候都处于同步阻塞状态，无法进行其它操作
       2. **数据不一致**：如果Coordinator只发送了部分 Commit 消息，此时网络发生异常，就只有部分参与者接收到 Commit 消息 并提交了事务
       3. **单点问题**：Coordinator若发生故障，所有参与者会一直等待状态，无法完成其它操作

       3PC的改进：

       1. 增加超时机制
       2. 两阶段之间插入准备阶段

     - **TCC** （Try-Confirm-Cancel）补偿事务 —— *最终一致性*

       针对每个操作，都要注册一个与其对应的确认和补偿（撤销）操作。

       1. TRY: 尝试执行业务
          - 完成所有业务检查
          - 预留必须的业务资源
       2. CONFIRM: 确认执行业务
          - 不作任何业务检查
          - 只使用Try阶段预留的业务资源
          - Confirm 满足幂等性
       3. CANCEL: 取消执行业务
          - 释放Try阶段预留的业务资源
          - Cancel 要满足幂等性

     - **MQ 事务消息**

       如 RocketMQ：

       - 第一阶段Prepared消息，会拿到消息的地址。
       - 第二阶段执行本地事务
       - 第三阶段通过第一阶段拿到的地址去访问消息，并修改状态

       RocketMQ的两个概念：
       
       - Half Message，半消息
       
         Producer 已经把消息发送到 Broker端，但是此消息的状态被标记为不能投递，故暂时不能被Consumer消费
       
       - 事务状态回查
       
         可能会因为网络原因、应用问题等，导致Producer端 一直没有对这个半消息进行确认，那么这时候 Broker服务器会定时扫描这些半消息，主动找Producer端 查询该消息的状态
       
       事务消息的实现原理就是基于 两阶段提交 和 事务状态回查，**来决定消息最终是提交还是回滚的**

4. 实现细节：

   - **2PC**

     percolator的分布式事务两阶段提交是通过在BigTable表内部每行数据中添加额外的column来标记相关的锁信息的

     就是在事务开始的时候，会对涉及修改的所有行在 lock column上标记加锁，然后在 data column修改数据内容本身，最后在write column上标记最新成功提交的数据的具体时间戳版本。

   - **事务的驱动框架**

     在Percolator里面是通过一个称为notification的机制来实现的，本质上就是表数据的变更 会被用户编写的Observer观察到，并触发后续的事务

   - **Tradeoff**

     引入了大量额外的读写操作，牺牲了性能，换取系统实时响应能力。

