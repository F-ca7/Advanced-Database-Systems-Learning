## Lakehouse Technology as the Future of Data Warehouse

> 第37届CCF数据库学术会议，Matei Zaharia

1. 现今，数据存在的最大挑战：

   - Quality
   - Staleness

   导致数据分析师大部分时间花在 理解数据上

2. 发展历程

   - 1980s，发展出了Data Warehouse。从数据库系统中 ETL 得到数据，最红用于 BI 或者报表
   - 2010s，发展出了 Data lake。用低成本的存储(S3，HDFS)来保存raw data(结构化/半结构化/非结构化)，通过ETL/ELT 将数据装载到 Warehouse中

3. 数据湖存在的问题：**系统架构更复杂**。

   - 多种不同语义、不同sql dialect的存储系统
   - ETL步骤可能带来问题

4. **Lakehouse**：新提出的技术，metadata层 + 新的查询引擎 + 面向ML优化的数据访问

5. **metadata层**

   数据湖通常就是 一系列文件的集合。所以元数据层会记录哪些文件是表的一部分，提供更丰富的数据管理功能

   参考现有系统：Delta lake, Iceberg, HIVE

   **去读Delta lake的论文**

6. **查询引擎**

   性能优化：

   - Caching：针对热数据。比如通过SSD或memory进行cache
   - 辅助数据结构：统计数据、索引。比如通过min/max信息，可以跳过一些文件数据
   - data layout：尽量减小IO。比如Z-ordering，针对multi-dimension的访问
   - 向量化执行引擎：发挥现代CPU的优势。

7. 面向ML优化的数据访问

   和sql workload不同，机器学习的workload需要通过非SQL代码（Tensorflow、XGBoost）来处理大量数据；而且基于JDBC的SQL接口 在这种数量规模上太慢了。

   Lakehouse则对ML提供了以下特性：

   - Use time travel来实现版本控制，从而也可以进行可重复的实验
   - 使用事务来对表进行可靠的更新
   - 通过流式IO，来获取到最新数据

8. Q&A

   - 使用GPU，或使用Ethernet带来更高带宽
   - 列式存储，smaller object带来的好处——读取每个文件的cost，可以参考Uber的机器学习框架

