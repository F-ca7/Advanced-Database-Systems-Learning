## Survey on NoSQL Database

1. **问题**：在云计算和大数据时代，传统关系型数据库面临的挑战：
   - 低延时的高并发读写
   - 高效的大数据存储
   - 高可扩展性和高可用性
   - 更低的管理与运维费用

2. NoSQL中的**Data Model**
   - **Key-Value**: 结构简单，比关系型数据库更快，支持大容量的存储与高并发
   - **Column-oriented**: 只用访问query查询的列，减少了系统I/O；相同的数据类型，压缩比率更高
   - **Document**: 和KV型很相似，通常是JSON或XML格式；而且支持二级索引 来方便上层应用的存取

3. 依据CAP理论的数据库分类

   - CA: 

     通过**Replication**来保证数据的一致性与可用性，如 Vertica(Column-oriented)

   - CP:

     数据存储在分布式的结点中，并且确保这些节点数据的一致性，如BigTable, HyperTable, HBase(Column-oriented), MongoDB(Document), MemcacheDB, Berkeley DB(Key-value)

   - AP:

     有 CouchDB(Document), Tokyo Cabinet(Key-value)

