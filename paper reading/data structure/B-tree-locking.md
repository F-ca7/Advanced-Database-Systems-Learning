[TOC]



## A Survey of B-Tree Locking Techniques

### Introduction

1. 索引的本质就是 key->information的映射
2. B树还支持 **范围查询**、**基于排序的查询执行算法**(如: [Sort-Merge Join](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/17%20parallel%20join/17-sort-merge-join.md))
3. 在B-tree的并发保护中，要区分 
     - **B-tree内容**的保护 与 **B-tree结构**的保护
     - **线程之间**的隔离 与 **用户事务之间**的隔离
4. 常规：
     - Shared locks(S，共享锁) 和 Exclusive locks(X，独占/排他锁)
     - 锁被 **线程** 或 **事务** 持有

### B-Tree Locking的两种形式

​	同一个事务中的两个线程，应该"看到"相同的数据库内容。即，一个线程不应该看到中间态或不完整的数据结构。

0. 两种形式：
   - 多事务查询/ 修改数据库内容以及B-tree索引  的并发控制
   - 多线程修改内存中的B-tree数据结构 (缓冲池中 基于磁盘的B-tree结点的 镜像) 的并发控制

1. 以上两种通常由 **Locks** & **Latches** 实现

   - lock 通过 数据页/B-tree键/B-tree键区间 上的 **读/写锁** 来分离事务

   - 注意：悲观锁和乐观锁是一种概念/思想，不能与数据库的锁机制(行锁、表锁、排他锁、共享锁)混为一谈。

     - 在DBMS中，悲观锁正是利用数据库本身提供的锁机制来实现的。每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁
     - 不是数据库本身的锁(用户自己实现的一种锁机制)，是利用数据比较结果来当做抽象的锁。实现方法包括 **版本号**、**时间戳**

   - 行锁 vs 表锁:

     |            | 表锁       | 行锁       |
     | ---------- | ---------- | ---------- |
     | 开销       | 小，加锁块 | 大，加锁慢 |
     | 死锁       | 不会       | 会         |
     | 粒度       | 大         | 小         |
     | 锁冲突概率 | 高         | 低         |
     | 并发度     | 低         | 高         |

   - latch 负责分离 访问缓冲池B-tree页、缓冲池管理表、所有内存中数据结构 的多线程。比如，在 一个线程修改lock manager的哈希表时，需要获取 write latch (即使是同一个事务中的多线程也会发生冲突)

   - ![](https://s2.ax1x.com/2019/09/17/n5CgzD.png)

2. 数据库恢复：
   - 在等待**全局事务协调器(global [transaction coordinator](https://stackoverflow.com/questions/1253836/whats-the-difference-between-a-transaction-coordinator-and-a-transaction-manage))**——(a server for managing distributed transactions)做出决定期间，不需要latch；但是需要持有locks来保证本地事务(local txn)会服从全局协调的最终决定。
   - 在**事务回滚**时，必须持有locks，如: 向一个事务保存点(savepoint, 不必重新提交所有语句) 部分回滚时(回滚完成后，释放**回滚语句使用的锁**；即释放次保存点后获得的表级锁和行级锁，但之前的数据所依然保留)。
   - 由latch(而不是lock)保护的操作是: 改变B-tree的结构，如 B-tree的结点分裂，均衡邻居结点。

3. 保护B-tree物理结构：
   - 在buffer pool的一个数据页镜像，当在被一个线程读取时，禁止另一个线程向它写(类似于 临界区的保护)
   - 在跟随B-tree索引从父结点到子结点时，另一个线程不能使当前的pointer(一个页标识符, page identifier)失效，如回收(de-allocate)子结点的页空间。由于在进行I/O (将磁盘中子结点的数据页读入buffer pool中)时，不建议持有latch (否则 临界区会变得很长)
   - 