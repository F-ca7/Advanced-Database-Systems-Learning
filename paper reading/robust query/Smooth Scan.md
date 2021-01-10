## Smooth Scan: Robust Access Path Selection without Cardinality Estimation

1. **Motivation**:

2. **Contribution**: 
   - 提出新的*smooth access path*算子，该算子可以根据**运行时的统计信息**，从一种实现过渡到另外一种实现
   - 实现了一种对不用感知统计信息的算子——Smooth Scan，根据运行时selectivity的变化，来进行*index scan*和*full scan*的动态过渡转化(morph)
   - 对Smooth Scan策略在最坏情况下的性能保障 提供了理论分析
   
3.  不同的扫描算法：
   - 全表扫描：当没有其他access path，或者access path的**选择率很高**时，会使用全表扫描
   - 索引扫描：是**随机读取**（所以如果选择率很高的话，索引扫会发生很多次随机读取，导致性能下降）
   - 排序扫描：先利用索引找到所有满足条件的元组id，再对id排序，从而利用预读取来达到（近似）顺序扫描的效果；但是无法利用B树索引的有序性，来进行一个range query
   
4. 流程：

   在*index scan*到*full scan*的过渡中，有三种模式。

   1. Index scan: 当存在索引时，首先还是会从索引扫描开始。过程中，会持续关注结果集的cardinality，达到阈值时，进入下一个状态；

   2. Entire page probe: 通过CPU开销来换取IO的开销，会分析每个装载的page中满足条件的元组（原本的index scan只会取该page中索引指向的那一条记录）；

   3. Flattening: 结果集越来越大时，**Smooth Scan**会将随机IO的代价 分摊到 用顺序读代替随机读上，原理就是<u>每次回预取邻近的page，做一个顺序读</u>；结果集继续增大，则可以逐渐地flatten，从而过渡为近似的full scan

   预期达到的效果如下：（虽然当选择率比较低时，一开始比走纯索引慢，但是随选择率上升，执行时间更平滑）

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210110_103847.png)

5. 策略
   - **Greedy**: 当选择率很高时，Smooth Scan的morphing size会随每一次索引访问而扩大；显然，如果选择率没那么高，会带来额外开销
   - **Selectivity increase**: 当检测到选择率上升（局部大于全局）时，会扩大morphing size；没有上升，则不变
   - **Elastic**: 比如在大数据集的场景，由于数据分布的偏度，元组在磁盘中可能有分布稀疏和密集的区域；所以弹性策略，当局部的选择率高于全局时（密集区），会扩大过渡区morphing size，反之亦然。
6. **Morphing**触发时机
   - Optimizer driven: 当结果集的cardinality超过优化器的估计时，开始使用Smooth Scan
   - SLA driven
   - Eager: 所有access path都使用Smooth Scan的算法
7. 实现设计
   - **Page** **ID** **cache**: 保存读取过的page，bitmap的结构（每个bit对应一个page）从而Smooth Scan每次只读取之前没有访问过的page
   - **Tuple ID cache**: 如果是从Mode0开始的，需要保证后来不会重复读取到之前读过的元组，也是bitmap结构
   - **Result cache**: 是hash的结构。当索引可以保证query需求的顺序时，需要先要对每个叶子节点页面的tupleID 进行Result Cache的hash probe，再进行index probe；如果该元组在cache中找到了，则直接返回，否则正常从磁盘中读取
   - **Memory management**: 对于bitmap结构，通常几百GB的数据最多只要几MB的内存；但是result cache可能会增长得很大，思想是对其进行partition，写到临时文件中（根据B树根节点来决定划分的数量）