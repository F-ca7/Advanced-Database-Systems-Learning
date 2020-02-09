## DATABASE INDEX

1. 两类索引：

   - 保序索引 Order Preserving：

     有序的树形结构，任何谓词的搜索 都在 O(log n)的复杂度内

   - hash索引：

     通过哈希映射 key -> record，只支持 O(1)复杂度的 等于搜索

2. B-tree VS B+tree
   - 共同优点：**磁盘IO**相对于内存来说是很慢的。数据库索引是存储在磁盘上的，当数据量大时，就不能把整个索引全部加载到内存了，只能逐一加载每一个磁盘页（对应索引树的结点）。故B树/B+树的**阶数 取决于 磁盘页的大小**。
   - B树每个结点都存储数据，因此经常访问的元素可能离根结点更近，因此访问也更迅速；B+树只有叶子结点存储数据。
   - 所有的叶子结点使用链表相连，**便于区间查找和遍历**。B树则需要进行每一层的递归遍历。相邻的元素可能在内存中不相邻，所以缓存命中性没有B+树好。

3. 索引上锁 和 数据库对象上锁 不一样，因为只要逻辑内容是一致的，物理结构可以随便变。

4. [Lock VS Latch](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/B-tree-locking.md)

5. Latch 实现：

   - 阻塞式的OS 互斥锁：

     简单，Non-scalable

     **std::mutex**

   - Test-and-Set 自旋锁：

     高效，Non-scalable, not cache friendly

     **std::atomic\<T>**

   - 基于队列的 自旋锁

     比mutex高效，局部性缓存更友好

     **std::atomic<Latch*>**

   - 读者-写者 锁

     支持并发读者，需要管理 读写者队列

     基于自旋锁实现

6. Crabbing——保证事务不会再修改数据时 破坏了内部的数据结构

   一个线程 只有在一个父结点的子结点是<u>safe</u>时，才能释放该父结点的latch

   即 在插入时不会发生split，删除时不会发生merge。

   **查询**：从根结点向下，反复地

   - 获取子结点的 读闩锁 read latch

   - 释放父结点上的latch

   **插入/删除**：从根结点向下，获取需要的 写闩锁 write latch。一个子结点被上锁后，检查是否安全

   - 如果子结点安全，则释放祖先的所有锁

7. Lock：保护索引的逻辑内容，避免其他事务的幻读。

   <u>与Latch的不同</u>：

   - lock在整个事务期间持有
   - lock只在叶子结点处获取
   - 不会存储在索引的树结构中

8. Lock的方案：

   - **谓词锁 Predicate Locks**: 

     对于**SELECT**语句的**WHERE**使用**共享锁** Shared lock；对于**UPDATE**/ **INSERT**/ **DELETE**的WHERE使用**独占锁** Exclusive lock。

   - **键值锁 Key-Value Locks**: 

     只覆盖一个键的锁，对于不存在的值需要用 虚拟键。

   - **间隔锁 Gap Locks**: 

     解决了**KV锁**需要虚拟键的填充。

   - **键范围锁 Key-RangeLocks**: 

     **键值锁**和**间隔锁**的组合。范围是 从一个key，一直到下一个key。

   - **层次锁 Hierarchical Locks**: 

     更大的键范围，支持不同的锁模式。减少访问lock manager的次数。

9. B+树 是为磁盘的访问设计的。对于内存数据库，用什么数据结构？

   ——T树。

> Based on AVL Trees. Instead of storing keys in nodes, store pointers to their original values.

10. T树优点：

    - 内存占用更少，因为每个结点不会保存key
    - 在访问元组的同时，可以进行数据表上谓词的运算

    T树缺点：

    - 平衡算法难
    - 难以实现 安全地并发访问
    - （降低缓存的局部性）Must chase pointers when scanning range or 
      performing binary search inside of a node.

