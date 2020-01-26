## Logging

1. Recovery => 在failure下保证了

   - **数据库的一致性**consistency：数据库从一个一致性状态变到另一个一致性状态。

     注意以下几个一致性的含义区别

     1. 多副本的一致性：分布式系统中，在多副本的情况下，如何保证各个副本间数据的一致性。

     2. 一致性hash：普通的负载均衡的hash算法 是对服务器的数量进行取模，但这样当缓存服务器数量发生变化时，几乎所有缓存的位置都会发生改变，怎样才能尽量减少受影响的缓存？

       使用一致性哈希算法时，服务器的数量如果发生改变，并不是所有缓存都会失效，而是只有部分缓存会失效，前端的缓存仍然能分担整个系统的压力，而不至于所有压力都在同一时间集中到后端服务器上。

     3. CAP理论的一致性：在分布式系统中的所有数据备份，在同一时刻是否同样的值。

     4. ACID里的一致性：a transaction can only bring the database from one valid state to another, maintaining database invariants: any data written to the database must be valid according to all **defined rules**, including *constraints, cascades, triggers*

   - **事务的原子性**atomicity：Series of database operations in an atomic transaction will either all occur (a successful operation), or none will occur (an unsuccessful operation). 要么完成，要么都不完成

   - **事务的持久性**durability：对数据的修改就是永久的，即便系统故障也不会丢失。

2. Recovery算法分为两部分

   - 在事务处理过程中，保证DBMS能从宕机后恢复的措施。
   - 在宕机后，使数据库恢复到ACD状态的措施。

3. 日志分为两种：

   - **物理日志**：记录数据库中某条record的变化（如 存储 属性变化前后的值）——支持所有并发控制方案
   - **逻辑日志**：记录事务执行的高层操作（如 UPDATE, DELETE, INSERT）——更快但不通用

   -----

   逻辑日志 向每条日志record写的数据 比物理日志少。但是当有并发事务时，通过逻辑日记来恢复很难，因为 1. 难以确定宕机前 数据库哪一部分被修改；2. 耗时长，因为需要(根据逻辑日志)重新执行每一个事务

4. **面向磁盘的日志&恢复**disk oriented：

   标准——Algorithms for Recovery and Isolation Exploiting Semantic 基于语义的恢复与隔离算法，[ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks](https://blog.acolyer.org/2016/01/08/aries/)

   - **Write-Ahead Logging** （WAL 预写日志）：在数据写入到(磁盘上的)数据库之前，先写入到在稳定存储介质上的日志(即非内存)。每条日志记录(log record)对应一个唯一的标志(Logging Sequence Number，LSN 日志序列号)。DBMS通过 存储在内存(如缓冲池)和外存的 LSN 决定 系统中几乎所有东西的 **顺序**。
   - Repeating History During **Redo**：重启时，把数据库恢复到崩溃前的一个状态。
   - Logging Changes During **Undo**：在日志中记录undo撤销动作

5. **恢复三阶段**：
   1. **Analysis**：读取WAL来确定 停机时 缓冲池中的脏页 和 进行中的事务。
   2. **Redo**：从日志中一个合适的起点开始，重复所有的动作。同时把redo的每一步记录到日志中，以防恢复时再次崩溃。
   3. **Undo**：回滚在停机前 未提交事务的动作。

6. 事务中最慢的一步 通常是 等待DBMS把log record刷到磁盘上。要么是一个timeout到了，要么是缓冲池满了，就进行一次flush。可参考 [InnoDB](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/innoDb.md)的笔记。

7. [内存数据库](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/02%20in-memory%20db/2-imdb-notes.md)的recovery相对简单，因为不用考虑脏页问题，不需要存储undo记录。VoltDB是一种面向内存的DBMS，不包含传统的面向磁盘的DBMS组件，如缓冲池等。

8. Non-volatile memory(NVM，非易失存储器)，读写性能接近DRAM、掉电可以保存数据。但现在DBMS架构在设计之初都是建立在易失性内存上的，也就是基于由易失性DRAM和非易失性HDD/[SSD](https://zhuanlan.zhihu.com/p/43362595)组成的双层存储架构。

   |         | SSD         | HDD |
   | ------------- |-------------| -----|
   | 容量 |较小 | 大 |
   |  价格  | 贵      | 便宜                             |
   |  随机存取  | 极快  | 一般 |
   | 写入次数 | 有限制（每个NAND Block都有擦写次数的限制，当超过这个次数时，该Block可能就不能用了） | 无限制（机械硬盘的寿命按小时算） |
   | 磁盘阵列 | 可以 | 很难 |
   | 数据恢复 | 难 | 简单 |
   
   