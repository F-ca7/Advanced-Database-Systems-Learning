## Pessimistic Cardinality Estimation: Tighter Upper Bounds for Intermediate Join Cardinalities

1. **Motivation**

   针对多表join中间结果基数的估计问题。

   目前优化器在filter谓词过滤上的估计 表现是比较良好的（有很多不同的辅助方法），然而对于join谓词的话，就不得不基于很多假设以及默认值。

   比如优化器会做出如下假设：不同表中join列是相互独立的，然而现实中这些列可能是高度关联的。并且 要注意：高估基数会导致稍微较慢的查询，但是**低估基数会导致执行时间的爆炸性膨胀**，所以假设独立性是有很高风险的（因为假设独立性 等价于 低估了选择率）。

   对于key与foreign key的join，基数不会超过foreign key表的大小（1:n的连接），但是 foreign key与foreign key的join 就很难给出严格的上界保证了（当然除了笛卡尔积本身）。并且如果join的两个关系 本身是其他join产生的中间结果，就更难使用特定的方法去进行基数估计了。  

2. **Contribution**

   - 通过随机hash划分，给join基数提供更严格上界保障
   - 给出了一种简单的算法，来生成 定熵公式（本质是上界约束）
   - 实现了一种划分预算策略来控制空间复杂度 以及 边界计算的时间复杂度
   - 通过基于数据集GooglePlus community graphs、 Join Order Benchmark的实验，证明了方法的实用性

3. **Related Work**

   - *Sketch Techniques for Approximate Query Processing(2010)*：本文主要基于该篇文献提出的sketch方法 进行了改进。在聚合函数中经常使用sketch的方法来估计。包括Count-Min Sketch, Count Sketch, AMS Sketch等。
   - Sampling: 采样法，在实际应用中，有以下两个问题：(1) 当有多表连接的情况，如果采用均匀采样可能会导致样本刚好无法join；(2) 当谓词选择率很高时，样本数可能不够，导致无法反映真实的选择率。可参考 [Two-Level Sampling for Join Size Estimation (2017)](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/query/join%20size%20estimation.md)，其提出了Two-Level Sample算法，并指出：关联采样法对于发现表中连接列之间的相关性非常有效，而伯努利采样法对于发现同一张表不同列之间的相关性非常有效。

