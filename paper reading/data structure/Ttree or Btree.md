## T-Tree or B-Tree: Main Memory Database Index Structure Revisited

1. **问题**：T树被广泛认为是适合 内存型数据库的索引结构，但是大多数研究没有考虑**并发控制**concurrency control.

2. **结论**：当使用并发控制时，B树的变种——B链树 B-link Tree 比T树表现得更好。因为T树需要的锁操作更多，而上锁和解锁的代价很高。

3. [B-link Tree](https://sciencedirect.xilesou.top/science/article/pii/0022000086900218/pdf?md5=fafabf86d6f6aced3c490eacb5d30d46&pid=1-s2.0-0022000086900218-main.pdf&_valck=1): 

   

4. T-tail Tree: 

   **结点设计**：参考[06 index-notes](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/06%20index/6-index-notes.md)

   对T树的改进。当T树结点溢出时，允许一个额外的结点链接到该结点，从而延迟树的旋转操作。树的旋转是由于结点的上溢和下溢造成的，即 对**固定大小的结点**进行插入或删除Key的操作造成的。所以，在T-tail Tree中，一个结点允许有一个T-tail。

   ![](https://s2.ax1x.com/2020/02/09/1fxHSA.png)

   当插入满结点时，就为其创建一个tail结点，接着所有该结点的插入 都在tail上进行。只有当tail结点也满了，才说该T结点是完全满。当删除导致结点下溢时，tail结点的key会补充进来。

5. T-tail Tree采用的锁机制：

   - S-lock
   - SIX-lock
   - X-lock

   T-tail Tree的两种并发访问方式：

   - 悲观：参考[Concurrent search and insertion in AVL trees](https://ieeexplore.ieee.xilesou.top/abstract/document/1675680/)

     搜索时 从根结点到边界结点bounding node 使用 lock-coupling；更新时 所有从关键结点 critical node（边界结点最近的祖先，且该祖先结点的平衡因子为±1）到边界结点 上使用SIX-lock，以备树的旋转。如果后面发生了树的旋转，SIX-lock 就转化为 X-lock.

   - 乐观：搜索时不会锁，更新时 同一时间最多锁一个结点。只有当插入一个*完全满*的结点时，才会(互斥)锁住整棵树（固定状态）。

6. 基本Cost的评估：

   ![](https://s2.ax1x.com/2020/02/09/1hiR9U.png)



