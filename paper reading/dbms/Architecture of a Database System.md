## Architecture of a Database System

1. 数据库组件：

   ![image.png](https://i.loli.net/2020/02/14/219vGjMC7zXxVFA.png)

   一个query的处理流程如下：

   1. Client 调用API 与 *Client Communications Manager*组件 建立连接（有时，也可能是client和DB server直接连接，比如通过ODBC或JDBC连接协议）

   2. 接受到SQL command后，DBMS为该命令 分配一个计算线程，这个任务由 *Process Manager*完成。该步骤最重要的是，DBMS需要决定 **admission control**: 系统是立即执行该query，还是延迟执行

      > **admission control** 准入控制
      >
      > 高并发的请求会造成极大压力，通过准入控制来保证DB能正常运行。比如：Hash join消耗大量内存等情况。
      >
      > 1. 用户请求到达时，保证Client连接不超过阈值
      > 2. 执行查询计划时，可参考当前的负载状况：
      >    - I/O
      >    - CPU
      >    - 内存

   3. 接着进入*Relational Query Processor*，该模块检查用户是否有权执行该query，接着将SQL文本 编译为 内部的**query plan**，并交给 *plan executor*（由一系列的**operator**组成）

   4. **Operators**通过 *Transactional Storage Manager*组件（负责所有的数据读写操作）来获取数据。存储系统中包括了 磁盘上数据的组织和访问方式（**access methods**）与数据结构（如表和索引）；还包括 **buﬀer management**模块（负责 缓冲池与磁盘的数据交换）。为了保证ACID，在访问数据前，需要从 **lock manager**获取锁；如果query涉及更新的话，需要通过 **logger manger**来保证事务提交后的durability

   5. 计算后，返回结果元组（存放在 *client communications manager*的buffer中）；如果结果数据集很大，client可能会增量式地获取结果。

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/db.jpg)

2. **Process Models**

   四个概念：

   - Process： This single unit of program execution is
     scheduled by the OS kernel and each process has its own
     unique address space

   - OS Thread：由系统内核管理

   - Lightweight thread：由用户空间调度，从操作系统的角度看只有一个线程

     > A **lightweight thread** is a computer program process, normally a **user thread**, that can share address space and resources with other **threads**, reducing context switching time during execution.
     >
     > 用户线程是完全建立在用户空间的线程库，用户线程的创建、调度、同步和销毁全又库函数在用户空间完成，不需要内核的帮助。因此这种线程是极其低消耗和高效的。

   - DBMS worker：为client工作的线程，与客户端连接一一对应

   三种进程模型：

   - Process per DBMS worker

     相对容易实现，易于调试。

     其中比较复杂的是 内存中的数据结构——如lock table，buffer pool 这些共享的数据结构，需要通过操作系统来使所有DBMS进程共享这部分内存。

     比如：Oracle、PostgresSQL。

   - Thread per DBMS worker

     - OS thread
     - DBMS thread

     高并发比较好。但调试起来难，且移植性较差（不同平台的线程实现不同）

     比如：MySQL

   - Process / Thread pool

     - DBMS worker multiplexed over a process pool

       Oracle中可选

     - DBMS worker multiplexed over a thread pool

       比如：SQL Server