4. 背景知识：

   - **超图 Hypergraph**

     普通的图graph中，**一条边**edge只能连接两个顶点vertex；在超图中，一条边可以和任意多个顶点连接。

     ![](https://tse1-mm.cn.bing.net/th?id=OIP.Wj-78RVzpLpkkMCyIVXGAgHaFW&w=201&h=160)

     图中，边集 `E={e1,e2,e3,e4}={{v1,v2,v3}, {v2,v3}, {v3,v5,v6}, {v4}}`

     对于如下的sql查询

     ```sql
     SELECT
     MIN (a1.name) AS writer_pseudo_name ,
     MIN (t.title) AS movie_title
     FROM
     aka_name AS a1 ,
     cast_info AS ci ,
     company_name AS cn ,
     movie_companies AS mc ,
     name AS n1 ,
     role_type AS rt ,
     title AS t
     WHERE
     cn. country_code = '[us]'
     AND rt. role = 'writer'
     AND a1.person_id = n1.id
     AND n1.id = ci. person_id
     AND ci.movie_id = t.id
     AND t.id = mc.movie_id
     AND mc. company_id = cn.id
     AND ci.role_id = rt.id
     AND a1.person_id = ci. person_id
     AND ci.movie_id = mc.movie_id;
     ```

     其对应的超图如下所示

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210219_110710.png)

     

   - Robust Queey Optimization: 不一定要选出最快的执行计划（估计代价最低的），重点关注的是 **避免选出糟糕的计划**。可参考 [Robust query processing: Mission possible](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/robust%20query/Robust%20query%20processing%20Mission%20possible.md)

   - **Bound Sketch**（一个数据结构）

     | 符号               | 含义                                               |
     | ------------------ | -------------------------------------------------- |
     | T(**a**)           | 为包含属性**a**的表（**a**可包含多个属性）         |
     | *H*                | 随机哈希函数，W -> {1, 2, ..., M}                  |
     | [M]                | 哈希函数的定义域                                   |
     | *I*                | 哈希值数组，*I* ∈ [M]<sup>\|**a**\|</sup>          |
     | T<sup>*I*</sup>    | T的子集，满足 {t ∈ T : *H*(T[a]) = *I*[a]}         |
     | c(T<sup>*I*</sup>) | \|T<sup>*I*</sup>\|，即T<sup>*I*</sup>有多少个元组 |

     以下图 表R为例，有两个属性x和y

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210227_074759.png)

     令哈希函数为*H(x)*=x%2，因此数组*I* 有4个元素（每个元素可理解为**一个sketch**），分别对应x, y属性值模2等于(0, 0)、(0, 1)、(1, 0)、(1, 1)的情况（可以推论说，hash函数的值域不能太大），最后可求得以下参数

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210227_080310.png)

     其中，c<sub>R</sub>代表了 数组*I*每个元素的 元组数，d<sub>R</sub>[x]代表了 数组*I*每个元素中 属性x相同值出现最多的次数（也就是most common frequency）

   - **Joint entropy** 联合熵: 信息论中的概念。信息论认为 信息是一种不确定性的度量，与事件的概率相关。**信息熵用来表示每个 符号平均携带多少比特（bit）信息**。而联合熵上与联合分布相关，表示为H(X,Y)=∑∑P(X,Y)·log(P(X,Y))。

     令 Q(x,y,z): R(x, y), S(y,z), T(z,x)表示R、S、T连接后的大小，(X, Y, Z)为对应属性值的三元随机变量（认为**呈均匀分布**）。

     核心： The size of the query is tied to the joint entropy of all three variables.

     则联合熵 h(X, Y, Z) = log | Q(x, y, z) |。又假设**查询只包含 相等谓词**，可以得到 变量子集熵的上界，即h(X, Y) <= log(c<sub>R</sub>)，h(Y, Z) <= log(c<sub>S</sub>)，h(Z, X) <= log(c<sub>T</sub>)。

     再根据条件熵（在一定已知条件下）的公式有：

     h(X | Y) <= log(d<sub>R</sub>[y])，h(Y | Z) <= log(d<sub>S</sub>[z])，h(Z | X) <= log(d<sub>T</sub>[x])

     根据Shearer定理，可以得到熵h(X, Y, Z)的上界：

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_120650.png)

     可以将KNS上界表示为：

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_120832.png)

5. 具体实现Selection Propagation and
Preprocessing
   
以JOB基准中的查询08c为例（其中实线箭头表示PK-FK的join（箭头指向PK），虚线表示FK-FK的join）：
   
![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_121026.png)
   

   
接下来分为以下几步：
   
   1. **Selection Propagation and Preprocessing**
   
      Selection propagation可以帮助消除参与的JOIN属性。这个预处理的时间 比起糟糕的计划选择带来的时间膨胀 就相对小了很多。
   
      核心：
   
      - 如果参与join条件的属性 还参与了filter条件，就可以把 filter条件给传递到join等值之类的条件。
      - 外键对应的主键属性，给合并到外键上，从而减小超图的大小
   
   2. **Hash Partition Budgeting**
   
      主要需要解决这个问题——每个join属性需要用多少个hash桶来装？
   
      核心思想：给定一个桶数量阈值*B*（可理解为哈希函数的值域），对于每个定界公式，只计算最多*B*个哈希值组合的上界。
   
      以查询Q (x, y, z, w) :- aka (x, y) , ci (y, z) , mc (z, w) , cn (w)为例，可以得到以下两个熵公式
   
      ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_123914.png)
   
      下图则展示了公式7、8对应的哈希桶划分
   
      ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_123744.png)
   
      可以推得性质：增大桶的划分数，可得到越严格的上界。
   
      
   
   3. **Bound Formula Generation**
   
      遍历超图中的 所有表的覆盖集，对于集合中的每个关系 计算其在定界公式中的贡献多大。
   
      对于如下超图：
   
      ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_125646.png)
   
      为了覆盖每个点，有以下8种集合覆盖选择：
   
      ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_125753.png)
   
      对于每种覆盖的组合，给出上界的公式如下：
   
      ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210228_122005.png)
   
      可以看出，式2、3、4、6将查询图分成了非连通图；而1、5、7、8将查询图组合为单链状。
   
      其中，只有式2、4、6、8包含了c<sub>cn</sub> (为表`company_name`的count)，所以它们能够从`company_name.country_code`的过滤中 获得更多好处。这是因为 `companyID`是表`company_name`的key，但是选择率不是很良好，所以d<sub>cn</sub><sup>companyID</sup>这一项给出的值基本是1。
