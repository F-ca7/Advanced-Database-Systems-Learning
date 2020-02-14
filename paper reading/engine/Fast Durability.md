## Fast Databases with Fast Durability and Recovery Through Multicore Parallelism

1. **问题**：现代的高速内存型数据库可以支持OLTP的 高事务处理率，但是宕机后的recovery需要很长的时间。

2. **Contribution**: 不使用replication，在更短的时间内恢复一个大型数据库（内存型数据库）。

3. 多核内存数据库——可以支持OLTP的高事务率，但宕机后的恢复比较麻烦。普通的**logging**和**checkpoint**技术 会让 普通情况的执行 变慢，但频繁的磁盘同步 使得在只降低可接受的吞吐量 的情况下，能支持高负荷量。

4. **Replication** can allow one site to step in for another, but even replicated databases must write data to persistent storage to survive correlated failures, and performance matters for both persistence and recovery.

   要注意区别Replication 和 Duplication：

   In database technology, replication is a **repeated activity** where as duplication is a **one time event**. I want to replicate TableA on Server1 to Server2 TableA continuously. so if server1.tableA changes values so does Server2.TableA. Duplication means I create Server1.TableA once on Server2.TableA. And as Server1.TableA changes, Server2.TableA does not and get out of sync. So I guess it would be dependent on the context in which the words are used. Replication is repetitive/scheduled event, duplication is a one time

5. **Logging**:  在Silo的基础上，添加了 Log truncation，并使得replay的并行度更高。

   - 由**workers** 和 **loggers**一起负责

     **loggers**即单独的logging线程，专门负责logging, checkpointing 和 housekeeping

     **workers**在事务提交时 生成log record，再把这些record传给**loggers**；**loggers**就把这些日志持久化到磁盘，当通过*fsync*成功提交到磁盘后，**loggers**会通知**workers**，从而**workers**可以把事务执行的结果告诉客户端。

     将**worker**划分为不相交的子集，每个worker子集分配一个**logger**，通过**core pinning**保证logger与对应的workers在同一个CPU上运行，使得在一个socket上的log buffer 只会在该socket中被访问。

   > Processor affinity, or CPU ***pinning*** or "cache affinity", enables the binding and unbinding of a process or a thread to a central processing unit (CPU) or a range of CPUs, so that the process or thread will execute only on the designated CPU or CPUs rather than any CPU, can improve the performance of your code by increasing the percentage of local memory accesses.
   >
   > The **Socket** is a hardware on the motherboard which accepts the CPU for its mounting on the motherboard. The Socket thereby connects the CPU with the motherboard circuitry. One socket accepts exactly one CPU.

   - **Value** logging vs. **Operation** logging

     使用**Value logging**，即日志包含 每个事务输出的 key与value（而不是执行的操作和参数）。其缺点是 会记录更多的日志数据，事务执行较慢；但恢复时的并行性很好，最终只用保留 **TID**最大的值（**TID**对应了写的次序）。

     通过添加硬件 使得**Value logging**的I/O不是瓶颈。

   - **Buffer Management**: 

     当log buffer满了 或者 一个新的epoch开始时，worker就把buffer给flush到其对应的logger。一个日志文件即 把不同worker传来的buffer该连接起来（不需要按特定顺序）。

6. **Checkpoint**: 检查点之间的距离越小，recovery需要replay的日志数据越少。

   - 内存数据库必须周期性地保存检查点，才能让恢复的过程更快，并支持日志截取 log truncation（防止log文件被填满）。

   - 每个checkpoint disk有一个checkpointer线程，有checkpoint manager 为每个checkpointer分配数据库不同的slice.

   - Checkpointer会按key的顺序（根据索引树）遍历自己负责的slice，写下每条经过的record. 

   - 检查点不一定必须是 数据库在某一点的一致的快照。因此，需要在恢复检查点后，再replay一部分的日志。

   - **Cleanup**: 当检查点完成后，需要删除不需要的旧文件。

     Logger把日志存在同个文件夹下的一系列文件中。新的条目写入名为**data.log**的文件，隔一定周期(比如每100个epoch)，logger将该文件改名为**old_data.e**，*e*为该文件包含的最大epoch号。

     而所有**e**小于当前epoch号的文件 即可被删除。

   - Log replay的代价比checkpoint 的代价高，故通过增加checkpoint 的频率，来减少log replay

7. **Future Work**: 
   
   - 更弹性的 checkpoint方案，当log文件不会增长太快时，可以延缓checkpoint。

