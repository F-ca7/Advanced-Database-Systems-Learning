## Optimizing Queries with Crowd-Based Selections and Joins

1. **问题**：（现有系统，如CrowdDB/ Qurk/ Deco）
   - 树形模型只能提供表级连接 粗粒度的优化，限制了不同join的元组能有有不同的join次序。
   - 主要关注经济上的开销，忽略了lower latency 和higher quality的优化目标（见 [Crowdsourced Data Management）](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/crowdsource/Crowdsource%20data%20management.md)

2. **Contribution**: 

   - **采用基于图的查询模型，提供更细粒度的查询优化**

     在SQL中加入众包操作，提出CQL。请求发起者requester 可以使用CQL DDL来让crowd填写或收集需要的数据，使用CQL DML对数据进行众包操作（如crowdsourced selection/ join）

     根据给出的CQL查询，可以构建一个图模型，其顶点为表中的一个元组、每条边根据join/ selection的谓词来连接两个顶点；从而提供元组级的查询优化

   - **基于图模型 使用统一的框架来进行多目标的最优化**

     1. 使用图模型来描述task selection问题（证为NP-hard），提出xx算法来**减少cost**
     2. 在一轮中，尽量同时问 不能被其他答案推出的 问题，从而减少轮数，**减少了latency**（但是 cost 和 latency 之间有tradeoff）
     3. 提出一个适合不同类型任务的 task assignment 和 truth inference的框架，**提高quality**

3. **Conclusion**: 通过模拟与现实实验，证明了CDB众包数据库系统在cost/ latency/ quality 上的优越性。

4. **细节**：

   - **FILL**和**COLLECT**的数据清洗：

     首先为用户提供自动补全功能，以减少同一个实体不同的说法；接着使用crowdsourced entity resolution (详见[CrowdER](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/crowdsource/CrowdER.md))。

   - **Join**的图模型：

     每条边的权重是 两个元组t<sub>x</sub>和t<sub>y</sub>分别的两个属性t<sub>x</sub>[C<sub>i</sub>]和t<sub>y</sub>[C<sub>j</sub>]匹配的概率。比如使用Jaccard相似度，两个属性值的相似度越大；相似度即为边的权值，如果 权值*w(e)*<阈值*ε*，则去除该条边（匹配的可能性很小）。

     为方便计算，假设边的权重(即匹配概率)是独立的。设有*N*个连接谓词的CQL query对应模型图*G*，若*G*中有*N*条边的连通子图包含query谓词的每一条边，则该子图称为一个**Candidate**. （必须要准确有*N*条边，不能多不能少）。

     如果一条边不在任何Candidate中，则称该条边是无效的。可以使用深度优先遍历 来检查某条边是否在一个candidate中。所有无效的边可以从图中被去除。

     接着让crowd来决定每条边是否符合对应的谓词，如果符合，将该条边标蓝；如果不符合，则标红。最后，有*N*条蓝边的Candidate就是答案。(不需要把所有边问出来，有可能一条边标红，与之相关的几条也会断掉)

   - **Cost Control**: 通过task selection实现
     - 已知边的颜色
       - Chain Join: 每张表最多和两张表相连接，且不形成环路
       
         使用深度优先算法找到有N条蓝色边的链BLUE Chain，这些边必须作为问题给crowd。
       
         再通过最小割 min-cut算法，找到必须作为问题的红色边，从而其它的红色边 可以被剪枝。
       
       - Star Join: 一张中心表与其他所有表连接
       
       - Tree Join: 无环路，可转换为Chain Join
       
       - Graph Join: 有环路，可转换为Tree Join
       
     - 未知边的颜色
     
       使用贪婪算法来找到 覆盖S个样本的最少边数(NP-hard问题)，需要生成很多样本，代价很高 =》基于期望的方法，若剪去一条边会造成其他的边invalid，则该边的 剪枝期望 pruning expectation为 (1-*w(e)*) * 造成invalid边的数目。最后可得到总的每条边的剪枝期望，从大到小排列 即可。
     
     - 给定预算

5. **Future Work**: 
   
   - Join时的匹配概率 不是独立分布的（可能有正相关或负相关等关系）



