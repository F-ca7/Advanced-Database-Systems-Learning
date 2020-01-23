## InnoDB存储引擎

1. MySql由以下几部分组成：
   - 连接池 Connection Pool
   - 管理服务和工具 Management Service & Utilities（如备份, 恢复, 安全, 数据迁移等）
   - Sql接口 Sql Interface(DML: 数据操作语言(insert/update/delete), DDL: 数据定义语言(create), Triggers等)
   - 查询分析器 Parser
   - 优化器 Optimizer
   - 缓存与缓冲 Cache & Buffer
   - 插件式存储引擎 Pluggable Storage Engine (如MyISAM, InnoDB, NDB等)
   - 物理文件 File System
2. 存储引擎是**基于表**的，不是基于数据库的！插件式的架构提供了一系列标准的管理和服务支持，存储引擎是底层**物理结构**的实现。
3. InnoDb(默认的存储引擎)的特点:
   - 支持ACID事务 (不是所有存储引擎都支持事务)
   - 行锁设计
   - 支持外键
   - 非锁定读(默认读取不会产生锁)
   - 使用多版本并发控制(MVCC), 使用next-key locking 避免幻读
   - 插入缓冲(insert buffer) => **性能提升**、二次写(double write) => 数据页的**可靠性**、自适应哈希索引(adaptive hash index)、预读(read ahead)
   - 采用聚集(cluster)的方式存储数据; 每张表的存储都按主键顺序进行存放(没有主键时, 会为每行生成一个6byte的ROWID作为主键)
   - InnoDb更适合OLTP, MyISAM更适合OLAP

