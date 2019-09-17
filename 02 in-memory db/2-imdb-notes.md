## In Memory Database

1. Disk-oriented DBMS(面向磁盘):

   - 主要存储位置是在HDD, SSD上，数据库分为固定大小的块——称为**slotted pages**

   - 使用在内存中的缓存池，缓存来自磁盘的块(表现为一个个page)

   - 当一个查询想访问一个page P (通过一个Page Table查询得到)时，DBMS先检查P是否在缓存池中：若不在，则DBMS从磁盘中取出页面P，并复制P放在缓冲池的一帧(frame)中 —> 若无空的frame，则按一定的替换原则evict一个在缓存池的page (此时应该还要更新page table，原PPT中没提到) —> 若该page是dirty(被修改过)，则要将对应修改后的内容写回磁盘；未修改过就不用管。 <!-- 以上和操作系统中页面调度的过程基本一样-->

   - 需要设置lock(锁)和latch(闩锁)来为txns(transactions)保证ACID，二者区别: (参考[B-tree Locking](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/B-tree-locking.md))

     | 作用 | Locks                             | Latches                          |
     | ---- | --------------------------------- | -------------------------------- |
     | 隔离 | 用户事务                          | 线程                             |
     | 保护 | 数据库内容                        | 内存中的数据结构                 |
     | 持续 | 整个事务                          | 临界区critical section           |
     | 死锁 | 检测(wait-for图、timeout) 与 解除 | 避免(编码时的规范，如加锁的顺序) |
     | 存在 | Lock Manager(哈希表)              | 受保护数据结构                   |

     - latch可以分为互斥锁和读写锁，主要是保证并发线程操作临界资源的正确性
     - lock对象一般用来锁定数据库的表、页、行，在事务commit或rollback后才释放

2. In-memory database: 

   - 不需要将数据库存为slotted pages，但是仍然会把元组放在block/page中
   - 为什么**不用mmap**(把数据库文件映射到内存中，让OS来负责数据在需要时的换入换出)—— 因为mmap无法在精细粒度上控制内存中的内容:
     1. 无法非阻塞地访问内存
     2. 在磁盘上的表示形式要和在内存中的相同
     3. DBMS无法知道一个page是否在内存中
     4. 与mmap相关的系统调用无法移植

   - 仍需要**WAL**(预写日志系统，提高IO效率——在数据写入到数据库之前，先写入到日志，在将日志记录变更到存储器中)
   - 仍需要设立检查点**checkpoint**来加快恢复

   