## Multi-Version Concurrency Control

1. COMPARE-AND-SWAP（CAS）:使用了3个基本操作数——内存地址V，旧的预期值A，要修改的新值B。更新一个变量的时候，只有当变量的预期值A和内存地址V当中的实际值相同时，才会将内存地址V对应的值修改为B。

   在Java中：synchronized就是悲观锁，CAS操作就是乐观锁。

   CAS缺点：

   - ABA问题：如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。

     ——解决思路：使用**版本号**version

   - 循环时间长开销大：自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。

     ——解决思路：jvm支持处理器的**pause指令**。pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率

   - 只能保证一个共享变量的原子操作：对多个共享变量操作时，循环CAS就无法保证操作的原子性

     ——解决思路：**锁synchronized** 或 把多个共享变量合并成一个共享变量来操作，如java的AtomicReference类可以把多个变量放在一个对象里来进行CAS操作

2. 隔离级别 Isolation Level：

   ![](https://s2.ax1x.com/2020/02/01/1Gd1nf.md.png)

   主流数据库考虑到串行化效果与并发性能的平衡，一般默认隔离级别都介于RC与RR之间

   - Read Uncommitted -> Read Committed -> Repeatable Reads -> Serializable

   - **Cursor Stability** (DB2的默认隔离级别): 强度介于Read Committed和Repeatable Reads 之间，DBMS的cursor会在当前记录上维护一个锁，直到移向下一条记录。有些情况下，可以防止丢失更新异常 lost update anomaly。

     **丢失更新异常**：txn2 先写A，txn1后写A；但txn1先提交，txn2后提交；导致txn2的写A操作丢失。

   - **Snapshot Isolation** (Oracle的最高隔离级别 **快照隔离** SI): 保证同一个事务中的读操作 会读到一个一致的数据库快照(在事务开始时的)。容易遇到写偏序异常Write Skew Anomaly

     写偏序"黑白球"问题——两个并行事务都基于自己读到的数据集去覆盖另一部分数据集，在串行化情况下两个事务无论何种先后顺序，最终将达到一致状态，但SI隔离级别下无法实现。

     ![](https://s2.ax1x.com/2020/02/01/1GBE7T.md.png)

3. MVCC: DBMS为数据库中的一个逻辑对象 维护多个物理版本

   - 当一个事务写入一个对象时，DBMS会创建 该对象的一个新版本
   - 当一个事务读取一个对象时，会读取 该事务开始时 此对象的最新版本

   优点：

	- 写者 不会阻塞 读者
	- 只读事务 在不需要锁的情况下 就可以读到 一致的数据库快照

4. 元组格式的设计

   | 属性     | 含义                                                         |
   | -------- | ------------------------------------------------------------ |
   | Txn_Id   | 唯一的事务id                                                 |
   | Begin_Ts | 开始时间戳                                                   |
   | End_Ts   | 结束时间戳                                                   |
   | Pointer  | 指向下一个版本create a latch-free version chain per logical tuple |
   | …        | 其他metadata                                                 |
   | Data     | 具体数据                                                     |

5. **时间戳顺序**：(MVTO Multi-Version Timestamp Ordering)

   元组新增一个Read_Ts属性：Use “read-ts” field in the header to keep track of the timestamp of the last txn that read it.
   
   在元组没有锁的情况，事务只能读取 其id在Begin_Ts和End_Ts之间的版本，同时更新该元组的Read_Ts。

6. **Pointer**：通过pointer 为每一个 逻辑元组 创建一个 不需要闩锁latch-free 的**<u>版本链version chain</u>**

   - DBMS可以通过版本链，在运行期间，找到对一个事务可见的version
   - 索引 总是指向 版本链的头结点

   版本链有两种顺序，在性能上有不同的trade-off：

   - Oldest-to-Newest (O2N): 
   - Newest-to-Oldest (N2O):

7. **版本存储** Version Storage：

   - Append-Only: 新版本 会直接附加在 同一个表空间

     | Main Table    | KEY  | VALUE | POINTER       |
     | ------------- | ---- | ----- | ------------- |
     | A<sub>1</sub> | X    | 100   | A<sub>2</sub> |
     | A<sub>2</sub> | X    | 200   | A<sub>3</sub> |
     | B<sub>1</sub> | Y    | 0     | nil           |
     | A<sub>3</sub> | X    | 300   | nil           |

     同一个(逻辑)元组每次更新，会在表中插入一个新的(物理)版本。老的指向新的。

   - Time-Travel: 老版本会复制到 一个独立的表空间Time-Travel Table

     | Main Table    | KEY  | VALUE | POINTER                      |
     | ------------- | ---- | ----- | ---------------------------- |
     | A<sub>3</sub> | X    | 300   | A<sub>2</sub> in Time-Travel |
     | B<sub>1</sub> | Y    | 0     | nil                          |

     | Time-Travel Table | KEY  | VALUE | POINTER       |
     | ----------------- | ---- | ----- | ------------- |
     | A<sub>1</sub>     | X    | 100   | nil           |
     | A<sub>2</sub>     | X    | 200   | A<sub>1</sub> |

     新的指向老的。

   - Delta: 被修改属性的原始值会拷贝到 一个独立的delta record space

     | Main Table    | KEY  | VALUE | POINTER                |
     | ------------- | ---- | ----- | ---------------------- |
     | A<sub>3</sub> | X    | 300   | A<sub>2</sub> in Delta |
     | B<sub>1</sub> | Y    | 0     | nil                    |

     | Delta Storage | DELTA        | POINTER       |
     | ------------- | ------------ | ------------- |
     | A<sub>1</sub> | (VALUE->100) | 100           |
     | A<sub>2</sub> | (VALUE->200) | A<sub>1</sub> |

8. **垃圾回收** Garbage Collection: 

   DBMS需要 随着时间推移 删除**可以回收的**物理版本。

   - 没有 active的事务 可以看到该版本——快照隔离
   - 该版本由一个 已结束的事务创建

9. 两种层面的GC: 

   - 元组级 Tuple-level：

     1. Background Vacuuming: 独立的线程 会周期性地扫描表，寻找可回收的版本。
     2. Cooperative Cleaning: 工作线程在 遍历版本链 时，找到可回收的版本。只适用于**O2N**。

   - 事务级 Transaction-level：

     每个事务会记录 自己的 读/写集合。由DBMS来决定 一个已完成的事务 创建的所有版本 什么时候不可见。

     注意 时间戳到达最大值的问题。

   > If the DBMS reaches the max value for its timestamps, it will have to wrap around and start at zero. This will make all previous versions be in the "future" from new transactions.
	
	​	Postgres的解决方法：当系统即将达到Txn_id的最大值时，停止接收新命令。在每个元组头部设置flag，标志其为过去的状态。而任何新的Txn_id都比过去状态的元组要新。

10. **索引管理** Index Management：

    - 主键索引——总是指向版本链的头结点。DBMS更新 主键索引的频率，取决于 系统是否 在一个元组更新时 就创建新版本。当一个事务更新元组的主键属性时，会等同于 先执行一次**DELETE**，再执行一次**INSERT**。

    - 辅助索引——较为复杂。可以使用 **Logical Pointers**（每个元组有一个固定不变的标识符）或 **Physical Pointers**

-----

### 具体实现

11. SAP HANA—— In-memory **HTAP** DBMS with time-travel version storage (N2O) 

> 一种内存数据平台，可以作为内部预置型设备部署，也可以部署在云端。这是一个革命性的平台，最适合执行实时分析，开发和部署实时应用程序。
>
 - 支持 乐观 以及 悲观的 MVCC

 - Time-travel表中存储最新的版本

 - 使用[混合存储layout](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/Flexible-Storage-Model.md)（行+列式）

   **版本存储**：
   
- 在main table中存储最老的版本

- 每一个元组维护一个 flag 来标记是否有newer version

- 通过一个 hash 表，key为**record的identifier**，value为**版本链的头结点**。

12. 









