## A Distributed In-Memory Similarity-Based Query Processing System

1. **问题**：现有的算法或系统不是完全开发好的 full-fledged，不支持复杂的数据分析；且这些算法 要么不支持大规模地数据分析， 要么 负载上不均衡。
2. **Contribution**: 第一个支持基于相似度的分布式内存型查询处理系统。
   - 扩展了SQL编程接口，支持**similarity search**和**similarity join**
   - 设计了selectable **signatures**，当两条record的签名相同时，它们就近似匹配；从而避免分布式系统中高代价的数据转换
   - 构造了 基于签名的全局和本地索引，支持高效的search和join；同时可以自适应地选择签名来负载均衡
   - 无缝集成到Spark中，并在Spark中开发了有效的优化查询

3. 细节实现：

   - 基于相似度的查询：
     1. **Set-Based Similarity**: 对record进行分词 得到一个token集合，再对比集合的相似度，可使用 Jaccard、余弦、DICE
     2. **Character-Based Similarity**: 如使用 编辑距离edit distance (由一个单词w1转换为另一个单词w2 所需要的最少**单字符编辑操作**次数，包括删除、插入、替换)
     3. **Similarity Search**: 给定查询*s*，相似度函数*f*，阈值*T*，找到所有使*f(r, s, T)*为真的 元组*r*
     4. **Similarity Join**: 给定表R 和表S，找到所有使*f(r, s, T)*为真的 元组对*(r, s)*

   - **Selectable Signatures**: 

     对于每条record *r*，生成两种indexing signature（来源于元组r）和 两种probing signature（来源于查询s）: 

     - **Indexing Segment Signature**

       将*r*划分为*n*个不相交的段 *seg<sub>1</sub>*，*seg<sub>2</sub>*… *seg<sub>n</sub>*. **( seg<sub>i</sub>, i, r )**即为一个indexing segment signature. 而*r*的索引段签名集合 即为*n*个的并。

       *n*的大小可根据 为Jaccard设定的阈值*T* 计算得到。如果**集合s** 与 **集合r** 相似，那么s最多有 **minMatchCnt** = Math.floor( (1-*T*)/*T*  \* |r| )个 和r不匹配的token. 所以当我们把 r划分为 **minMatchCnt**+1 段时，若s与r相似，则s与r **至少有1个段**相同。 （一个segment都不相同，一定不相似，从而可以剪枝）

       在划分时，要保持token的全局顺序，不同record中相同的token必须在同一个段内。可以用哈希函数*h*，将token *t*映射到 第*h(t)*段中。

     - **Indexing Deletion Signature**

       对于每个索引段签名，生成一个indexing deletion signature，即 **( del<sub>i</sub><sup>k</sup> , i, |r| )**，其中 del<sub>i</sub><sup>k</sup> 是将 seg<sub>i</sub> 删除掉第k个token 所得到的子集。

     - **Probing Segment Signatures for Length l**

       如果**记录s** 和**记录r**相似，则s长度必须和r不能差太大；r的长度|r| 应该在区间 [ |s|\**T*, |s|/*T* ] 内。

       对于长度为l的记录，Probing segment signatures即  **( seg<sub>i</sub>, i, l )**

     - **Probing Deletion Signatures for Length l**

       即 **( del<sub>i</sub>, i, l )**.

       重点在于 **剪枝**Pruning，找到 r和s不匹配token数的阈值，超出该阈值则不可能相似。 

   - **Distributed Indexing**

     不同的query会规定不同的阈值，故在offline阶段，需要建立local index；再用**Frequency Table**记录每个签名的频率，从而建立 一个从signature映射到 包含该signature的划分 的global index (不需要用hash表，只需要固定的hash函数将 签名 映射到 划分)

   