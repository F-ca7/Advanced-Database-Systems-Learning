## An End-to-End Learning-based Cost Estimator

1. **问题**：传统基于经验的 代价和基数估计方法 无法提供高质量的估计结果，因为它们 不能有效地把握多表之间的关联。而现有的基于学习的方法有以下不足，
   - 关注基数的估计，没有对cost进行估计
   - 要么不够轻量，要么很难表示复杂的结构(如 复杂谓词)
2. **Contribution**:
   
   - **基于树结构的模型**，开发了一个 高效的 端到端 基于学习的Cost Estimator，可以同时进行 **代价和基数** 的估计
   
   - 提出高效的 **特征提取与编码**技术，在特征提取过程中 会结合考虑 query 和 physical execution，并将提取出的特征 应用到 树结构模型中，利用树结构来估计 cost和cardinality
   
   - 对于带有string类型的谓词，提出一种高效编码字符串值的方法。设计了一种基于pattern的方法 来嵌入字符串值
   
     > 特征嵌入，将数据转换（降维）为固定大小的特征表示（矢量），以便于处理和计算。embedding的主要目的是对（稀疏）特征进行降维，它降维的方式可以类比为一个全连接层（没有激活函数），通过 embedding 层的权重矩阵计算来降低维度。
   
   - 在真实数据集上进行实验，证明了该方法比现有方法的优越性

