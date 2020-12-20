## SQL Plan Management

### Oracle中的SPM

1. SPM的三个核心：
   
   - **Plan capture**
   
     创建SQL计划的baseline，保存在**SQL Management Base**(SMB)中
   
   - **Plan selection**
   
     保证有baseline的SQL查询只会使用已经Accepted的执行计划，优化器找到的新执行计划会暂时保存会Unaccepted状态
   
   - **Plan evolution**
   
     对于一条SQL，评估所有Unaccepted状态的执行计划（根据文档，该评估过程在夜间执行，因为本质是要把这些计划跑一边），如果验证为有性能提升，则把对应计划标记为Accepted状态
   
2. 使用SPM的查询流程（在Oracle11g中，本质是一种保守的计划选择策略）可参考[Oracle白皮书](https://www.oracle.com/technetwork/database/bi-datawarehousing/twp-sql-plan-mgmt-12c-1963237.pdf)：

   1. SQL语句先编译(hard parse)成计划，然后通过CBO优化选出最低代价的执行计划P0

   2. 如果该SQL语句存在计划baseline，优化器则把P0与baseline中的计划对比，进入3

   3. 如果 找到对应的计划P1 且**标志位为Accepted状态**，则使用P1；如果baseline中没有P0对应的计划，则对比baseline中所有Accepted状态的计划，选出最低代价的计划P2，进入4

   4. 如果P0的代价比P2低，则P0会被添加到baseline中，且状态为**Not-Accepted**。此时执行计划仍然选择的是P2，[因为P0在没有被验证不会造成性能退化之前，是不会被真正使用的](https://blogs.oracle.com/optimizer/sql-plan-management-part-2-of-4-spm-aware-optimizer)（当验证成功P0更优时，则进行演化 **Evolve**）

   5. 如果整个数据库系统发生较大的变化 且影响到目前所有Accepted状态的计划，则这些计划不可再利用（无法再复现），此时会选择优化器原本生成的最低代价计划

      数据库系统可能发生的变化（都可能导致相同的SQL语句生成新的执行计划，<u>尽管大部分新计划的性能是会有改进的，但仍有一些会发生性能上的退化</u>）：

      - 手动/自动地更新了统计信息
      - 增加/删除了字段的索引
      - 修改了优化器相关的参数
      - 数据库系统的升级

3. SQL profile(10g) 和 SQL plan baseline(11g)的区别

   - SQL profile通常由SQL Tuning Advisor生成，包含了一些辅助信息，来最小化优化器的误差(主要是基数估计的误差)，从而更大可能选择到最优计划，而不是强迫优化器去选哪个特定的计划 或 从哪些特定的计划中选择。但**无法保证计划的稳定性**
   - SQL plan baseline是由一系列的accepted plans组成集合。当查询语句被解析后，会从这个集合里选出最优计划；如果在CBO阶段找到了不在集合中 且代价更低的计划，优化器会把它添加到not-accepted状态的计划历史中，但该计划暂时不会被使用（直到evolve过程）
   - 所以，当希望系统能快速根据新的统计信息做出调整时，就可以用SQL profile；当你需要更保守的计划执行，就可以使用SQL plan baseline

4. Plan cache 和 SPM的区别

   Plan cache顾名思义是执行计划的缓存。本质就是相同的SQL语句采用的执行计划给缓存起来，跳过解析、优化的步骤。

   比如对于Oracle，Sql语句的parsing分为 <u>硬解析(hard parse)</u> 和 软解析(soft parse)，还有<u>soft soft parse</u>。软解析 即通过Sql语句hash 从cache中使用对应执行计划。

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20201211_104035.png)

   对于PostgreSQL，pg只提供prepared statement的cache（session级别）。（pg对于存储过程也实现了plan cache，PL/sql的解释器会把SQL语句以prepared statement的形式存储）

   - 对于无参数的prepared stmt，第一次执行时会生成计划 并 保存到cache中，以后在执行到这个stmt会重用执行计划
   - 对于带参的prepared stmt，pg采用如下策略（在[德哥的博客](https://github.com/digoal/blog/blob/master/201606/20160617_01.md)中，指出了遮掩顾总可以避免plan cache的倾斜）：
     - 前五次执行时，每次会根据输入的参数来生成执行计划称作custom plan，然后记录下**总的custom plan的代价与数量**。
     - 第六次开始执行时，生成一个不依赖参数的执行计划并保存起来——称作generic plan。如果generic plan的代价小于之前所有custom plan的平均代价的1.1倍，则采用generic plan；否则重新根据参数生成新的custom plan，更新总的custom plan的代价与数量。

   在Mysql中，同样也只有[prepared stmt的cache](https://dev.mysql.com/doc/refman/8.0/en/statement-caching.html)（以及存储过程），既支持SQL语句，也支持C/S二进制协议。由`max_prepared_stmt_count`系统变量来控制cache的语句数量。

   对于TiDB的[执行计划缓存](https://docs.pingcap.com/zh/tidb/stable/sql-prepare-plan-cache)，其支持prepared请求与execute请求（session级别），通过LRU链表实现缓存。这里其无法应对参数取值变化情况(如下)，所以TiDB的plan cache适用于 查询编译耗时占比较高且执行计划不容易变的业务场景。（2020-10-22 prepared plan cache选项默认开启）

   > 比如查询过滤条件为 `where a < ?`，假如第一次执行 `Execute` 时用的参数值为 1，此时优化器生成最优的 `IndexScan` 执行计划放入缓存，在后续执行 `Exeucte` 时参数变为 10000，此时 `TableScan` 可能才是更优执行计划，但由于执行计划缓存，执行时还是会使用先前生成的 `IndexScan`

   阿里云的PolarDB-X，其[执行计划管理](https://help.aliyun.com/document_detail/144299.html)也有相应文档。（默认开启plan cache）对于到来的SQL查询，会对SQL做参数化处理，即把常量参数用`?`占位符替换。注意：这里PolarDB-X的**plan cache**和SPM都是用参数化后的SQL语句作为Key（而不是原本的语句）。

   - 与Mysql和TiDB不同，plan cache会缓存所有SQL计划
   - SPM仅对复杂查询SQL进行处理（可能认为这一类查询退化的可能性更大）；如果是简单查询，不会进入SPM这一阶段
   - 在其SPM中，每个SQL对应的一个baseline中包含一个或多个执行计划。实际使用中，会根据当时的参数选择其中代价最小的执行计划来执行。也就是类似于 PQO（参数化查询优化）问题

5. 