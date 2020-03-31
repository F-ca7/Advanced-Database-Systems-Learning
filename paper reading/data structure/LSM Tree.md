## LSM-Tree (Log-Structured Merge Tree)

1. 实际中应用：
   - **NoSQL**
     - BigTable
     - HBase
     - Cassandra
     - MongoDB
   - **Storage Engine**
     - LevelDB
     - RocksDB
   - NewSQL
     - TiKV
     - CockroachDB

2. 原本的LSM-Tree: 

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200331095848.png)

   - Out-of-place update
   - Optimized for write: 单纯的append，很快
   - Sacrifice read: 一次读取可能有多次I/O，直到找到为止
   - Not optimized for space: 有很多过期数据，浪费一定空间（在merge阶段中回收）
   - Require data reorganization: merge / compaction

3. 现代结构（Level structure）: 

   ![](https://upload-images.jianshu.io/upload_images/14534869-73e713865daea2ab?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

   - **Read**:

     - **Point query**

       有Read amplification问题，最差情况的I/O：2*(N-1+L0的文件数)。

       优化：

       - Page cache / Block cache

       - Bloom filter (通过 filter 来减少不必要的 I/O，**不支持范围查询**)

         Level的每个文件有一个布隆过滤器，来确定该文件有没有这个key，<u>有可能有的情况才会去读</u>。

     - **Range query**

       

       优化：

       - Parallel Seeks
       - Prefix bloom filter (RocksDB): like 'xx%'
       - SuRF (Sufficient Range Filter): Level-Ordered Unary Degree Sequence 的Trie树，节省内存

   - **Write**: 

   - **Compaction**: 基本都是在**读放大**、**写放大**和**空间放大**这三者间做 trade-off

     > - 写放大
     >   Write Amplification : 写放大，假设每秒写入10MB的数据，但观察到硬盘的写入是30MB/s，那么写放大就是3。写分为立即写和延迟写，比如redo log是立即写，传统基于B-Tree数据库刷脏页和LSM Compaction是延迟写。redo log使用direct IO写时至少以512字节对齐，假如log记录为100字节，磁盘需要写入512字节，写放大为5。
     > - 读放大
     >   Read Amplification : 读放大，对应于一个简单query需要读取硬盘的次数。比如一个简单query读取了5个页面，发生了5次IO，那么读放大就是 5。假如B-Tree的非叶子节点都缓存在内存中，point read-amp 为1，一次磁盘读取就可以获取到Leaf Block；short range read-amp 为1~2，1~2次磁盘读取可以获取到所需的Leaf Block。
     > - 空间放大
     >   Space Amplification : 空间放大，假设我需要存储10MB数据，但实际硬盘占用了30MB，那么空间放大就是3。有比较多的因素会影响空间放大，比如在Compaction过程中需要临时存储空间，空间碎片，Block中有效数据的比例小，旧版本数据未及时删除等等。

     - **Leveled**

       向L<sub>n</sub>层的压缩 将L<sub>n-1</sub>层的数据 归并到L<sub>n</sub>层，会重写先前 合并到L<sub>n</sub>的数据。

       最小化空间放大 和 读放大，

     - **Tired**

       最小化写放大，但牺牲 读放大 和 空间放大

   按照箭头的序号，插入一条kv记录时，kv先存入Log中；然后该条kv再插入到**有序的***MemTable*（保存了最近的更新）中。当日志文件达到一定大小的，对应的*MemTable*就会被转化为一个只读的*ImmutableMemTable*。接着创建新的日志文件和*MemTable*，同时后台会把 *ImmutableMemTable*保存为在磁盘上有序的*SSTable*。（ Ln-1 is the Delta of Ln）

4. **Pipelined Compaction**

   具体步骤：

   1. Read data blocks
   2. Checksum
   3. Decompress
   4. Merge sort
   5. Compress
   6. Re-checksum
   7. Write to disk

5. **PebblesDB**

   通过 **Guard**将key空间划分为多个不相交的单元，减小了写放大。

6. **WiscKey**

   *Separating Keys from Values in SSD - conscious storage*

   Compaction = 排序 + GC （只有Key是需要有序的，**通过Key-Value的分离，使得排序和垃圾回收分开进行**）

   KV分离的新问题：

   - 每次查询可能需要额外的一次I/O：WiscKey的LSM Tree较小，可直接存入内存
   - 范围查询需要随机I/O：利用SSD的并行I/O特性，进行prefetch
   - 将value日志和WAL结合
   - crash后的一致性

   **垃圾回收**

   LevelDB等之类LSM-tree的存储系统对于对象的删除只是追加删除标记，延期至sstable compaction的时候回收那些无效数据。

   > https://zhuanlan.zhihu.com/p/38810568
   >
   > log区有两个指针 head和tail。head指向当前新数据待写入的位置，而tail则指向当前待回收的位置。
   >
   > 当触发垃圾回收时，从tail位置读取一条记录，并从元数据判断其是否已经被删除：
   >
   > - 如果是，则tail向前移动至下一条记录
   > - 如果不是，则将该记录写入至head位置，接下来再将tail移动至下一条记录

   





