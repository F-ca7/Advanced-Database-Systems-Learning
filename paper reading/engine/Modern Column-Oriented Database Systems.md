## The Design and Implementation of Modern Column-Oriented Database Systems

1. 列式数据库在OLAP上更快的原因：

   - Columns read on demand: 减少了IO的量

   - Batch execution: 以一个chunk进行计算，对locality有优势

   - Compression: 比如int类型的压缩（部分高位可能都是0）

   - Vectorization: SIMD

   - Codegen: 在列式数据库上更简单

     > 在运行时根据逻辑动态的生成可执行代码，那么对于java平台来说，就是动态的生成类实现

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200330104838.png)

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200330104938.png)

   - Late materialization: 带有filter条件的query，可以先进行filter的运算，知道哪些行有效，再进一步取需要的字段被过滤出来的值。可参考 *Materialization Strategies in a Column-Oriented DBMS, 2007*

   - Block IO: IO块的单位可以增大

   - Rough index: 粗略的索引，定位到感兴趣的块（传统的index是 B树 / Bitmap）；

   - Inverted index

   - Join index

   - Database cracking: 数据分成很多块，可以选择某种行的排序方式（如 按照A列排序、按照B列排序，可以有多个排序的副本，对应不同的索引）；可以和AI结合起来，根据query自动调整排序方式。可参考[Database Cracking](https://stratos.seas.harvard.edu/files/IKM_CIDR07.pdf)

     > Index maintenance should be a byroduct of query processing, not of updates. Continuously reacting on query requests brings the powerful property of self-organization.  

   - FPGA, GPU: GPU加速的[DB](https://www.omnisci.com/blog)

2. 列式存储更新：

   - SQL Server

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200331033336.png)

     - Immutable storage中的数据未排序

       优点：直接append

       缺点：不能二分查找

     - Delta Store在内存中，使用B+树的结构

     - 更新的代价高

   - Vertica

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200331033746.png)

     - **C-store**

       *C-Store: A Column-oriented DBMS, 2005*

       一个模块WS负责处理快速写入，另一个模块RS负责提供高效的查询 ，通过Tuple Mover，将 WS 中的数据同步到 RS 中。

       ![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X3BuZy9CZ2JiYmVIWXJwNDJuR1FzckM4N1oyQXNqMWR4S0xmaWJkWnd0cTRtbVN3VjNUZWFtZkh6Q0o5bWljbmxmUjZLS2dBekFjR21OZlZhNlBndW5hUW0zUzN3LzY0MA?x-oss-process=image/format,png)

     - **Projection as Index**

       存数据会存多份，通过*Projection*快速得到结果

     - **Sorted WOS and ROS**

       写优化区、读优化区

     - **DV** - Delete vector

       删除通过Delete vector进行，不会更新树的结构

