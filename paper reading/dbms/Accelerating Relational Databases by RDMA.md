## Accelerating Relational Databases by Leveraging Remote Memory and RDMA

1. **问题**：内存不足的时候，RDBMS需要强制使用SSD或HDD，极大地降低了性能

2. **Contribution**: 

   - 研究了 SMP(Symmetric Multi-Processing 对称多处理架构) RDBMS 如何利用 **RDMA**和集群中的**remote memory** 来提高性能
   - 将 通过RDMA的remote memory，抽象为轻量的文件API；从而可以轻松地整合到现有的RDBMS中
   - 在SQL Server中实现 并进行benchmark测试

3. **RDMA**：

   Remote Direct Memory Access，远程直接数据存取，就是为了解决网络传输中 **服务器端数据处理的延迟** 而产生的 现代化高性能网络通信技术。Direct: 没有内核的参与，内容都在网卡上；Memory: 在用户空间虚拟内存与RNIC网卡直接进行数据传输不涉及到系统内核，没有额外的数据移动和复制。

   > 传统的TCP/IP网络通信，数据需要通过用户空间发送到远程机器的用户空间。*数据发送方* 需要把数据从**用户应用空间Buffer**复制到**内核空间的Socket Buffer**中。然后**内核空间中添加数据包头**，进行数据封装。通过一系列多层网络协议的数据包处理工作。数据才被发送到到**网卡中的缓冲区**进行网络传输。*消息接受方* 接受从远程机器发送的数据包后，要将数据包从**网卡缓冲区**中复制数据到**Socket Buffer**。然后经过一系列的网络协议进行数据包的解析，解析后的数据被复制到相应位置的**用户空间Buffer**。这个时候再进行系统上下文切换，用户应用程序才被调用。

   TCP/IP通过内核发送消息的劣势：

   - **低性能**：通过内核传递 导致很高的 **数据移动** 和 **数据复制**的开销
   - **低灵活**：所有网络通信协议通过内核传递，很难支持**新的网络协议** 和 **新的消息通信协议** 以及 **发送和接收接口**

   RDMA优势：

   - **低时延**
   - **低CPU开销**
     1. 应用程序可以访问远程主机内存而 **不消耗远程主机中的CPU资源**；远程主机内存能够被读取而不需要远程主机上的进程（或CPU)参与。
     2. 远程主机的CPU的**缓存(cache)**不会被访问的内存内容所填充。
   - **高带宽**

   三类RDMA网络硬件实现：

   - Infiniband：专为RDMA设计的网络，从硬件级别保证可靠传输 ；需要支持该技术的NIC和交换机
   - RoCE：基于以太网的RDMA技术，支持在标准以太网基础设施（**交换机**）上使用RDMA
   - iWARP：基于以太网的RDMA技术 (**RDMA over TCP/IP**) 

   ![](https://www.teimouri.net/wp-content/uploads/2018/06/rdma_operation.jpg)

   ![](https://www.teimouri.net/wp-content/uploads/2018/06/rdma_vs_tcp-768x744.jpg)

   三种队列：

   - 发送队列(SQ)
   - 接收队列(RQ)
   - 完成队列(CQ)

   RDMA是基于消息的传输协议，数据传输都是异步操作

   **读操作**：

   - 本质上是Pull操作，远程系统内存里的数据拉到本地系统的内存里
   - Receiver 提供 **虚拟地址** 和 目标内存的**remote_key**
   - Sender 是完全被动的，不会接受任何通知

   **写操作**：

   - 本质上是Push操作，把本地系统内存里的数据推到远程系统的内存里
   - Sender 提供 **虚拟地址** 和 目标内存的**remote_key**
   - Receiver 是完全被动的，不会接受任何通知

   具体工作流程：Work Request -> Work Queue -> Hardware -> Completion Queue -> Work Completion

   1. 当应用需要通信时，就会创建一条Channel连接（建立在本端和远端应用之间），每条Channel的首尾端点是一对Queue Pair - SQ和RQ
   2. Queue Pair 会被映射到应用的虚拟地址空间，使得应用直接通过它访问RNIC网卡
   3. CQ用来知会用户 WQ上的消息已经被处理完
   4. 提供了一套软件传输接口，方便用户创建传输请求Work Request(描述了应用希望传输到 Channel另一端的消息内容）

4. 细节实现：

   - **Remote Memory**

     四种实现选择（本文使用的是**In-memory blocks**）

     - **Byte-addressable**: 远端内存以**字节寻址**，允许现存的系统无缝地将本机内存扩展到远端内存（和OS提供的虚拟内存类似）
     - **In-memory blocks**: 远端内存 被分为固定大小的块（大小可自定），应用程序通过**唯一的标识符**、**offset**和**读/写数据的大小** 来访问一个block；从而允许读取block中单独的一些字节，而不用读取整个block
     - **In-memory ﬁles**: 操作系统允许将一部分物理内存 挂载为文件系统，如Linux的**ramfs** 和windows的**Ramdisk**。数据库的设计本身就是高效地处理文件
     - **In-memory Key-value store**: 应用只用put和get的API来交互，隐藏了 内存管理 与 远端读写的实现细节

     文件API与RDMA操作的对应关系：

     | 文件操作                      | RDMA实现                |
     | ----------------------------- | ----------------------- |
     | Create (ServerEndpoint, Size) | Obtain lease on MRs     |
     | Open                          | Connect to server       |
     | Read/Write (Offset, Size)     | RDMA read/write         |
     | Close                         | Disconnect from server  |
     | Delete                        | Relinquish lease on MRs |

   - **Architecture**

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200417111313.png)

     通过 **Memory Broker**记录集群中内存的使用情况，每台Server都会向 **Memory Broker**报告可用的内存。Server *M<sub>i</sub>*将可用的内存切分为固定大小的Memory Region 作为带序号的Block。*M<sub>i</sub>*上的一个内存代理进程 **保证available memory不会分配给本机的其他进程**，固定可用的MR，将MR注册到本地的网卡，并从操作系统的角度标记为不可用。
   
     当某台DB server内存不足时，可以向**Broker**请求 远程的MR，**Broker**来决定哪一台Server *M<sub>i</sub>* 来提供MR（其机制类似于Zookeeper，具有容错性 与 高可用性）。
   
   - **Preregistration of MRs**
   
     最大的挑战在于，SQL Server的缓冲池 不是一片连续的内存，且大小可能动态变化。又由于 只有一个内存分配器来 处理 **缓冲池的内存请求** 以及 **其他SQL Server组件的内存请求**，则连续的大片内存中 既有缓冲池的内存，又有其他引擎组件使用的内存；故 如果静态地注册整个缓冲池，可能会直接包含了整个DBMS的地址空间，**会包含大量RDMA不会传输的内存块**。
   
   - **I/O Micro benchmark**
   
     通过**SQLIO**（一个磁盘基准测试工具）来评估随机/顺序读的性能。参考SQLIO Disk Subsystem Benchmark Tool， http://www.microsoft.com/en-us/download/details.aspx?id=20163.
   
   - 四个场景
   
     **性能提升**、**remote memory实现的复杂度** 以及 **整合到DBMS的容易度** 三者的tradeoff。
   
     - **Extending the Caches**
   
       数据库有多种cache: procedure cache(存储 **优化后的执行计划**、**部分执行结果**)、buffer pool cache… 当cache大小达到临界值时，就要淘汰策略。这里，可以不直接丢弃 淘汰的entry，而是把该entry放到remote memory中；当再次访问它时，会比在磁盘上读取更快
   
     - **Spilling Temporary Data**
   
       在执行复杂查询时，DBMS会产生许多临时数据，如果数据量太大，需要把中间结果保存到文件中（比如 SQL Server的 TempDB，Oracle的Temporary Tablespaces）。
   
     - **In-Memory Semantic Caching**
   
       如 非聚集索引、部分索引（*Generalized partial indexes. 1995*）、物化视图等的cache。
   
     - **Priming the Buffer Pool**
   
       云数据库通常使用多个副本 来保证高可用性。典型的，主copy进程处理 读取/更新的事务，其他副本通过 逻辑或物理的replication 来与主节点保持一致。
   
       *primary-secondary swap*（从节点 变为 主节点）可能会发生，当workload在 新选举出来的主节点上执行时，其缓冲池是冷启动的，故性能会有很大损失。
   
       故对于物理replication的数据库，可以使用高速的RDMA传输 来预热(warm up)缓冲池——使用原主节点的缓冲池内容，通过push或根据需要来fetch。
   
   

