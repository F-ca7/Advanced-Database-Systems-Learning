## The Adaptive Radix Tree:  ARTful Indexing for Main-Memory Databases

1. **问题**：对于内存数据库，索引结构的性能是关键瓶颈。传统的平衡二叉搜索树 在现代硬件上不够高效（包括T树，即使是cache友好的B+树 在更新操作也需要更高的代价），因为没有最好地利用CPU cache，而且由于比较的结果不能预测，会造成更多的流水线 stalls。哈希表够快，但是只支持单点查询。
2. **Contribution**: 提出 ART，自适应的 radix tree (即trie，就是压缩前缀树；但是trie更着重于字符串型的key，**这里使用radix是为了强调适用于多数据类型的基数排序原理**)。
   - Space efficient，自适应地为内部节点选择紧凑而有效的数据结构，解决了radix树常见的空间消耗问题
   - 数据有序，支持范围查询、前缀查询等
   - 描述了 常见的内置数据类型如何在 radix tree中**有序存储**
   - 将ART整合到 HyPer中，通过现实中的事务处理 证明了其优越的性能

3. **Related Work**: 

   - 在基于磁盘的数据库系统中，B+树最普遍
   - 对于内存数据库，提出了 红黑树、T树
   - 由于 类二叉搜索树的 利用cache性能很差，提出了 针对cache优化的B+树变种 (G. Graefe and P.-A. Larson, “B-tree indexes and CPU caches,” in ICDE, 2001.)
   - 使用 SIMD指令减少比较的次数，也减少了 cache miss
   - 提出 FAST(Fast Architecture Sensitive Tree) ，使用了SIMD, cache line,  page block 来最大化利用cache和内存带宽

   > 每个高速缓存行完全是在一个突发读操作周期中进行填充或者下载的。即使处理器只存取一个字节的存储器，高速缓存控制器也启动整个存取器访问周期并请求整个数据块。缓存行第一个字节的地址总是突发周期尺寸的倍数。缓存行的起始位置总是与突发周期的开头保持一致。
   >
   > 当从内存中取单元到cache中时，会一次取一个cache line大小的内存区域到cache中，然后存进相应的cache line中。

   - Burst trie 使用trie结点作为树的上层结构，当一个子树的元素很少时，就转化为链表；HAT-trie把链表替换为哈希表

4. **实现细节**：

   - ART的自适应结点 不会影响最终的树高度（因为 radix tree高度只有key的长度决定，而不是元素的个数）

   - 两种结点类型

     - **Inner Node**: 将partial key映射到其他结点

       由两个List组成，一个List存放key部分，一个存放指向下一个结点的指针部分。

       有Node4, Node16, Node48, Node256 四种大小的内部结点。

     - **Leaf Node**: 存储key对应的value

       如果是非unique的key，即一个key可对应多个value的情况，可在key后附加元组标识符。

   - **Binary-comparable keys**

     可以直接比较内存中的二进制表示形式

     1. 无符号整数：只需要考虑大小端即可
     2. 有符号整数：补码形式需要重新排序，对于b位的有符号整数x，可转化为 x XOR 2<sup>b-1</sup>
     3. IEEE 754 浮点数：根据定义重新计算即可
     4. 字符串：对于Unicode的比较规则很复杂
     5. Null: 赋予特定的次序
     6. 复合key: 分别对每个属性进行转换即可

5. **Future Work**: 支持并发更新；计划研发一个latch-free的同步方案，如使用CAS机制

