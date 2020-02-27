## Learning State Representations for Query Optimization with Deep Reinforcement Learning

1. **Motivation**:

   在数据库领域，查询优化仍然是一个难题。因此尝试探索强化学习在 查询优化上的应用。

2. **Challenge**: 

   状态转移函数（决定了当前子查询状态 如何与下一个operation结合 到达下一个状态）如何定义？

3. **Contribution**:
   - 开发了一种训练模型的方法，该模型学习以增量方式 *incrementally* 生成每个子查询的中间结果 的简洁表示。模型的输入为 **子查询**与**新的操作**，从而来预测 结果子查询的属性，再根据这些性质 可以推断子查询的基数。
   - 提出使用强化学习，通过Markov过程建模 来增量式地 构建query plan，其中每个决策 都基于对应状态的属性。

4. 实现细节：

   - 模型概况

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200227101059.png)

     图中的初始状态为 *t*，通过强化学习 选择出动作，使得模型转变到新状态 *t+1*；每个Action对应一个operation，每个State会包含 子查询中间结果的表示。通过 *NN<sub>ST</sub>*，状态转移函数(是一个神经网络)，来生成中间结果的representation。*NN<sub>ST</sub>*以时间t的子查询表示法为输入，输出为时间t+1的子查询表示法。

     该方法的关键在于，使用一个**有限长度的向量 来 表示每个状态**，并且**使用深度学习模型来 学习 状态转移函数**。（很多场景下的强化学习是已知 状态和状态转移函数的，比如围棋中，棋子在棋盘的位置是可知的，状态转移 也就是从一个棋盘到下一个棋盘的变化 是很清晰明了的）

   - 学习 Query的表示法

     给定数据库*D*，查询*Q*，可以将(Q, D)转化为特征向量，以特征向量为输入、基数结果为输出，来训练深度神经网络；但是该方法中，特征向量的大小 会随D和Q的复杂度上升 而增大，从而会产生很大的稀疏向量，训练集需要很大才行。

     因此，采用一种循环的方式。

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200227111958.png)

     其中， *NN<sub>ST</sub>*: (h<sub>t</sub>, a<sub>t</sub>) → h<sub>t+1</sub>为状态转移函数，h<sub>t</sub>为子查询的特征向量表示（不是人为赋值的，是模型自我学习得到的），a<sub>t</sub>为h<sub>t</sub>上的*single relation operation*。 *NN<sub>Observed</sub>*学习 从子查询representation 到一组观测到的变量 的映射。在训练中，通过后向传播来调整两个函数的权重。

     在使用循环的*NN<sub>ST</sub>*模型之前，需要先学习得到另一个函数，*NN<sub>init</sub>*；其输入为 (*x<sub>0</sub>*, *a<sub>0</sub>*)，其中*x<sub>0</sub>*为一个能表示数据库D属性的向量， *a<sub>0</sub>*为*single relation operator*，其输出为 **执行*a<sub>0</sub>*的子查询的基数**。对于D中的每个属性，使用以下特征来定义*x<sub>0</sub>*：最小值、最大值、域的大小、一维直方图的representation

5. Future Work: 通过结合强化学习模型，来学习得到 最优的执行计划。