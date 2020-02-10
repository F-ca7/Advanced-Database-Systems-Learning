## Storage Model & Data layout

1. 不同类型数据的表示：

   - INTEGER / BIGINT / SMALLINT / TINYINT

     使用C/C++中的表示形式

   - FLOAT / REAL(不能存储精确数值) vs. NUMERIC / DECIMAL(存储精确数值)

     前者为IEEE-754标准，后者为[定点十进制数 fixed-point decimal](http://www-inst.eecs.berkeley.edu/~cs61c/sp06/handout/fixedpt.html) 如Postgres中的实现：

     ```c
     typedef unsigned char NumericDigit;
     typedef struct {
         int ndigits;	// 数字的数量
         int weight;		// 首位的权
         int scale; 		// 缩放系数
         int sign;		// 正/负/NaN
         NumericDigit *digits;	// 实际数字的存储
     } numeric;
     ```

   - TIME / DATE / TIMESTAMP

     32/64位的整形，Unix epoch (从1970年1月1日（UTC/GMT的午夜）开始所经过的毫秒/微秒数)

   - VARCHAR / VARBINARY / TEXT / BLOB

     指针指向数据，头部带有长度和下一段的地址 以及数据类型

     ![](https://s2.ax1x.com/2020/02/10/14I7QS.png)

2. NULL类型表示：

   - 特殊值

   - Null Column Bitmap Header：

     在元组头部存 一个bitmap，来表示该元组哪些属性位null

   - Per Attribute Null Flag

     会浪费空间，因为每个属性多一个bit 需要重新字对齐

3. 字对齐的元组：(与结构体的字节对齐类似)

   - padding: 在属性的后面补充空的bits

     ![](https://s2.ax1x.com/2020/02/10/15iFmj.md.png)

   - reordering: 重新排序，最后可能也需要padding

   两种方法的性能对比：(插入测试)

   | 对齐方式   | 平均吞吐量   |
   | ---------- | ------------ |
   | 无对齐     | 0.523 MB/sec |
   | padding    | 11.7 MB/sec  |
   | reordering | 814.8 MB/sec |

4. 存储模型: 可参考 **[Flexible-Storage-Model](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/engine/Flexible-Storage-Model.md)**

   - N-ary Storage Model (NSM)

     物理存储的选择：

     1. **Heap-Organized Tables**

        元组存储在 叫做**堆**的blocks中，不需要有序

     2. **Index-Organized Tables**

        元组存储在自己的主键索引中，和聚集索引不相同。

     优点：

     - 插入、更新、删除快（适合OLTP）
     - 适合查询整个元组
     - 可以使用面向索引的存储

     缺点：

     ​	- 不适合扫描表的大部分 以及 只查询一部分属性

   - Decomposition Storage Model (DSM)

     优点：

     - 只读取需要的数据，减少无用的工作
     - 压缩更好

     缺点：

     - 插入、更新、删除慢，因为元组需要分裂或者拼接

   - Hybrid Storage Model

     两种选择：

     - Separate Execution Engines: 对NSM和DSM 分别使用最合适的执行引擎。返回结果时需要合并两个引擎的结果，从而对外表现在逻辑上是一个数据库。如果一个事务跨执行引擎，需要引入同步机制。

       如何确定哪些数据存入DSM中？

       - Manual: 由DBA指定哪个表存入DSM中
       - Off-line: DBMS离线监控访问日志，然后决定哪些数据移入DSM
       - On-line: DBMS在运行期监控数据库访问，然后决定哪些数据移入DSM

     - Single, Flexible Architecture: 只用单一的执行引擎，同时适合NSM和DSM

5. 表模式的更改 Schema Changes: 

   - 添加列

     - NSM: Copy tuples into new region of memory
     - DSM: 直接创建一个新列

   - 删除列

     - NSM 1: Copy tuples into new region of memory
     - NSM 2: 标记列为 deprecated，后期再清理
     - DSM: 直接删除列 释放内存

   - 更改列

     检查改变是否合法

6. 索引操作

   - 创建索引

     - 扫描整个表、填充索引

     - 当一个事务在建立索引时，另一个事务需要记录修改表的裱花
     - 当扫描完成后，锁住表；整合扫描过程中表的变更

   - 删除索引

     - 从catalog中直接删除索引（逻辑上删除）
     - 只有当删除该索引的事务 commit后才变得不可见

7. 

