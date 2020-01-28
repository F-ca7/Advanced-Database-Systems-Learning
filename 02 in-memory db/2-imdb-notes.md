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

2. In-memory database: 又叫Main-memory database

   - 不需要将数据库存为slotted pages，但是仍然会把元组放在block/page中
   - 为什么**不用mmap**(把数据库文件映射到内存中，让OS来负责数据在需要时的换入换出)—— 因为mmap无法在精细粒度上控制内存中的内容:
     1. 无法非阻塞地访问内存
     2. 在磁盘上的表示形式要和在内存中的相同
     3. DBMS无法知道一个page是否在内存中
     4. 与mmap相关的系统调用无法移植
	- 仍需要**WAL**(预写日志系统，提高IO效率——在数据写入到数据库之前，先写入到日志，在将日志记录变更到存储器中)
   - 仍需要设立检查点**checkpoint**来加快恢复

3. 对比：

   |  | 内存数据库 | Right-aligned |
   | :----------- | :------------ | :------------|
   | 数据存储   | 行、列存储，段-分区式存储模型，不要求数据在内存中连续        | 行、列存储，在磁盘上连续存放 |
   | 缓冲管理   | 无                                                           |      有 |
   | 并发控制 | 较大粒度的锁，如库级、表级；或乐观锁机制 | 为了提高事务的并发度，一般支持多粒度和多类型的锁 |
   | 恢复机制 | 备份、日志、检查点技术；预提交、组提交等提交方式；用稳定内存来存储log record | 备份、日志、检查点、保存点 |
   | 索引结构 | T树、hash | B树、hash |
   | 查询优化 | 基于处理器代价、cache代价 | 基于I/O代价 |

4. 各种树的对比：具体参考

   - [T树](http://www.memdb.com/paper.pdf)：由AVL树（自平衡二叉查找树）和B树发展而来，既有AVL树的二分查找特性，又有B树良好的更新和存储特性。

   > T树索引用来实现关键字的范围查询。T树是一棵特殊平衡的二叉树（AVL），它的每个节点存储了按键值排序的一组关键字。T树除了较高的节点空间占有率，遍历一棵树的查找算法在复杂程度和执行时间上也占有优势。
   >
   > 参考：https://blog.csdn.net/u013815649/article/details/51940548

   - [R树](http://www-db.deis.unibo.it/courses/SI-LS/papers/Gut84.pdf)：是一种空间索引数据结构，用于处理多维数据的数据结构。

     空间数据对象通常会覆盖多维空间的区域，不便于使用点坐标来表示。空间数据中的一个常用操作是 搜索一个区域内的 所有对象。

     而经典的一维数据库索引 不适合多维的空间搜索。1. 基于精确值匹配的数据结构，比如hash表，在范围查询不适用。2. 基于一维顺序的数据结构，比如B树、ISAM索引，不能适用 多维的搜索空间。

   > 多维索引技术的历史可以追溯到20世纪70年代中期。就在那个时候，诸如Cell算法、四叉树和k-d树等各种索引技术纷纷问世，但它们的效果都不尽人意。在GIS和CAD系统对空间索引技术的需求推动下，Guttman于1984年提出了R树索引结构，发表了《R树:一种空间查询的动态索引结构》，它是一种高度平衡的树，由中间节点和页节点组成，实际数据对象的最小外接矩形存储在页节点中，中间节点通过聚集其低层节点的外接矩形形成，包含所有这些外接矩形。其后，人们在此基础上针对不同空间运算提出了不同改进，才形成了一个繁荣的索引树族，是目前流行的空间索引。。
   > 原文链接：https://blog.csdn.net/windgs_yf/article/details/86534384

   - [X树](https://kops.uni-konstanz.de/bitstream/handle/123456789/5734/The_X_Tree.pdf)：线性数组和层状的R树的杂合体。

5. Cache性能优化：cache失效时CPU需要等待从内存中读取数据。

   ​	基本思想：**对数据划分，使得划分结果能够放入cache，以提高cache命中率。**

   - [radix-cluster算法](http://www.vldb.org/conf/1999/P5.pdf)：通过多路运算将一个关系划分为多个cluster，最后将两个join的关系划分为 小于cache大小的cluster。
   - 处理projection：(1)pre-projection，先投影，再连接；(2)post-projection，先连接，再投影。(3)先生成join-index——[oid, oid]，然后再计算投影列，生成查询结果。如果关系中的元组太多，每个列都不能放入cache内，则进行projection时，随机查找会引起很多次cache失效。