3. 实现细节：

   - Estimator架构

     dataset + queries → Query Optimizer → **Training Data Generator** → **Feature Extractor** → **Tree-structured Model**

   - **Training Data Generator**

     根据数据集可能的join graph 以及 workload对应的谓词 生成一些query，接着对于每个query 通过优化器获得物理执行计划 (在DBMS中 使用计划分析工具)，并获得真实的cost和cardinality。一条训练数据为一个三元组：<物理执行计划，该计划真实cost，该计划真实cardinality>

   - **Feature Extractor**

     从query plan中提取有用的特征 (如 操作符 和 谓词)，然后query plan中每个结点会编码为 特征向量feature vector，对于简单的特征（如 Operation: Nested Loop, Index Scan…; TableName; Operator）使用**one-hot** vector来编码得到向量；对于复杂的特征（如 LIKE谓词），直接将该每条元组\<column, operator, operand\> 通过一对一映射 得到一个向量。向量再组成tensor，最后形成向量树，

     > one-hot是比较常用的文本特征特征提取的方法。**用N个bit来编码N个状态**，**保证每个样本中的每个特征只有1位处于状态1,其他都是0**。解决了分类器处理离散数据困难的问题。

   - **Tree-structured Model**

   - **Representation Memory Pool**

     在处理query时，存储 **subplan** 到 **representation** 的映射。（就是在动态规划时 子任务结果的存储）

   - **Feature extraction and encoding**

     query node → node vector → tree-structured vector

     ![query plan encoding](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/test1.png)

     **影响查询代价的因素主要有四种**，所以要提取出这些特征。

     - **Operation**

       每个结点都有

       *Join* (Hash/ Merge/ Nested Loop Join)；*Scan* (Sequential/ Bitmap Heap/ Index/ Bitmap Index/ Index Only Scan)；*Sort* (Hash/ Merge Sort)；*Aggregation* (Plain/ Hash Aggregation)

       使用one-hot vector编码，比如1号结点操作类型是Nested Loop，其Operation编码为0001

     - **Predicate**

       join condition (a.movie_id = b.movie_id) 或 filter condition ('year > 1998')。一个谓词由三部分组成，*column*, *operator*, *operand*；前两者用one-hot vector编码，数值型*operand* 通过规格化的浮点数来编码，

       ----
  
       字符串型*operand* 通过 基于embedding的方式——可以表示同一个元组中一起出现的字符串。两种字符串型的谓词，一种精确匹配型(= / IN)，一种是模式匹配型(LIKE)。对所有的字符串子串 进行预训练，而不是对workload中出现的原始字符串进行编码；具体是通过对子字符串的泛化，生成一些规则，这些规则 **可以从数据集中提取出query workload的所有关键词**。再提取出满足规则的子串 来训练模型。
     
       - **Rule Generation**
     
         一条规则由三部分构成(借助DSL的概念)，*Pattern*(匹配数据集中元组的子字符串), *String Function*(决定 提取子字符串中哪个关键词), *Size*(待提取关键词的长度)。此处的*String Function*仅有两种——*Prefix*和*Suffix*
     
         给定一个谓词中的keyword 以及 一个数据集中的string，先找到该字符串所有能匹配keyword的子串。对于每个匹配的子串，生成所有将 **keyword映射到子串**的pattern(通过穷举 Pattern的组合)。
     
         再根据候选规则，找到最优的规则集合(使用最少的规则 来覆盖workload中的关键词)。是集合覆盖问题(SCP)的减弱版。
     
         > 给定一个集合U以及U内元素构成的若干各小类集合S，目标是找到S 的一个子集，该子集满足所含元素包含了U中的所有的元素且使小类集合个数最少。例如，U={1,2,3,4,5}，S={{1,2},{3,4},{2,4,5},{4,5}},找到集合能满足条件的可以有O={{1,2},{3,4}{4,5}}或是O={{1,2},{3,4},{2,4,5}}，
     
       - **String Indexing**
     
         使用 前缀以及后缀trie树索引来存储 字符串→编码 的映射，通过*Prefix*函数提取出的子字符串就放入前缀树，*Suffix*函数提取出的子字符串就放入后缀树，叶子结点就存储字符串的向量表示(即 对应的编码)。
     
         Online Searching的过程中，会有不在dictionary的查询字符串；若它是前缀搜索(LIKE s%)，则搜索该query string的最长前缀，其他同理。即在dictionary中选择 最长的可以匹配的，作为其表示。
     
       -----
     
       如4号结点的filter为 <u>info = 'top 250 rank'</u> ，从Dictionary中找到top 250 rank的表示法，故4号结点的Predicate编码为 000000100000 0001 0.14,0.43,…,0.92
     
     - **Metadata**
     
       列、表、索引的集合，分别都使用 one-hot向量来表示。（有些结点可能没有谓词，所以需要对 metadata也要进行编码）比如5号结点，没有谓词，所以其Predicate编码全部用0填充；其Table Name为movie_info_idx，故编码为0010
     
     - **Sample Bitmap**
     
       固定长度的 0-1向量，每一位表示 对应的元组是否满足query node的谓词条件(满足为1，不满足为0)。如图中的9号结点，production_year>2010的只有第二条元组，故9号结点的Sample_Bitmap编码为0100
   
   - **Model Design**
   
     模型有三层：*embedding layer*, *representation layer*, *estimation layer* 
   
     ![Model Design](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/test2.png)
   
     - **Embedding Layer**
   
       Sparse vector →  dense vector，使用 一个ReLU激活的全连接层 来嵌入 *Operation*, *Metadata*和*Sample Bitmap*，
   
       对于 *Predicate*，使用 Tree Pooling来编码(其中树的结构和 谓词树一样)。其中，叶子结点为全连接神经网络，**OR**使用最大池化层，**AND**使用最小池化层。因此，只有叶子结点才需要训练，容易做到高效的batch training；同时模型收敛更快，性能更好。
   
     - **Representation Layer**
   
       第一个问题 information vanishing: 查询计划中的上层结点 无法捕捉到下层结点之间的关联。
   
       第二个问题 space explosion: 如果要存储足量的信息 来保留表之间的关联，需要大量的存储空间和中间结果。
   
       在表示层使用了 循环神经网络(RNN)，作为一个连接网络，它决定了哪些信息会被传递。如果 表示模型representation model使用朴素神经网络，会有梯度消失和梯度爆炸的问题。因此使用**LSTM**来解决这个问题。
   
       > RNNs理论上是可以将以前的信息与当前的任务进行连接，例如使用以前的视频帧来帮助网络理解当前帧。理论上RNN可以通过调参来解决“长依赖”问题。但是在实践过程中RNNs无法学习到这种特征。可参考 *Learning Long-Term Dependencies with Gradient Descent is Difficult* (1994)
       >
       > Long Short Term Memory networks（以下简称LSTMs），一种特殊的RNN网络，该网络设计出来是为了解决长依赖问题。LSTMs的核心是细胞状态，用贯穿细胞的水平线表示。
       >
       > 细胞状态像传送带一样。它贯穿整个细胞却只有很少的分支，这样能保证信息不变的流过整个RNNs。即图中 *G<sub>t-1</sub>* 到 *G<sub>t</sub>*这一条水平线。
       >
       > LSTM网络能通过一种被称为门的结构对细胞状态进行删除或者添加信息。门能够有选择性的决定让哪些信息通过。其实门的结构很简单，就是一个sigmoid层和一个点乘操作的组合。
       >
       > LSTM由三个门来控制细胞状态，这三个门分别称为**忘记门**、**输入门**和**输出门**。
   
       如图中，*G<sub>t</sub>*即为长记忆的通道，*f<sub>t</sub>* 决定了在长记忆中要遗忘哪些信息（第一步就是决定细胞状态需要丢弃哪些信息），*k<sub>t</sub><sup>1</sup>* 决定了在长记忆中要加入哪些信息（第二步先通过一个输入门来决定更新哪些信息，再通过一个tanh层得到新的候选细胞信息），*k<sub>t</sub><sup>2</sup>* 决定了记忆同道中的哪些信息应该当做子计划的representation
   
     - **Estimation Layer**
   
       两层全连接神经网络，使用ReLU激活函数，输出层为sigmoid 预测归一化后的基数与代价。
   
       由于需要同时预测两个，使用**多任务学习 multitask learning**（参考 Multitask learning. Machine Learning, 1994），其中最常用的一种实现策略是  Parameter sharing，同时训练多个任务，这些任务的模型共享部分神经网络层。
   
       > 只专注于单个模型可能会忽略一些相关任务中可能提升目标任务的潜在信息，通过进行一定程度的共享不同任务之间的参数，可能会使原任务泛化更好。
   
     

