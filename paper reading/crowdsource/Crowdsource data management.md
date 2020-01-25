## Crowdsourced Data Management: A Survey

1. 众包数据管理主要问题：

   - **质量**quality

     恶意worker给出错误答案、worker缺乏相关经验，会导致众包得到的结果包含很多噪声。可对worker进行建模，再通过以下方法来提高结果的质量：

     - worker elimination：消除低质量的worker
     - answer aggregation：将同一个任务分配给多个worker再聚合他们的答案
     - task assignment：将任务分配给合适的worker

   - **开销**cost

     把整个任务直接全交给人力来做，很贵。有以下方法来控制开销：

     - 剪枝：可参考[Crowdsourcing Entity Resolution](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper reading/crowdsource/CrowdER.md)中提出的人机混合的c处理方式

     - 任务选择：为众包任务划分优先顺序

     - 答案推断：只把原任务的子集众包出去，根据收集到的大男来推断其他任务的答案

     - 根据特定的operator(众包操作)来设计开销控制

       **operator**：如[sort和join](https://arxiv.xilesou.top/pdf/1109.6881.pdf)。其中，sort用于如用户对产品的打分、评价、投票等，join用于两个数据集的整合和去重，如entity resolution。

       sort的实现：1. **Comparison-based**；2. **Rating-based**；3. **Hybrid Algorithm**，先使用基于评分机制的sort并生成一个列表L，再使用基于比较机制来迭代地修正L大小为S的子集，其中迭代数决定了最终答案的准确率。选取子集有三个方法——随机法、置信度法、滑动窗口法。

       join的实现：设两张表分别为***R***和***S***。1. **Simple Join**，worker直接判断两个元组是否满足谓词条件，总HIT任务量为|***R***|\*|***S***|；2. **Naive Batching**，worker一次性选出多对元组是否满足，总HIT任务量为|***R***|\*|***S***|/b，b为一批次的大小； 3. **Smart Batching**，分为两列，一列来自表***R***，一列来自表***S***，让worker去进行连线匹配，总HIT任务量为|***R***|\*|***S***|/(r*s)，r和s分别为两列的选项数。(看实际例子理解比较容易)

       ![](https://s2.ax1x.com/2020/01/24/1ZaCVS.png)

       

   - **等待时间**latency：

     可能会花费很长时间。策略如下：

     - 用更高的价格去雇佣

     - latency modeling：1. 轮次模型：任务分多轮发布，每一轮的时间可视作常数；整体等待时间可用任务的轮数来描述。2. 统计模型：从先前的众包任务来建立统计模型，再来预测下次整体等待时间。

     - Budget Allocation Strategy：采用基于动态规划的预算分配算法（多项式时间复杂度），来最小化latency。之前的研究通常关注cost和accuracy，要么是 在给定最大的问题数下 使准确率最大化、要么是 在给定的准确率阈值下 使问题数最小化。而 *An Optimal-Latency Budget Allocation Strategy for Crowdsourced MAXIMUM Operations* 中关注latency的指标。

       通过竞赛图tournament graph（通过在无向完全图中为每个边分配方向而获得的有向图）定义了MinLatency Problem。事实上，只需要是完全图，不需要有向性。

       设图G(c<sub>i−1</sub> , c<sub>i</sub>): 由c<sub>i−1</sub>个顶点形成的、有c<sub>i</sub>个团(clique，满足两两之间有边连接的顶点的集合)的图。

       ![](https://s2.ax1x.com/2020/01/24/1Z425j.md.png)

       每一轮的winner会进入下一轮(一个tournament只会有一个winner)。其中，winner可由reliable worker layer (RWL) 的算法决定，用以消除人为的误差或主观性，如[Dynamic max discovery with the crowd](http://ilpubs.stanford.edu:8090/1032/2/winner-long.pdf)提出的。

   - 其他

     - **任务设计**Task Design：包括任务类型(单选/多选/评分/聚类/打标签)、任务设置(定价格/时间约束/质量控制)。
     - **特定操作**specialized operators：每种operator可对应一种或多种任务类型。比如Filtering、Find等对应单选，CrowdMiner对应标签。
   
   