4. InnoDB采用**多线程**模型，运行时有多个后台线程，它们分别负责不同的任务:  

   - Master线程: 负责将缓冲池的数据**异步刷新**到磁盘，保证数据的一致性。包括脏页(内存数据页和磁盘数据页上的内容不一致)的刷新、合并插入缓冲(INSERT buffer)、UNDO页(分为insert undo log和update undo log用Undo Log来实现MVCC)的回收等。

   - IO线程: 使用AIO(Async)来处理写IO请求，以提高数据库性能。IO线程负责的是这些IO请求的回调。可使用SHOW ENGINE INNODB STATUS查看：

     ![](https://s2.ax1x.com/2020/01/09/lWg958.png)

   - Purge线程: 为什么需要Purge——得从并发机制说起，InnoDB的多版本一致性读是采用了基于回滚段的的方式。无论是更新还是删除，都只是设置记录上的deleted bit标记位，而不是真正的删除记录；后续这些记录的真正删除, 是通过Purge后台线程实现。

     Purge: Purge parses and processes **undo log** pages from the **history list** for the purpose of removing clustered and secondary index records that were marked for deletion (by previous DELETE statements) and are no longer required for **MVCC** or **rollback**. Purge frees undo log pages from the history list after processing them.

   - Page Cleaner线程: InnoDB 1.2.x版本引入，将原本Master线程中的脏页刷新操作放到单独的线程中完成，减轻对于用户查询的阻塞。

5. 基于磁盘的DBMS通常使用**缓冲池**技术来提高整体性能。

   - 数据库读取页时，首先把从磁盘读到的页存放到缓冲池中；下一次再读相同的页时，先判断页是否在缓冲池中，若是则命中，若不是则再从磁盘读。
   - 数据库修改页时，首先修改缓冲池中的页，然后再以一定的频率刷新到磁盘上。不是每次页更新就会将页从缓冲池刷到磁盘！是通过**checkpoint**机制刷新回磁盘。checkpoint需要决定每次刷新多少页到磁盘、每次从哪里取脏页、什么时间触发。InnoDB中有两种checkpoint: 
     1. Sharp Checkpoint: 在数据库关闭时，将所有脏页刷新回磁盘。
     2. Fuzzy Checkpoint: 运行时，只刷新一部分脏页；否则可用性会受到影响。
   - 多个缓冲池实例，每个page根据**哈希来平均分配**到不同的实例中——减少了数据库内部的资源竞争，增加数据库的并发能力。

   ![](https://s2.ax1x.com/2020/01/09/lfMGc9.png)

   - 缓冲池通过**LRU**算法来管理。最频繁使用的页在LRU列表的前端，当没有空位存放新读取的页时，收件释放LRU列表尾部的页。此外，InnoDB采用了**midpoint insertion**策略——新读取的页不是直接放到首部，而是放到midpoint的位置(默认是列表长度的5/8处)。在分割点前的列表称作new列表，可视作是最为活跃的热点数据。

     ![](https://dev.mysql.com/doc/refman/5.6/en/images/innodb-buffer-pool-list.png)

     直接用朴素的LRU算法有什么劣势？当某些Sql操作需要访问表中的许多页甚至全部页时，而这些页通常又仅在这次查询中需要，并不是活跃的热点数据。如果直接放入LRU首部，就很可能把原本需要的热点数据页移除。
     
     下图中，pct决定midpoint的位置，37即37%(3/8)，即新页插入到LRU尾端37%的位置；time表示页读取到midpoint后要等待多久才被加入到new列表中: 

   ​		![](https://s2.ax1x.com/2020/01/09/lf3Sgg.png)

   - 支持压缩页的功能，即把原16KB的页压缩为1/2/4/8KB，并通过unzip_LRU列表管理。不同的表压缩比率不同，所以会有的表页大小为8KB，有的为4KB，如何从缓冲池分配内存？——unzip_LRU列表对不同大小的页会分别管理，并且使用伙伴(buddy)算法分配内存。

     

6. Master线程: 

   - 最高的优先级别

   - 由多个loop组成: 主循环、后台循环、刷新循环 flush、暂停循环 suspend。

   - 主循环包括两类操作：每1秒执行的操作和每10秒执行的操作(通过thread.sleep实现，故并不精确)。

     每1秒的操作(依次)包括: 日志缓冲刷新到磁盘(总是，即使事务还未提交，日志也会刷新到redo log中)、合并插入缓冲(可能，会判断当前的IO压力)、刷新缓冲池中的脏页到磁盘(可能，会判断脏页的比例是否超过阈值)、如果没有用户活动则切换到后台循环。

   - 后台循环，在数据库空闲时或数据库关闭时发生，包括: 删除无用的undo页(总是)、合并插入缓冲(总是)、跳回到主循环(总是)、不断刷新页知道符合条件(可能，会跳转到flush loop中)

   -----

   新版本的优化点: 

   - 引入参数innodb_io_capacity，用来表示磁盘IO的吞吐量，并用其控制**刷新缓冲区脏页**的数量和**合并插入缓冲**的数量(原本刷新脏页的数量、以及合并插入缓冲的数量是有上限的)。若使用SSD类磁盘，即存储设备IO速度较快时，可以将innodb_io_capacity调高，使之匹配磁盘IO吞吐量。
   
   - 引入参数innodb_adaptive_flushing，影响每秒刷新脏页的数量。原本刷新规则是——当脏页在缓冲池的比例小于innodb_max_dirty_pages_pct时，不刷新脏页；大于时，刷新100个脏页。现在是——通过判断redo log的速度来决定最合适的刷新脏页数量。
   
     注: innodb事务日志包括redo log和undo log。redo log是重做日志，提供前滚操作；undo log是回滚日志，提供回滚操作。Redo log写入磁盘时，必须进行一次操作系统fsync操作，防止redo log只是写入操作系统磁盘缓存中。一般情况下，对硬盘的write操作，更新的只是内存中的页缓存，而脏页面不会立即更新到硬盘中。
   
     The [redo log](https://dev.mysql.com/doc/refman/5.5/en/innodb-redo-log.html) is a disk-based data structure used during crash recovery to correct data written by incomplete transactions. During normal operations, the redo log encodes requests to change table data that result from SQL statements or low-level API calls. Modifications that did not finish updating the data files before an unexpected shutdown are replayed automatically during initialization, and before the connections are accepted.
   
   - 引入参数innodb_purge_batch_size，控制每次full_purge回收的undo页的数量。

7. 插入缓冲: 对于非聚集索引的插入或更新，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入索引页；若**不在，则先放入到一个Insert Buffer对象中**（Changes are only recorded in the change buffer when the relevant page from the secondary index is not in the **buffer pool**）。

   - 使用的两个前提: 1. 索引是辅助索引(二级索引) secondary index——插入时，数据库中该非聚集索引已经插到叶子结点，但实际上其只是存放在另一个位置，然后再以一定频率进行Insert Buffer对象和辅助索引页子结点的merge；2. 索引不是unique——因为在插入缓冲时，不会去查找索引页来判断插入记录的唯一性

     注: 二级索引——按照每张表创建的索引列创建一棵B+树，叶子节点并不包含行记录的全部数据(一张表可以有0/1/n个辅助索引)。查找过程为：遍历辅助索引→找到叶节点→找到指针获取主键索引的主键→通过主键索引找到对应的页→找到一个完整的行记录。

   ----------

   insert buffer现在变为**change buffer**，因为现在除了INSERT，还支持UPDATE和DELETE的DML语句。删除的两步: 1. 将一条元组标记为已删除；2. 真正将记录删除。

   - 其数据结构为一棵**B+树**。以前版本是一张表有一棵Insert Buffer B+树，现在是全局只有一棵，负责所有表的辅助索引进行插入缓冲。

     其**非叶结点**存放的是查询的search key。search key的属性包括4字节的space，表示待插入记录所在表的表id(每张表有一个唯一的space id)；1字节的marker，兼容作用；4字节的offset，表示页所在的偏移量。

     当一个secondary index要插入到page(space_id, offset)时，如果该page不在缓冲池中，则会先构造一个search key，再查询Insert Buffer B+树，再将该条record插入到树的叶子结点中。

8. double write：（有些文件系统本身就提供了部分写失效的防范机制，就不需要开启double write）

   - 部分写失效 partial page write: 正在写入某个页到表中，比如16KB的页只写了前4KB，就发生了宕机，则会导致数据丢失

   - double write的两部分——**内存中**的double write buffer和**磁盘上**共享表空间中连续的128个page。（连续空间 => 顺序写 => 效率高）

     - 共享表空间：InnoDB的所有数据保存在一个单独的表空间里面，而这个表空间可以由很多个文件组成，一个表可以跨多个文件存在，所以其大小限制不再是文件大小的限制（**优点**），而是其自身的限制。
     - 独立表空间：每一个表都将会生成以独立的文件方式来进行存储，每一个表都有一个.frm表描述(结构)文件，还有一个.ibd文件。

     当要刷新脏页时，不直接写磁盘；通过**memcpy**函数将脏页复制到内存中的double write buffer，之后通过double write buffer再分2次，每次写入1MB到共享表空间doublewirte中，然后马上调用fsync函数，同步到磁盘上。

     如果写入磁盘时崩溃，可以从共享表空间中的doublewirte中找到该page的副本，将其复制到表空间文件，再应用重做日志。

9. 自适应哈希索引 AHI：

