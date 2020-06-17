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

9. Instance Read-After-Write：一个事务 可以看到 当前实例上在其之前提交的所有事务，其他节点上已执行的事务（受replication延迟限制）
10. Regional Read-After-Write ：一个事务可以看到 集群中所有实例上已提交的所有事务