3. **Join**实现：

   最直接的Hash Join：

   Join的输入只包含 predicate涉及的列，所以 Hash表会更紧凑，Cache miss也会更小。而Join的输出为两个输入列匹配的 位置对，其中输出的左列会按顺序排列，右列则不会（因为通常使用左列来迭代遍历）。

   

   > 对比传统Oracle的hash join：
   >
   > 1.  首先Oracle会根据参数HASH_AREA_SIZE、DB_BLOCK_SIZE和HASH_MULTIBLOCK_IO_COUNT的值来决定Hash Partition的数量（**一个Hash Table是由多个Hash Partition所组成，而一个Hash Partition又是由多个Hash Bucket所组成**）
   > 2. 数据量较小的那个**表A** 会被Oracle选为哈希连接的驱动结果集
   > 3. 接着Oracle会遍历A，读取**A**中的每一条记录，并对S中的每一条记录按照Join的列 做哈希运算；会使用两个内置哈希函数，分别记算出hash_value_1和hash_value_2
   > 4. 然后Oracle会按照hash_value_1的值把相应的S中的对应记录存储在不同Hash Partition的不同Hash Bucket里，同时和该记录存储在一起的还有该记录用hash_func_2计算出来的hash_value_2的值。注意，存储在Hash Bucket里的记录并不是目标表的完整行记录，而是只需要存储位于目标SQL中的跟目标表相关的查询列和连接列就足够了；**把S所对应的每一个Hash Partition记为Ai**，内存放不下的那部分Hash Partition会放在磁盘上
   > 5. Oracle会遍历B，读取B中的每一条记录，并对B中的每一条记录按照连接列做哈希运算，这个哈希运算和步骤3中的哈希运算是一模一样的，即这个哈希运算还是会用步骤3中的hash_func_1和hash_func_2，并且也会计算出两个哈希值hash_value_1和hash_value_2；接着Oracle会按照该记录所对应的哈希值hash_value_1去**Ai**里找匹配的Hash Bucket；如果能找到匹配的Hash Bucket，则Oracle还会遍历该Hash Bucket中的每一条记录，并会校验存储于该Hash Bucket中的每一条记录的连接列，看是否是真的匹配

   **改进：** *Dimitris Tsirogiannis, Stavros Harizopoulos, Mehul A. Shah, Janet L. Wiener, and Goetz Graefe. Query processing techniques for solid state drives. In Proceedings of the ACM SIGMOD Conference on Management of Data, pages 59–72, 2009.* 使用Jive Join，通过额外增加两次join输出数据的排序（大部分数据库系统都有实现外部排序算法），来使得 所有列可以被依次迭代。

   进一步研究表明：不需要完整的排序。由于存储介质上划分为多个连续的block，单个block内的随机访问 比 跨block的随机访问 要快得多。故数据库不需要对 *position list*进行完整地排序，只需要将其划分到storage的block上，在每个划分内 position是无序的，从而用于 **提取值的列** 是按block的顺序进行访问，而不是严格按 position的顺序。

4. **Insert / Update / Delete**

   比起行式存储，列式存储 对更新操作 更敏感。若每一列单独存在一个文件中，则关系表中的一条元组会存储在多个文件中；即使只更新一行数据 也要进行多次I/O。

   CStore和MonetDB 将架构分解为 **read-store**来管理数据的主题，**write-store**来管理最近的更新（通常在内存中，更新周期性地传到**read-store**中）。如MonetDB 为每个base column附加两个辅助列，用于存储待定的insert和delete。这种存储delta方法的缺点是，query需要先访问read-store获取基表信息，再访问write-store合并更新（insert使用MergeUnion，delete使用MergeDiff，消耗CPU资源）。

   VectorWise系统使用新的数据结构—— **Positional Delta Trees**来存储delta。当一个query提交时，首先找到表的哪个位置受影响，把merge操作 从查询期 转移到 更新期，符合read-optimized的流程。**Positional Delta Trees**是一种计数型B树，可以在对数复杂度下 记录最新的位置。delta可以利用计算机的层次架构进行分层设计。可参考 *Positional update handling in column stores. In Proceedings of the ACM SIGMOD Conference on Management of Data, pages 543–554, 2010.*

   而Hyper系统 同时对OLTP和OLAP上有较好的支持，主要依赖 硬件辅助的 page shadowing技术 来避免更新时锁住页面。

   > Shadow paging is a copy-on-write technique for avoiding in-place updates of pages. Instead, when a page is to be modified, a *shadow page* is allocated. Since the shadow page has no references (from other pages on disk), it can be modified liberally, without concern for consistency constraints, etc. When the page is ready to become durable, all pages that referred to the original are updated to refer to the new replacement page instead. Because the page is "activated" only when it is ready, it is **atomic**.

5. **Group-by**

   基于hash table。通常使用compact hash table，只用group的属性才会被使用到。

6. **Aggregation**

   聚合操作可以很好地利用列式存储，比如sum( ), min( ), avg( )等可以只扫描相关的列，最大化利用内存带宽（byte/s）。

7. **Indexing**

   CStore提出了projection的概念，为每个表创建多个副本，每个副本根据不同的属性排序（每个副本不一定包含整个表的所有属性）。由于列存储的压缩很好，所以多个副本不会占用太大空间。而projection的数量和种类 取决于workload，太多projection会导致更新的代价变高。

   另一种列存储的索引是 zonemap，在每一个page上存储轻量的元信息（如min/max），从而减少读取没有符合条件元组page次数，加速了扫描。



