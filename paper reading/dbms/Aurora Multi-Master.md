## Aurora Multi-Master

1. 问题：关系型数据库 不是为云环境设计

   - Monolithic architecture

   - Large failure blast radius （也就是宕机后对整个系统的影响很大）

     > when a particular IT service fails, the users, customers, other dependent services that are affected, fall into *Blast Radius*

2. 云环境的数据库：

   - 计算 与 存储 的生命周期不同。为了可扩展性、高可用性，两者要低耦合。
   - 实例规模的调整
   - 集群实例的调整

3. 分两层 Database tier 和 Storage tier

   **Database tier**

   - 通过网络写redo log
   - 无checkpoint （日志就是数据库）
   - 向存储层推送日志
   - master节点读取replica来

   **Storage tier**

   - 高并行的 横向扩展 redo

     > **scale-up(纵向)**
     >
     > 利用现有的存储系统，通过不断增加存储容量来满足数据增长的需求。只增加了容量，而**带宽和计算能力并没有相应的增加**
     >
     > **scale-out(横向)**
     >
     > 升级通常是以节点为单位

   - 根据需求 生成/物化 数据库中的page

   - 即时的redo恢复

4. 主要过程

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200617100450.png)

   1. 存储节点 收到 *Log Record*，将其添加到内存的 更新队列中
   2. 将*Log Record*持久化，并且ACK
   3. 整理record，找到日志的空隙
   4. 从对等节点获取log来填充空隙
   5. 将*Log Record*合并到新的数据块版本中
   6. 定时把 log和新的块版本 快照到S3中
   7. 定时 回收旧版本
   8. 定时 进行数据块的CRC校验

   所有都是异步的

5. 对比

   - Shared disk

     优点：

     - 所有节点的所有数据都可用
     - 容易搭建
     - 多处理器中，缓存一致性很相近

     缺点：

     - Heavyweight cache coherency traffic on per-lock basis
     - 网络代价高
     - 热点block的扩展困难

   - Shared nothing

     优点：

     - query被分解、发送至数据节点
     - Less coherence traffic（只在commit时）
     - 可扩展到多节点

     缺点：

     - commit和成员变更的 代价高
     - 可以有热点partition
     - 跨partition操作很昂贵

6. 多master节点
   - 每个实例都能执行 写事务，不需要与其他协调
   - 实例之间共享 分布式存储
   - 节点的宕机与恢复是独立的
   - 基于page的乐观冲突解决
   - 无悲观锁
   - 无全局commit的协调过程

7. 写冲突

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200618120749.png)

   乐观的冲突解决（绿色的master节点直接回滚重试）

8. 存储节点的机制：

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200618121412.png)

   *Log Record*是由多个master同时产生的，存储节点会检测 写冲突。

   ----------

   基于[AWS re:Invent 2019: Amazon Aurora Multi-Master: Scaling out database write performance](https://www.youtube.com/watch?v=p0C0jakzYuc)

   1. 现有的解决方案：
      - 分布式的lock manager：悲观的同步（重量级锁），对scaling不友好
      - 全局ordering：引入了一个单点的**Ordering Unit**，是scaling的瓶颈所在
      - Paxos leader with 2PC：节点间的coordination是重量级的，当事务涉及多个分片时，性能受很大影响
   2. Aurora 只读节点 与 写节点(Master)的scale out：
      - 只读节点：写节点会向只读节点发送更新的Page cache，然后只读节点会从共享存储中读取数据 （page cache的更新是** physical delta**）
      - 写节点：也就是multi-master，所有master实例都能独立地执行写事务，在共享存储层会有乐观的冲突检测；之后master之间会相互发送更新的Page cache
   3. 三种多节点写的情况：
      1. No conflict：正常执行
      1. Physical conflict：（当两个节点的事务修改了同表的同行）两个节点都是独立执行的，期间不需要交流；一个节点提交成功，另一个节点则Rollback，也就是乐观的解决方案
      1. Logical conflict：当一个节点A修改了表1的一行，它发送page cache更新告诉了节点B，从而节点B知道了这一行被修改了，同时也知道这个事务还没提交。如果此时节点B的客户端要求修改表1的这一行，则会节点B的这个事务会Rollback （传统单机数据库会有一个lock wait，但是Aurora没有分布式的锁）
   4. 一致性模型：
      1. Instance Read-After-Write：一个事务可以看到 当前节点已提交事务的结果，以及其他节点上提交的事务（受replication延迟的影响）
      1. Regional Read-After-Write：一个事务可以看到 当前集群内所有节点已提交事务的结果
   5. Cluster endpoint：
      1. 传统的一写多读中，写节点就是endpoint，其他只读节点都follow它；当writer宕机后，某个reader会成为新的writer，原来的writer会变为reader
      1. 在multi-master中，集群的endpoint是**其中一个可用的writer**，而每个writer实例都有一个**instance endpoint** 指向一个特定的节点。
   6. Workload：由于写冲突是page级别的，比如两个master指向两个数据库 -> 没问题，或者两个master写同一个数据库的两张表 -> 没问题，或者两个master写同一张表的不同分片 -> 没问题，如果这张表无分片 -> 可能会冲突；所以workload的structure很重要。
   7. 在information_schema中，可以看到冲突的情况 -> 从而可以看到workload是不是高冲突的