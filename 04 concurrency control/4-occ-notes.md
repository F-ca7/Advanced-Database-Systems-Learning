## Optimistic Concurrency Control

1. 减少 由于使用对话式API 造成的stall：

   - **Prepared Statement**：消除了query准备的开销（query preparation overhead）
   
   > Overhead——overhead is any combination of excess or indirect computation time, memory, bandwidth, or other resources that are required to perform a specific task.
   
   - **Query Batches**：减少网络的往返次数
   
   - **Stored Procedures**：同时减少 准备的开销和网络的时延
   
     数据库系统中，一组为了完成特定功能的 SQL 语句集，这些SQL语句集存储在数据库中，经过第一次编译后，后续调用不需要再次编译，用户通过指定存储过程的名字并给出参数（如果该存储过程带有参数）来执行它。

2. **Prepared Statement**的优点
   - Less overhead for parsing the statement each time it is executed. 很多时候数据库执行的语句都是一样的，只是 WHERE的条件不一样/ UPDATE 设的值不一样 等。
   - Protection against SQL injection attacks. 语句中使用了占位符，规定了sql语句的结构。用户可以设置"?"的值，但是不能改变sql语句的结构。因此可以防sql注入。

3. **Stored Procedures**的优点

   - 减少应用程序与数据库服务器之间的 网络往返
   - 由于query预编译好并存储在DBMS中，可以提高性能
   - 可以在不同应用中被 复用
   - 发生冲突时，服务端的事务会重启

   **Stored Procedures**的缺点

   -  不是大部分开发者都会写SQL/PSM代码
   -  在应用程序的范围之外，难以管理版本与调试
   -  不一定能移植到别的DBMS
   -  DBA不会授予权限

4. 并发控制方案：

   - Two-Phase Locking (2PL 两阶段锁 **悲观**)：假设事务之间会发生冲突，当要访问数据库对象时，必须获得锁。
   - Timestamp Ordering (T/O 基于时间戳 **乐观**)：假设事务之间的冲突概率很小，访问数据库对象时不需要获得锁；而是在提交时检查是否有冲突。

   ![不同的并发控制方案](https://s2.ax1x.com/2020/01/28/1K44Bt.png)

5. **2PL** ——早期的数据库已经通过严格两阶段锁协议（S2PL，Strict Two-Phase Locking）实现了完全的串行化隔离（Serializable Isolation），即正在进行读操作的数据阻塞对应写操作，写操作阻塞所有操作（包括读操作和写操作）。但存在死锁现象：
   - **死锁检测** Deadlock Detection：
     1. 每个事务维护一个队列，队列元素是所有持有 该事务等待的锁 的其他事务。
     2. 一个独立的线程会检查所有的队列是否存在死锁。
     3. 发现死锁时，使用启发式算法来决定终止(kill)哪个事务。
   - **死锁预防** Deadlock Prevention：

6. **Basic 基于时间戳** 协议如下

   - 每个事务到达DBMS时，分配一个唯一的时间戳

   - DBMS为 每一个元组 维护最后一个读/写该元组的事务 对应的时间戳

   -  每次事务访问元组时，会对比元组的时间戳和自己的时间戳，来检测冲突（每次都检测，不是乐观）

   - DBMS需要把一个元组拷贝到 事务的**私有空间**private workspace 来实现**可重复读**repeatable reads

     > Read uncommitted（读未提交）：事务中的修改，即使没有提交，在其他事务也都是可见的。结果：产生脏读。

     > Read committed（读已提交）：一个事务从开始直到提交之前，所做的任何修改对其他事务都是不可见的。这个级别有时候也叫做不可重复读，因为在**一个事务中执行多次相同的查询**，可能会得到**不一样的结果**。因为在这多次读之间可能有其他事务更改这个数据，每次读到的数据都是已经提交的。结果：产生不可重复读。

     > Repeatable read（可重复读）：解决了脏读，也保证了在同一个事务中多次读取同样记录的结果是一致的。但是理论上，可重复读隔离级别还是无法解决另外一个幻读的问题，指的是当某个事务在读取某个范围内的记录时，另外一个事务也在该范围内插入了新的记录或删除了原有的记录，当之前的事务再次读取该范围内的记录时，好像产生幻觉。结果：产生幻读。

     > Serializable（可串行化）：它通过强制事务**串行执行**，避免了前面说的幻读的问题，但由于读取的每行数据都加锁，会导致大量的锁征用问题，因此性能也最差。

-----

### 乐观并发控制

​	把所有改变保存在私有空间中，在提交时才检查冲突再合并。

7. 三个阶段：

   - Read Phase: (不是字面上的只有读操作) 把事务要访问的元组拷贝到私有空间，来实现可重复读；同时还要记录它读/写的集合。

   - Validation Phase: 当事务**COMMIT**时，DBMS检查它是否与其他事务发生冲突。每个事务需要按顺序(global order)获取 其**写集合**对应record的锁。

     - Backward Validation: 检查正在提交的事务的 读/写集合 是否与其他**已提交**的事务有交集。 

       ![](https://s2.ax1x.com/2020/01/28/1KrQ3V.md.png)

     - Forward Validation:  检查正在提交的事务的 读/写集合 是否与其他**未提交**的事务有交集。

       ![](https://s2.ax1x.com/2020/01/28/1KrlcT.md.png)

   - Write Phase: DBMS应用 事务write的修改，并使这些修改对其他事务可见。每个record更新成功后，该事务要释放对应的锁。

8. [时间戳分配](https://dspace.mit.edu/bitstream/handle/1721.1/100022/Devadas_Staring%20into.pdf)
   - Mutex：最坏的选择
   - Atomic Addition：使用CAS(compare-and-swap)来递增一个全局变量
   - Batched Atomic Addition：Needs a back-off mechanism to prevent fast burn
   - Hardware Clock： The CPU maintains an internal clock (not wall clock) that is synchronized across all cores. Intel only. 未来CPU中不一定存在硬件时钟
   - Hardware Counter：目前CPU中未实现

9. [Silo](https://github.com/stephentu/silo)内存型数据库，使用并行的Backward Validation实现可串行化的OCC。两个核心思想：
   - 避免对 **只读事务**的**共享内存** 的所有**写操作**
   - 使用epoch来进行批量时间戳分配

10. Silo实现细节如下：
    - Epoch
      - 时间被划分为**定长**的epoch(40ms)
      - 在同一个epoch内开始的事务 最终会在同一个epoch的末尾**一起提交**
      - 若事务时长超过1个epoch，则要自己刷新才能进入下一个epoch
      - 工作线程只需要在每个epoch的开始和结束 进行同步
    - Transaction ID
      - 每一个工作线程 会基于当前epoch号以及已赋值批次的下一个值 来生成唯一的事务ID号
    - Garbage Collection
      - 协作式线程的垃圾回收Cooperative threads GC
      - 每个工作线程 通过一个**reclamation epoch** 标记一个已删除的对象 （这是一个[内存型数据库](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/02%20in-memory%20db/2-imdb-notes.md)）
    - Range Query
      - 通过 记录事务在索引上的scan集合 来解决幻读问题
      - 在 validation phase 重新执行扫描，看索引是否发生变化
      -  Have to include virtual entries for keys that do not exist in the index to prevent two threads from trying to insert the same key.



