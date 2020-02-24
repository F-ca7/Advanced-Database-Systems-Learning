## Spanner: Google’s Globally Distributed Database

1. **介绍**：

   - Spanner 是可横向扩展的、全球分布式数据库。
   - a database that shards data across many sets of Paxos state machine (每个状态机都在数据中心里)
   - 从一个类似BigTable的多版本的Key-Value存储演变成了一个带有时间属性的多版本数据库
   - 应用程序可以在细粒度上对 数据的副本数 进行动态的控制：
     - which datacenters contain which data
     - how far data is from its users (控制 读取数据时延)
     - how far replicas are from each other (控制 写入数据时延)
     - how many replicas are maintained (控制 可用性和读取性能)
- 提供外部一致性读写 和 基于某个时间戳的跨数据库的全球一致性读
2. **实现**：
   - Spanner通过目录 *directory* 来管理数据的副本 replication 和本地性 locality，目录也是spanner中数据移动的基本单元
   - 一个Spanner的部署称为一个*universe*
   - Spanner以*zones*(是部署管理 以及 **物理隔离**的基本单元，在一个数据中心可能有一个或者多个zone) 的形式进行组织。每个zone相当于一个BigTable的一个部署集群，一个zone都有一个ZoneMaster和上百数千个的 *spanserver*；ZoneMaster将数据分配给spanserver，spanserver将数据提供给各个Client端。Client端利用每个zone中的*location proxy*来定位 给自己提供数据的 spanserver。

3. **软件栈** Spanserver Software Stack

   自下而上的软件栈

   - **Colossus**

     *Tablet*的状态存储在一系列 以B-Tree组织的文件 和 WAL（Write-Ahead-Log）文件中，所有这些文件都存储在名为*Colossus* (GFS的继任者)的分布式文件系统中。

   - **tablet**

     each spanserver is responsible for between 100 and 1000 instances of a data structure called a *tablet*，tablet实现 (key:string, timestamp:int64) → string 的映射；会将**时间戳信息也记录到key**中，更像是一个支持多版本数据的数据库，而不是一个单纯的KV型

   - **Paxos state machine**

     为了支持replication，每个spanserver 都会 在*tablet*上实现一个Paxos状态机。每个状态机存储 在对应的tablet中 存储该状态机的元数据 和 日志。该Paxos实现 支持 long-lived leaders with time-based leader leases.

   - **Lock Table**

     每个 spanserver通过一个lock table 来实现并发控制。Lock table 中有 **两阶段锁** 的状态，将 一个范围内的所有keys 映射到 lock states.

   - **Transaction Manager**

     每个 spanserver通过一个Transaction Manager 来实现 分布式事务。该transaction manager作为participant leader；Paxos Group中的其他的replica（leader之外的副本）作为participant slaves。

