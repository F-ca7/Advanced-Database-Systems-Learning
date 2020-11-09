## Bridging the Archipelago between Row-Stores and Column-Stores for Hybrid Workloads

1. 当DBMS既要适合OLTP，又要适合OLAP，称为HTAP(hybrid transactional-analytical processing)
   - 支持现代OLTP要求的 **高吞吐量和低延迟**
   - 支持复杂的、长时间的OLAP查询——包括 hot(事务型) 和 cold(历史型)数据的操作
   - **难点**：在OLAP获取 旧数据和新数据 的同时，执行事务更新数据库
2. 使用一个统一的架构来连接OLAP与OLTP之间的鸿沟——基于DBMS未来访问元组的方式，使用**混合结构(hybrid layout)**来存储数据表:
   - hot tuple(热点元组，最近时间内经常会发生变化): 以针对OLTP操作优化的方式存储
   - cold tuple: 以更方便OLAP查询的方式存储
3. Storage Model：存储模型
   - NSM(N-ary Storage Model)：对于单个元组，DBMS将其所有属性以**连续(contiguously)**的方式存储（行存）
     - 适合OLTP型工作，因为事务中的queries 倾向于 **一次对一个独立的entity进行操作**
     - 不适合OLAP，因为分析型queries 倾向于 一次获取一个表中多个entities的属性(子集)，而NSM读表时，只能是tuple-at-a-time(可参考 [query-compilation](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/03%20compilation/3-compilation-notes.md))，即 以元组为中心的方式——浪费 **I/O** 以及 **内存带宽** 去访问 不必要的属性
   - DSM(Decomposition Storage Model)：对于数据表的一个属性，DBMS将**该属性的所有值**以**连续**的方式存储（列存）
     - 适合OLAP，因为DBMS只需要取到query需要的属性值，以attribute-at-a-time的方式；提高CPU效率——lower interpretation overhead & skipping unnecessary attributes
     - 不适合OLTP，比如插入时，需要把元组的属性值放在分别的存储位置
   - FSM(Flexible Storage Model)：一种混合存储结构。适合OLTP和OLAP，即HTAP
4. FSM的构成：
   1. **tile tuple**: 最基础的单元。为 **一个元组若干个属性的集合**。
   2. **physical tile**: a set of ***tile tuples***
   3. **tile group**: a set of ***physical tiles***
   4. table: a set of ***tile group*** 由Storage Manager管理

5. FSM保存元组的过程
   - 当元组存储在FSM的数据库中时，它的存储结构会随着时间而改变。
   - 一开始所有新tuples进来时，都是以行存的形式
   - 随着这些tuples逐渐变为历史数据，会重新组织为OLAP友好型的垂直划分
   - （re-organization过程）先把这些元组**复制**到新结构的***tile group***中，在把表中原来的的***tile group***与***新的tile group***进行交换——整个过程在后台运行，且是事务安全的