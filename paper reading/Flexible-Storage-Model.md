## Bridging the Archipelago between Row-Stores and Column-Stores for Hybrid Workloads

1. 当DBMS既要适合OLTP，又要适合OLAP，称为HTAP(hybrid transactional-analytical processing)
   - 
2. 使用一个统一的架构来连接OLAP与OLTP之间的鸿沟——基于DBMS未来访问元组的方式，使用**混合结构(hybrid layout)**来存储数据表:
   - hot tuple(热点元组，最近时间内经常会发生变化): 以针对OLTP操作优化的方式存储
   - cold tuple: 以更方便OLAP查询的方式存储

3. 