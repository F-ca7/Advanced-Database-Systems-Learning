## Self-Crowdsourcing Training for Relation Extraction

1. **问题**：指导crowd worker的例子和问题
2. **Contribution**: 
   - 介绍了一种 众包的self-training策略
   - 提出一种 针对关系抽取 RE的人机合作训练框架（人类和机器的标注都会在迭代中 逐渐改进）
     1. 使用自动分类器，从样本中自动选出一个*less-noisy*的子集
     2. 使用该子集 训练标注器 annotator
     3. 使用标注后的数据 再迭代地训练分类器
   - 在标准基准测试下，只比 state-of-the-art的方法性能差一点

3. Relation Extraction: 

> Relationship extraction is the task of extracting semantic relationships from a text. Extracted relationships usually occur between two or more entities of a certain type (e.g. Person, Organization, Location) and fall into a number of semantic categories (e.g. married to, employed by, lives in).

4.  Distant Supervision (DS): 

   2009年Mintz等人提出了**远程监督**方法，借助外部知识库 Knowledge-Base （如 wiki, Freebase） 为数据提供标签，从而省去人工标注的麻烦。Mintz提出一个假设，如果知识库中存在某个实体对的某种关系，那么所有包含此对实体的数据都表达这个关系。理论上，这让关系抽取的工作大大简化。

   但远程监督也有副作用（噪声很大），因为不用人为的标注，只能机械地依赖外部知识库，而外部知识库会将同一对实体的所有情况都会标注一种关系，其标签的准确度就会大大的降低。

5. **Related Work**:

   - 把 DS data 和少量通过众包得到的人类标注数据 结合起来，提高准确率
   - 通过 主动学习active learning（针对数据标签较少或打标签“代价”较高的场景）来选择给众包的最好样本
   - 通过 *Gated Instruction*来训练crowd worker

6. **实现细节**：

   为什么叫Self-Crowdsourcing Training？-> 因为想要通过分类器的得分来训练crowd worker。

   - **Silver Standard Mining**

     基于有噪声的数据集 DS，以及一个由众包标注的数据集 CS。首先，将CS分为三部分：

     - CS<sub>I</sub>: 用于给crowd worker的指引。先在DS上训练出分类器C，再用分类器C来标注CS样本。选择预测分数最高的 Top N<sub>I</sub>个 作为CS<sub>I</sub>
     - CS<sub>Q</sub>: 用于句子标注的问题。从 CS－CS<sub>I</sub>的集合中再选择 Top N<sub>q</sub>个样本作为CS<sub>Q</sub>
     - CS<sub>A</sub>: 用于 收集 训练后的标注器 所标注的标签。即CS中剩下分数最低的样本，交给crowd worker

   - **Training Schema**

     训练标注器分为两步：

     1. **User Instruction**: 首先定义关系的类型，接着告诉crowd worker一系列的例子，比如定义 *Has nationality of* 不是 *live in* 或 *was born in*；这样有利于不同专业水平的人 来训练标注器
     2. **Interactive QA**: 给workers交互式的多选任务，并智能纠正他们的错误——告诉他们哪里错了

7. **Future work**: 将该方法推广到 其他更简单或更难的任务上。