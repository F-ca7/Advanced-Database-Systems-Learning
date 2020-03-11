## MonetDB/X100: Hyper-Pipelining Query Execution

1. **问题**：在计算密集型的应用（如 decision support、OLAP）中，数据库系统在现代CPU上只能取得较低的 IPC(instructions-per-cycle) 。一般来说，这些workload的计算都是相对独立的，从而CPU可以充分地进行并行流水线；但是 大多数DBMS采取的架构 **阻止了编译器使用最关键的性能优化技术**，从而导致了 低CPU效率。
2. **Contribution**: 
   - 分析了内存数据库 MonetDB 以及其MIL查询语言 的性能
   - 提议将 MonetDB的 column-wise execution和 Volcano-style pipelining的增量式 materialization结合起来
   - 设计并实现了一个MonetDB新的查询引擎 X100，采用了一个 向量化的查询处理模型

3. 实现细节：

   - X100的**目标**

     - 在高CPU效率下 处理大量查询
     - 对于不同领域的应用具有可扩展性，如 数据挖掘、多媒体检索，且达到相同的高效率
     - 可根据disk的大小进行规模扩展

   - **架构**

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200310054018.png)

   - 不同的**bottleneck**

     - **Disk**

       X100的 ColumnBM I/O子系统 针对高效的顺序访问。使用了 **vertically fragmented**的data layout 来减少带宽的需求，有时也会引入轻量的数据压缩。

     - **RAM**

       和Disk一样使用了 垂直的数据布局 以及 压缩数据来节省空间与带宽。包括了 平台相关的优化，比如 SSE（Streaming SIMD Extensions） 预取与 数据移动 的汇编指令。

       > SSE defines two types of operations; scalar and packed. Scalar operation only operates on the least-significant data element (bit 0~31), and packed operation computes all four elements in parallel. SSE instructions have a suffix -ss for scalar operations (*Single Scalar*) and -ps for packed operations (*Parallel Scalar*).
       >
       > **Prefetch Instructions**
       >
       > The prefetch instructions provide cache hints to fetch data to the L1 and/or L2 cache before the program actually needs the data. This **minimizes the data access latency**. These instructions are executed asynchronously, therefore, program **executions are not stalled while prefetching**.

     - **Cache**

       基于 *vectorized processing*的模型，使用了 Volcano-like execution pipeline。X100执行原语的基本单元 是 cache常驻数据的垂直小块 small vertical chunks（比如1000个值）。cache是唯一不用考虑带宽的，因此压缩/解压的过程 发生在RAM与cache的边界上。

     - **CPU**

       向量化的primitives，处理每个元组是独立的。

   - **执行实例**

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200310114642.png)

     1. **Scan**操作符 从Monet BAT(Binary Association Table，包含了[*oid*, *value*]。BAT是一个两列的表，一列为 *Head*，另一列为 *Tail*)中 以vector-at-a-time的形式 取出数据。**只有查询相关的属性才会被扫描**。
     2. **Select**操作符 创建了一个 *selection-vector*，该向量里是 匹配谓词的元组位置。
     3. **Projection**操作符计算了最后aggregation需要的表达式。
     4. 通过*map*原语 计算出相关元组，需要将 *selection-vector*传播到**Aggr**操作符（如虚线框所示）。

   - **数据存储**

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200310115834.png)
     
     垂直存储的缺点是 **update cost越来越高**(一条row的更新 需要对每一列做一次I/O)。X100通过 将每个垂直片视作不可变对象immutable object 来规避这个问题，update则记录在delta区域。
     
     如图中的delete操作，它把要删除的元组ID #5放入了 删除列表；insert操作 则会直接append，从而都只需要一次I/O。当delta区的大小 超过了整个表的一定比例，数据就会重新组织——清空delta区 使垂直片处于最新状态。
     
     垂直存储的优点是 **如果查询会访问很多元组但又不需要所有的属性，则会节省带宽(RAM以及I/O带宽)**。同时 X100还通过 *lightweight compression* 来进一步减少 带宽需求。
     
     > **Memory bandwidth** is the rate at which data can be read from or stored into a semiconductor memory by a processor. 
     >
     > Memory bandwidth that is advertised for a given memory or system is usually the maximum theoretical bandwidth. In practice the observed memory bandwidth will be less than (and is guaranteed not to exceed) the advertised bandwidth. 

4. Related Work:

   - **Volcano**

     Volcano - an extensible and parallel query evaluation system, 1994

   - **MonetDB**

     Monet: A Next-Generation DBMS Kernel For Query-Intensive Applications, 2002

   - **block-at-a-time** query

     Buﬀering accesses to memory-resident index structures, 2003

     Buﬀering database operations for enhanced instruction cache performance, 2004

