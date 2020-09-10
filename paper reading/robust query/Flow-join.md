## Flow-join: Adaptive skew handling for distributed joins over high-speed networks

1. 问题：分布式join的scalability会受到数据倾斜(skew)的影响，即某单台服务器节点上会处理大部分的输入，导致整个分布式查询变慢

2. **Contribution**:
   - 在对输入partition的同时，检测heavy hitter，只需要很小的开销
   - 可以在运行时调整redistribution策略 ， 从而能适用于heavy hitter的key子集；通过广播对应的元组，避免heavy hitter集中在一台服务器上
   - 基于以上两种机制，提出Flow-Join，利用RDMA来shuffle数据；同时区分开本地与分布式的并行度，
   - 使用 Symmetric Fragment Replicate的redistribution策略，来应对两个输入端相互关联的数据偏度

3. 存在问题示例：

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200910020916.png)

4. Heavy Hitter检测算法：

   基于*Efficient computation of frequent and top-k elements in data streams* *(2019)* 的SpaceSaving algorithm，该算法不能保证其找到的元组都是频繁项，但是多广播一些元组不会对性能有很大问题。

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200910021118.png)

5. Join流程：假设*R*作为build端输入，*S*作为probe端输入。传统上来说，通常是小的表作为build端，从而哈希表占用空间会小一点。

   1. Exchange *R*

      对于*R*中每个元组

      - 更新近似直方图
      - 根据阈值判断heavy hitter：如果是，则将该元组插入到local哈希表中；若不是，则将该元组发送到对应的节点
      - 创建*R*的全局heavy hitter list

   2. Exchange *S*

      对于*S*中每个元组

      - 更新近似直方图
      - 根据阈值判断heavy hitter：
        - 如果是在*S*中的倾斜，对该元组进行物化
        - 如果是在*R*中的倾斜而不是*S*中的，广播该*S*的元组到所有节点
        - 都不是，则将该元组发送到对应的节点
      - 创建*S*的全局heavy hitter list

   3. Handle skew (当*S*中存在倾斜才需要)

      1. 对*R*中不是heavy hitter，但需要和*S*的heavy hitter做join 的元组进行广播（与物化后的*S*的元组做join）
      2. 通过**Symmetric Fragment Replicate**的重分布策略，对join key既是R heavy hitter、也是S heavy hitter的元组进行redistribute

6. **Symmetric Fragment Replicate**

   原于论文 *A symmetric fragment and replicate algorithm for distributed joins 1993.*

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200910041550.png)

   Servers replicate heavy hitters for one input across rows, while those of the other input are replicated across columns. 保证一端输出的每个heavy hitter元组 和 另一个输出端的heavy hitter元组 能进行join。

7. 相关工作的对比

- *Practical skew handling in parallel joins 1992.*

  最早提出 对于 heavy hitter的join对象进行replication，不过这里的partition是基于range的。

- *Handling data skew in parallel joins in shared-nothing systems 2008.*

  将上述思想应用于hash的partition，需要详细的数据统计信息，来提前知道倾斜的数据值（非运行时），可能会有很大误差。

- *Tuple routing strategies for distributed eddies 2003.*

  核心思想： routes tuples between operators，从而解决operator的压力不均衡；但是不能解决数据倾斜的问题，比如一个很大的partition里可能全是同一个heavy hitter 的值

- Track join: Distributed joins with minimal network trafﬁc

  引入track阶段，来决定每个join的值应该路由到哪里，做到最小的网络流量；但是track阶段本身是一个独立的分布式join，对数据倾斜同样敏感

- *Frequent item computation on a chip 2011*

  利用了FPGA，在硬件中计算频繁项；对本文算法的性能有很大提升，