3. **Parallel Architecture**

   - Shared-Memory

     所有处理器共享内存地址空间。

     - UMA 统一内存访问

       每个处理器核心共享内存地址空间。

       但核数越多，**总线的带宽成为瓶颈**，且 **访问同一块内存的冲突变多**

     - NUMA

       多核CPU通过一条总线共享内存成为瓶颈，NUMA中，CPU平均划分为若干个Chip（不多于4个），每个Chip有自己的内存控制器及内存插槽；CPU访问自己Chip上所插的内存时速度快，而访问其他CPU所关联的内存的速度相较慢三倍左右。

       可参考[A brief update on NUMA and MySQL](http://blog.jcole.us/2012/04/16/a-brief-update-on-numa-and-mysql/)

   - Shared-Disk

     管理成本低，单节点故障不会影响其他节点

     比如：Amazon Aurora

   - Shared-Nothing 即分布式设计理念

     多个独立计算机组成一个集群（每个节点间就通过网络进行通信）。可扩展性好，成本低（不需要特殊的硬件）

     本质上是 **数据的水平分区**

     需要面对的问题：

     - 分区的方式：哈希还是连续
     - 分布式事务：2PC / 3PC / Percolator
     - 负载均衡：如果一个节点始终为热点的话，会成为读写的瓶颈（需要打散热点）
     - 故障处理

4. **Relational Query Processor**

   四个模块：

   - **Parser**

     主要任务是

     1. 检查query是否正确
     2. 规范化名字和引用，检查表示
     3. 将查询 转化为优化器的内部形式(AST)
     4. 检查用户是否有执行该query的权限

   - **Logical Optimizer** / **Rewriter**

     不改变查询语义的情况下，简化查询。主要任务是

     1. View Expansion 视图展开：对视图的操作，转化为对 原始表的操作

     2. Constant arithmetic evaluation 简化常量运算式
     3. Logical rewriting of predicates 谓词逻辑重写：如 NOT大于 改为 小于等于，节省了一个operator；通过逻辑传递关系，从而利用索引加速
     4. Semantic optimization 语义优化：如冗余连接消除
     5. Subquery ﬂattening and other heuristic rewrites 子查询平面化 和 启发式的重写：如子查询可以转化为

   - **Query Optimizer** / **Physics Optimizer**

     优化的点包括

     1. Plan space 计划空间

     2. Selectivity estimation 选择率估计

     3. Search Algorithms 搜索算法

     4. Parallelism 并行度

     5. Auto-Tuning 自动调优

     **预处理**：将查询预先传入Parser、Rewriter、Optimizer，并把生成的查询执行计划存储起来，在后面的执行中使用。（适用于 带有变量的动态查询）

   - **Query Executor**

     常用迭代器模型，负责执行一个具体的查询计划（包括对基本表的访问 以及各种查询执行算法）

     每个迭代器预分配了固定数量的元组指针，被引用的元组真实存储在

     - 元组在缓冲池页面中
     - 迭代器为内存中的元组分配空间

5. **Storage Management**

   两种管理方法：

   - **raw-mode access**:  DBMS interacts directly with the low-level block-mode device drivers for the disks
   - DBMS uses standard OS ﬁle system facilities

   **Space Control**：顺序读写 比 随机读写 快10~100倍

   两种方法控制空间局部性：

   - Raw: 数据直接存在磁盘设备中，但需要将整个分盘分区分配给DBMS，且 可移植性差（跑分可提高性能）
   - Alternative: 创建一个很大的文件，采用文件中的地址偏移来定位数据（大部分的实现方案）

   **Temporal Control**: Buﬀering

   DBMS除了控制数据在磁盘中存放的位置，还必须控制数据什么时候实际写入磁盘。但是OS提供的缓存机制会有以下问题：

   - DBMS不能在failure后，保证原子性的恢复
   - 现代操作系统的文件系统 对*read-ahead*和*write-behind* 有内置的支持，但这些通常不适合DBMS的存取方式。比如，DBMS层次的I/O组件 可以支持 基于SQL 查询处理时 未来读请求的**逻辑上的预I/O决策**，但在文件系统层面很难做到。
   - 双缓冲 和 内存拷贝时的高CPU开销。DBMS已经小心地管理自己的缓冲 来保证正确性，因此OS提供额外的缓冲是多余的，导致浪费了系统内存 以及 计算资源。

6. **Transaction: Concurrency Control an Recovery**

   带事务的存储管理 通常包括以下4个模块：

   - 并发控制的 Lock Manager
   - 错误恢复的 Log Manager
   - 分离I/O的 Buffer Pool
   - 管理底层磁盘数据的 Access Method

   三大类并发控制的方法：（参考了《数据库事务处理的艺术》一书）

   - 2PL
   - MVCC
   - OCC

   **Log Manager**主要功能：

   - 保证已提交事务的持久性（Redo Log）
   - 协助终止事务的回滚 确保原子性（Undo Log）
   - 系统崩溃时的恢复（Redo Log，Bin log）**TiDB中通过raft-log来做恢复管理**

   > binlog是Mysql sever层维护的一种二进制日志，作用主要有：
   >
   > - 主从复制：Master把它的二进制日志传递给slaves并回放
   > - 数据恢复：通过mysql binlog工具恢复数据
   > - 增量备份

   提升WAL性能——“DIRECT, STEAL/NOT-FORCE”

   - Direct: 数据项原地更新
   - Steal: 允许已修改的缓冲页(unpinned)写入磁盘
   - Not-force: 提交请求成功返回给用户前，缓冲页不用强制flush到数据库
   - 减少日志的大小：只记录逻辑操作
   - 使用Log Sequence Number，记录日志顺序
   - 通过checkpoint周期性记录LSN，从较近的时间点开始恢复

7. **Data Warehouse**

   主要是分析历史数据，关注响应时间——ETL系统（Extract-Transform-Load）

   > ETL负责将分布的、异构数据源中的数据如关系数据、平面数据文件等抽取到临时中间层后进行清洗、转换、集成，最后加载到数据仓库或数据集市中，成为联机分析处理、数据挖掘的基础。ETL是BI项目最重要的一个环节，通常情况下ETL会花掉整个项目的1/3的时间，ETL设计的好坏直接关接到BI项目的成败。

   需要解决的问题：

   - **Bitmap Indexes** 位图索引

     可以通过Bit vector的AND操作，快速得到 and的filter结果。

     | Row     | 1    | 2    | 3    | 4    | 5    |
     | ------- | ---- | ---- | ---- | ---- | ---- |
     | 男      | 1    | 0    | 1    | 1    | 1    |
     | 未婚    | 0    | 1    | 1    | 0    | 1    |
     | AND结果 | 0    | 0    | 1    | 0    | 1    |

     位图索引适合只有几个固定值的列，如性别、婚姻状况等，不适合身份证号、手机号等；适合静态数据，而不适合索引频繁更新的列。

   - **Fast Load** 快速加载

   - **Materialized Views** 物化视图

     三个方面：

     - selecting the views to materialize
     - maintaining the freshness of the views
     - considering the use of materialized views in ad-hoc queries

   - OLAP + Ad-hoc Query

   - Snowﬂake Schema Queries的优化

     通常有查询“customer X bought product Y from store Z at time T”，有一个**central table** surrounded by dimensions, each with a 1-N primary-key-foreign-key relationship to the fact table，这种称为 *star schema*；而多级的star就变为 *snowﬂake schema*。

8. **集群架构**（根据集群共享的程度：即在哪个层上发生共享，这里考虑两个层：查询处理层、存储引擎层）https://www.jianshu.com/p/9e7f3e387379

   - **Shared Disk Failover**

     适用于所有DBMS。所有的DBMS server连接到共享磁盘的同一个节点，节点之间用SAN（Storage Area Network）互联。

   - **Shared Disk Parallel**

     允许多个节点同时访问共享存储，需要保证缓存的一致性（比如Oracle的RAC）。针对CPU与内存带宽进行扩展。

   - **Shared Nothing Active**

     中间件层拦截所有客户机请求并将其转发到独立副本。只读请求在可用节点之间得到平衡，从而达到系统可扩展的效果。

   - **Shared Nothing Certification-Based**

     在无共享集群，可以使用基于认证协议避免主动复制更新事务。每个事务直接在副本上执行，而没有任何先验协调。因此，事务仅需要根据本地的并发控制协议在本地进行同步。仅在提交之前，协调的过程才开始。这个时候，发起协调的副本采用全序组通信原语广播更新。这将导致所有节点采用完全相同的更新序列，然后通过测试可能的冲突获取认证。

9. 