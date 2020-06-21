## Flexible Paxos: Quorum intersection revisited

1. Paxos

   包括两个阶段，都需要保证majority agreement才能继续，其中quorum指的是参与者的子集。而Paxos的安全性与活性(liveness ) 是基于 任意两个quorums都会有交集（只要满足majority，显然就会有交集；不过也有其他的quorum方案被提出）。

   三个角色：

   - proposer: wishes to have a particular value chosen
   - acceptor: agrees and persists decided values
   - learner: wishing to learn the decided value

   两个阶段：

   - Phase 1 - **Prepare & Promise**
     1. Proposer 选择唯一提议号*p*，并向Acceptor发送 *prepare(p)*
     2. Acceptor接收到*p*，如果*p*是最高的提议号，则将*p*写入到持久化存储，并且回复 *promise(p', v')*，其中(p', v')为最后一次接受的提议
     3. 当Proposer 收到大多数Acceptor的*promise*时，进入第二阶段。否则，Proposer 会用更大的提议号重试。
   - Phase 2 - **Propose & Accept**
     1. Proposer 选定值 *v*。

   **Liveness**

   > A liveness property states that, under certain conditions, some event will ultimately occur. 
   >
   > **Safety** and **liveness** are two important kinds of properties provided by all distributed systems. Informally, safety guarantees promise that nothing bad happens, while liveness guarantees promise that something good eventually happens. Every distributed system makes some form of safety and liveness guarantees, and some are stronger than others. For example, **atomic consistency** guarantees that operations will appear to happen instantaneously across the system (safety) but operations won’t always succeed in the presence of network partitions (liveness, in the form of availability).

2. Multi-Paxos

   实际中的使用，是希望一连串的值能够达到一致，即Multi-Paxos。所以一般使用 Paxos的第一阶段**election** 来选出*leader*，使用  Paxos的第二阶段**replication** 来记录一连串的值。只有当*leader*与quorum交互 并 等待它们接收值后，该值才能被提交。

3. FPaxos (Flexible Paxos)

   在Paxos的每个阶段，都不需要majority quorum，即无交集的quorum是安全的。当然，如果选择有交集的quorum就和原Paxos等价。

   其优势就在于 可以减小replication阶段需要保证参与的结点数，由于leader不需要等待大多数接受，所以减小了系统的等待时延。这样的代价是 **减少了可用性**（leader宕机后的恢复过程）。

4. 