11. T树设计：

    ![T树结点设计](https://s2.ax1x.com/2020/02/08/1RFO00.png)

    ![](https://s2.ax1x.com/2020/02/08/1RAi8g.md.png)

12. BW树：无闩锁的B+树索引(来自微软的Hekaton)

    - **Delta**: 减少缓存失效
    - **Mapping Table**: 允许页面物理地址的CAS

    1. **Delta Update**: page的更新 会生成一个新的delta，delta 物理上地指向一个基页面base page。通过CAS把delta的地址 装入mapping table中。
    2. **Search**: 像B树一样遍历，如果mapping table指向delta链，在第一次搜索key的第一次出现就停下，或在基页面进行二分查找。
    3. **Contention Update**: 不同的线程可能同时更新一个页面，会发生竞争，失败的会重试或终止。
    4. **Consolidation**: 通过创建新页面来合并更新，再利用CAS将mapping table的指向地址改为新页面，老的页面就成为垃圾。

13. GC: 

    判断回收内存是否安全：

    - **Reference Counting**

      每个结点维护一个计数器，记录访问该结点的线程数。在访问前增加计数器，结束时减少；只有计数器为0，回收该结点才是安全的。

      对于多核CPU来说性能会很差，因为增减计数器 会导致***cache coherence traffic***，即 高速缓存间一致性问题。

    > 在一个系统中，当许多不同的设备共享一个共同存储器资源，在高速缓存中的数据不一致，就会产生问题。这个问题在有多个CPU的多处理机系统中特别容易出现。

    - **Epoch-based Reclamation**

      维护一个全局的 ***epoch counter***，每个周期(10ms)更新一次。

      每个操作都有 epoch 标签，每个 epoch 记录属于它的线程 以及 可回收的对象。
    
      在Linux中，称为 ***Read-Copy-Update***
    
    > A synchronization mechanism implementing a kind of mutual exclusion which can sometimes be used as an alternative to a readers-writer lock. It allows extremely low overhead, wait-free reads. However, RCU updates can be expensive, as they must leave the old versions of the data structure in place to accommodate pre-existing readers. These old versions are reclaimed after all pre-existing readers finish their accesses.
    >
    > 被RCU保护的共享数据结构，读操作不需要获得任何锁就可以访问，但写操作在访问它时首先拷贝一个副本，然后对副本进行修改，最后在适当的时机把指向原来数据的指针重新指向新的被修改的数据。

    
    
    - **Hazard Pointers**

14. Trie树：又叫前缀树

    - 不需要平衡操作
    - 不依赖于插入的次序
    - 所有操作的复杂度为O(k)，k是key的长度
    - 一个 level 的跨度 span 表示了一个 单个字母代表的bits

    ![](https://s2.ax1x.com/2020/02/08/1W6nVe.md.png)

15. Trie树变种：

    - [Judy](https://sourceforge.net/projects/judy/) Arrays

      三种array类型：

      - Judy1: Bit array that maps integer keys to true/false.

        ```c
        // JUDY1 BITMAP LEAF (J1LB) SUPPORT
        
        #define J1_JLB_BITMAP(Pjlb,Subexp)  ((Pjlb)->j1lb_Bitmap[Subexp])
        
        typedef struct J__UDY1_BITMAP_LEAF
        {
                BITMAPL_t j1lb_Bitmap[cJU_NUMSUBEXPL];
        
        } j1lb_t, * Pj1lb_t;
        
        // BITMAPL_t是uint型
        ```

      - JudyL: Map integer keys to integer values.

        ```c
        // Assemble bitmap leaves out of smaller units that put bitmap subexpanses
        // close to their associated pointers.  Why not just use a bitmap followed by a
        // series of pointers?  (See 4.27.)  Turns out this wastes a cache fill on
        // systems with smaller cache lines than the assumed value cJU_WORDSPERCL.
        
        #define JL_JLB_BITMAP(Pjlb, Subexp)  ((Pjlb)->jLlb_jLlbs[Subexp].jLlbs_Bitmap)
        #define JL_JLB_PVALUE(Pjlb, Subexp)  ((Pjlb)->jLlb_jLlbs[Subexp].jLlbs_PValue)
        
        typedef struct J__UDYL_LEAF_BITMAP_SUBEXPANSE
        {
                BITMAPL_t jLlbs_Bitmap;
                Pjv_t     jLlbs_PValue;
        
        } jLlbs_t;
        
        typedef struct J__UDYL_LEAF_BITMAP
        {
                jLlbs_t jLlb_jLlbs[cJU_NUMSUBEXPL];
        
        } jLlb_t, * PjLlb_t;
        ```

      - JudySL: Map variable-length keys to integer values.

      是256-way radix tree的变种，结点可以根据它的keys来自适应调整自己的类型：

      - **Linear Node**: key稀少的情况
      - **Bitmap Node**: key中等的情况
      - **Uncompressed Node**: key稠密的情况

    - ART Index

      256-way radix tree，根据密度支持不同的结点类型（在结点头部存储元信息）。共四种结点类型，根据孩子结点的数目调整。

      ART是一个表索引，Value指向元组。

      使用二进制的Comparable Key：

      - Unsigned Integers: Byte order must be flipped for little 
        endian machines.
      - Signed Integers: Flip two’s-complement so that negative 
        numbers are smaller than positive.
      - Floats : Classify into group (neg vs. pos, normalized vs. 
        denormalized), then store as unsigned integer.
      - Compound : Transform each attribute separately.

    - Masstree

      每个结点使用一棵B+树（8字节的跨度），适合长整型的key，使用类似带版本号的闩锁协议。
      
      