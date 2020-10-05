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
      
        注意：Paxos 保证的是多个副本的一致性；2PC保证的是保证跨节点事务的原子性。比如需要考虑2PC的一个问题就是：如果参与者发生故障，需要等待；那么对于2PC协调者单点问题，可以利用Paxos协议解决，当协调者出问题时，选一个新的协调者继续提供服务。
      
        所以二者是互补的。
   2. Aurora 只读节点 与 写节点(Master)的scale out：
      - 只读节点：写节点会向只读节点发送更新的Page cache，然后只读节点会从共享存储中读取数据 （page cache的更新是**physical delta**）
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

---------

### 具体如何检测同一行冲突并解决

1. 按照之前说的，Aurora多写是乐观的冲突解决。有两种冲突的情况：

   - 一种是物理冲突，当两个节点的事务修改了同表的同行，两个节点都是独立执行的，期间不需要交流；一个节点提交成功，另一个节点则Rollback。那么这个冲突就是在共享存储层检测出来的，也是一个乐观的解决方案
   - 另一种就是逻辑冲突，因为当一个主节点对一行数据进行修改时，它是会向其他主节点 广播这个更新的page cache，其具体实现是 将这个数据更新的 redo change应用到其他主节点的buffer pool中。在这里绿色的节点B知道了这一行被修改了，同时也知道这个事务还没提交。如果此时节点B的客户端要求修改表1的这一行，则会节点B的这个事务会Rollback 

2. 共享存储层如何检测冲突？

   我们知道其他一些系统 之所以只在单节点上提供写服务，是为了方便在单节点上 检测并避免写入冲突。如果想知道Aurora在共享存储层如何检测冲突，先要知道其存储层的工作原理，在报告里也提到了 Aurora的一个特色就是 日志即数据，**数据库实例只负责向存储系统中写入redo log**，将数据和日志的管理交由底层的存储系统来完成，也就是只有redo log需要通过网络传输。

   由图所示，分为以下几步：

   1. 存储节点 收到 *Log Record*，将其添加到内存的 更新队列中；可以看到日志记录是有多个Master节点发出的。而在添加的过程中，就会进行冲突检测
   2. 将*Log Record*持久化，并且ACK；当然如果检测出冲突，就会reject掉
   3. 整理record，找到日志的空隙
   4. 从对等节点获取log来填充空隙Gap，事实上这里就是为了保证所有存储节点能够到达一致性
   5. 将*Log Record*合并到新的数据块版本中
   6. 定时把 log和新的块版本 快照到S3（Simple Storage Service，亚马逊的云存储服务）中
   7. 定时 回收旧版本
   8. 定时 进行数据块的CRC校验

   那么存储节点 检测冲突的原理其实非常简单，如之前所说，存储节点中是以Page为单位进行管理的（文档中指出每个page是16KB，且拥有一个ID 和 一个 Log Sequence Number）。然后存储节点在在收到 master的日志时，会对比当前存储内的Log Sequence Number 与 日志记录的LSN；如果存储节点这个page的序列号更新，就认为有冲突并reject掉。

   当然这样有一个问题就是，可能两个master修改的不是同一行，但它是同一个page里的不同两行，这里也会检测出冲突（尽管事实上这两个并发更新没有问题）。在下面的官方博客中，它就指出了这个问题，所以表的分片需要有比较好的设计 才能做到更高的性能。