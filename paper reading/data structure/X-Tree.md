## The X-tree: An Index Structure for High-Dimensional Data

1. **问题**：高维数据的索引在很多应用中 变得越来越重要，然而当前所流行的R*-tree在很高维度的情况下 性能会急剧下降，其主要问题在于 Bounding Box的重叠。

2. **Contribution**: 

   - 提出 X-树结构，使用了一种最小化overlap的算法来 分裂结点
   - 通过实验证明，当数据维度高于2时，X-树 比 TV-树和R*-树都要快

3. **实现细节**：

   - 使用了**基于R-树的结构**

     因为不只要对 空间点数据进行索引，也要对扩展的空间数据进行索引，而R-树就同时适合这两种类型。

   - Overlap的定义

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20200302122900.png)

     如果查询的点遵守均匀分布，则Overlap可以定义为 超矩形交集的并÷超矩形的并；但如果查询的点不均匀、可能聚集在一些地方，则需要用 带权重的Overlap

   - X-树结构

     ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/QQ%E6%88%AA%E5%9B%BE20200302194136.png)

     - **Data node**

       包含 **最小边界矩形** Minimum Bounding Rectangle、**指向实际数据的指针**。

     - **Directory node**

       包含 **MBR**、**sub-MBR**

     - **SuperNode**

       large directory nodes of variable size(块大小的倍数)。目的是 防止directory分裂后形成低效的结构

   - **插入算法**

     ```c++
     int X_DirectoryNode::insert(DataObject obj, X_Node **new_node)
     {
         SET_OF_MBR *s1, *s2;
         X_Node *follow, *new_son;
         int return_value;
         // 找到数据插入的子树
         follow = choose_subtree(obj); 
         // 将数据对象插入子树
         return_value = follow->insert(obj, &new_son); 
         // 更新原子结点的MBR
         update_mbr(follow->calc_mbr()); 
         if (return_value == SPLIT){
             // 向当前结点插入子结点的MBR
             add_mbr(new_son->calc_mbr()); 
             if (num_of_mbrs() > CAPACITY){ 
                 // 上溢出
                 if (split(mbrs, s1, s2) == TRUE){
                     // 可以做到最小化overlap的split
                     set_mbrs(s1);
                     *new_node = new X_DirectoryNode(s2);
                     return SPLIT;
                 } else  {
                     // 没有好的split方法
                     *new_node = new X_SuperNode();
                     (*new_node)->set_mbrs(mbrs);
                     return SUPERNODE;
                 }
         	}
         } else if (return_value == SUPERNODE){ /
             // 如果follow结点称为超结点
             remove_son(follow);
             insert_son(new_son);
     	}
         return NO_SPLIT;
     }
     ```

   - **Split算法**：返回是否可以最小化overlap

     ```c++
     bool X_DirectoryNode::split(SET_OF_MBR *in, SET_OF_MBR *out1, SET_OF_MBR *out2)
     {
         SET_OF_MBR t1, t2;
         MBR r1, r2;
         // 先尝试拓扑分裂
         topological_split(in, t1, t2);
         r1 = t1->calc_mbr(); r2 = t2->calc_mbr();
         // 测试overlap是否超出指标
         if (overlap(r1, r2) > MAX_OVERLAP)
         {
             // 尝试最小化overlap分裂
             overlap_minimal_split(in, t1, t2);
             // 测试结点是否平衡
             if (t1->num_of_mbrs() < MIN_FANOUT || t2->num_of_mbrs() < MIN_FANOUT) {     
                 // 都失败则生成supernode
                 return FALSE;
             }
         }
         *out1 = t1;
         *out2 = t2;
         return TRUE;
     }	
     ```

     

4. **Related Work**: 

   - **Point transformation**
     - kdB-tree
     - grid file

   - 利用真实的高维空间数据是 **高度关联的**且**聚集的**特性，来进行降维 从而再使用传统的多维索引
     - Fastmap
     - multidimensional scaling
     - principal  component  analysis (PCA)
     - factor  analysis
     - SS-树（基于R-树的索引结构）
   - 在大多数高维空间数据集中，只有小部分的维度承载了大部分的数据
     - TV-树： higher fanout and smaller directory