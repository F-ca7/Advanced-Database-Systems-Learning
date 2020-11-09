## Scalable Garbage Collection for In-Memory MVCC Systems

1. **问题**：支持HTAP的数据库系统通常都使用MVCC，而MVCC会产生很多过期的元组版本，最终需要被回收。但是我们发现HTAP中的GC通常是一个性能瓶颈。在长query中，最先进的GC方法的粒度过粗，从而版本的数量会迅速增加 使得整个系统变慢；同时 标准的后台清理方法 会使系统的工作量突增。

2. **Contribution**: 

   - 提出一种创新的GC方法，*Steam*，来剪枝过时的版本，可以无缝地融合到事务处理过程中，使得GC的开销最小
   - 该方法可以处理混合的workload，并在OLTP上 与state-of-the-art方法进行对比

3. **实现细节**：

   - GC的恶性循环

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200228040503.png)

   在长事务中，所有使用到的version都不能被丢弃，而版本链越长，每次取得版本对应的数据就越慢，从而事务执行时间就越长，导致版本仍不能被回收，从而形成了恶性循环。

   - **Basic Design**

     *Steam*基于HyPer的MVCC实现来扩展，使其更加的robust和scalable。开始一个新事务时，系统把它添加到 *active transaction*列表后。当一个事务提交后，系统再将其移动至 *committed transaction*列表。

     以上的两个list都隐式地 按事务的时间顺序排序，从而可以高效地取到 最小的 开始时间戳（即*active transaction*列表的第一个），则 *committed transaction*中 *commitId* ≤ min(*startTs*)的版本可被安全回收。

     下面的实现重点关注三个方面：**scalability**, **long-running transaction**, **memory-efficient**

   - **Scalable Synchronization** —— scalability

     前面提到的list方法可以在常数时间内进行GC，但是其可扩展性非常有限，因为 维护两个全局列表需要全局的mutex。因此，需要避免全局的数据竞争。如Hekaton使用一个 latch-free的transaction map来避免全局互斥锁。

     而*Steam*遵循了 最好使用不需要同步的算法 的范式 (Latch-free synchronization in database systems: Silver bullet or fool’s gold? 2017)。它使用多条线程来分别管理 事务不相交的子集，每条线程会暴露给外部 自己本地的最小值（使用一个64位的atomic整型存储），如果当前线程没有 活动中的事务，则将值设置为最大的整型；因此只要扫描一次所有线程的本地最小值，就可知道全局的可回收的最小值。

     ![Thread-local](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200228112545.png)

     为了防止线程处于空闲状态而导致 GC被延迟，调度器会周期性地检查线程是否进入空闲状态、如果有必要的话进行GC

   - **Eager Pruning of Obsolete Versions** —— long-running transaction

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200228115214.png)

     在遍历版本链时，无用的版本会使长事务变得更慢；因此设计了 Eager Pruning of Obsolete Versions，在版本不被活动中的事务所需要时，将它们都移除。

     当一个线程进入版本链时，使用如下算法来剪枝掉过时的版本：

     ```c
     input: 有序的active timestamps
     output: 剪枝后的version chain
     curVer := getFirstVersion(chain)
     for a in A
         visVer := retrieveVisbleVersion(a, chain)
         for v in (curVer, visVer)
             // 保证最后一个版本包含了所有属性
             if v.attrs不是visVer.attrs的子集
                 merge(v, visVer)
             chain.remove(v)
     curVer = visVer
     ```

     因为只会在版本record中保存 发生改变的属性（节省内存），所以需要检查visVer是否包含了所有v的属性，若v有多出来的属性，则将其合并到最终版本中。在*Steam*中，每次一个元组更新时就会相应有一次prune（也就是版本链要加入新版本的时候）。

   - **Layout of Version Records** —— memory-efficient

     一条版本的record 应该是空间上和计算上高效的，由3部分组成：

     - *Common Header*
     
       包括
     
       - Type {如Insert/Update/Delete}
       - Version {可见性，在提交时Version就设置为提交时的时间戳}
       - RelationId {当事务回滚时，使用RelationId 和 TupleId 来恢复关系中的元组}
     
     - *Additional Fields*
     
       包括
     
       - Next Pointer {版本链中指向下一个版本}
       - TupleId 
       - NumTuples
       - AttributeMask {每个bit对应该表的一个属性}
     
     - *Payload*
     
       包括
     
       - BeforeImages {只保存改变的属性}
       - Tuple Ids {可以复用Next Pointer来更好地管理批量插入}
     
     

4. **Related Work**: 
   - **Tracking Level**
   
     最细的粒度是**元组级别**，GC在扫描过每个元组后识别出 过时的版本（通常使用一个后台的清理进程来周期性清理）。
   
     有些系统**基于事务**来管理版本。由同一个事务创建的版本 使用相同的commit时间戳，所以可以同时找到多个过时版本 并清理。
   
     而**基于epoch**的系统 将多个事务同一个epoch中，epoch可根据 已分配内存或版本号数目的阈值 来判断是否进入下一个epoch。
   
     最粗的粒度是**表级别**。只对 规定了特定操作的workload适用，比如 stored procedures 和 prepared statements；因此比较少用。
   
   - **Frequency and Precision**
   
     Frequency and precision indicate how quickly and thoroughly a GC identiﬁes and cleans obsolete versions.
   
     *HANA* 和 *Hekaton* 就使用了**后台线程**(周期性调起用于GC)，但是如果频率不够高的话，GC的决定就可能基于过时的信息。
   
     *BOHM* 则以**batch**的形式来管理执行事务，在每个batch的末尾来GC，保证该batch的每个事务已经完成。
   
     而GC的彻底程度 取决于 GC如何辨别version是否可回收。**基于时间戳**的identification <u>不如</u> **基于区间**的方法 彻底。
   
   - **Version Storage**
   
     大多数系统将version records存储在一个全局的数据结构中，如哈希表，从而可以 独立地回收单个版本。
   
     *HyPer* 和 *Steam*直接将版本保存在事务内，即 ***Undo Log***。当事务落后于high watermark是，其所有版本都可以一起回收。总之，用Undo log作为版本的存储也很好，因为Undo log是本来就需要用来回滚的。
   
     > 高水位(high watermark)，通常被用在流式处理领域（比如Apache Flink、Apache Spark等），以表征元素或事件在基于时间层面上的进度。一个比较经典的表述为：流式系统保证在水位t时刻，创建时间（event time）= t'且t' ≤ t 的所有事件都已经到达或被观测到。
   
     *Hekaton*的版本管理则很特别，它不使用连续的表空间；元组的版本只能通过索引来访问（ does not distinguish between a version record and a tuple）。唯一一个在探讨的系统中 使用O2N来排序的；O2N会使 写-写冲突检测更加昂贵，因为事务需要遍历整条版本链 来检测发生冲突的版本。
   
   - **Identiﬁcation**
   
     commit时间戳是单调分配的，则很容易识别过时的版本。
   
     *HANA* 和 *Steam* 使用一种更细粒度、基于区间的方式，保证版本链长度最小，只是实现起来复杂。*HANA* 使用一个*引用计数表* 来追踪所有开始与同一时刻的事务——*Global STS Tracker*
   
   - **Removal**
   
     *HANA*中 整个GC是由专门的后台线程 周期性完成的。*Hekaton* 则在事务处理过程中就可以 进行version的清除，但<u>这只适用于O2N的顺序</u>。
   
     *HyPer*和*Steam*则在**前台**进行GC任务，将GC任务分散在 事务处理的过程间。一旦发现过时的版本，就会在每次commit后将它们回收。

