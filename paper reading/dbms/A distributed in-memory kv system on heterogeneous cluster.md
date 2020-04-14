## A distributed in-memory key-value store system on heterogeneous CPU–GPU cluster

1. **问题**：同构的多核CPU系统 与KV存储系统的特性不匹配，因为无法提供**高数据并行度** 以及 **高内存带宽**；**有限的计算核心** 不满足唯一的数据处理任务的需求；**cache的层次结构**不适合 大的工作数据集。

2. **Contribution**: 

   - 索引操作是 IMKV 处理系统中主要开销之一，但与传统的多核架构不匹配。所以需要把 该任务迁移到 一个能提供高数据并行度与高内存带宽的架构上。
   - 设计了 *Mega-KV*，将索引的数据结构及相应操作 交给GPU处理，并通过一个针对GPU优化的哈希表与算法，来达到前所未有的性能。
   - 设计了一个 周期性的调度算法，对于不同的索引操作 使用不同的调度策略，来最小化响应时延、最大化吞吐量。
   - 设计了实时的功率管理方案 ，通过动态地调整CPU和GPU的频率，来节省功耗。
   - 在异构的CPU-GPU集群上评估了*Mega-KV*，其具有接近线性的课扩展性。

3. 实现细节：

   - **Decoupling index data structure from key-value items**
   - **GPU-optimized cuckoo hash table**
   - **Periodic GPU scheduling**
   - **Dynamic frequency scaling**

4. **Background**:

   - **key-value store**

     通常IMKV存储系统是在分布式下实现，通过一致性hash来划分不同的节点，而每个IMKV节点是独立工作的。

     三种基本query：

     - GET(key)
     - SET(key, value)
     - DELETE(key)

     workflow中的四个主要操作：

     - **Network processing**：网络I/O，协议解析
     - **Memory management**：内存分配、淘汰策略（如redis中的引用计数算法 和 LRU算法）
     - **Index operations**：搜索、插入、删除
     - **Read key-value item in memory**：仅GET

     索引操作的瓶颈包括：**访问索引结构本身**、**访问存储的kv项**。而**内存的随机访问** 决定了索引的访问时间，因为大数据集的索引可能有几百MB，无法装入CPU的cache。

   - 