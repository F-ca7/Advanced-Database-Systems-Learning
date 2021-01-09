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
   - 对于带参的prepared stmt，pg采用如下策略（在[德哥的博客](https://github.com/digoal/blog/blob/master/201606/20160617_01.md)中，指出了这样做可以避免plan cache的倾斜）：
     - 前五次执行时，每次会根据输入的参数来生成执行计划称作custom plan，然后记录下**总的custom plan的代价与数量**。
     - 第六次开始执行时，生成一个不依赖参数的执行计划并保存起来——称作generic plan。如果generic plan的代价小于之前所有custom plan的平均代价的1.1倍，则采用generic plan；否则重新根据参数生成新的custom plan，更新总的custom plan的代价与数量。

   在Mysql中，同样也只有[prepared stmt的cache](https://dev.mysql.com/doc/refman/8.0/en/statement-caching.html)（以及存储过程），既支持SQL语句，也支持C/S二进制协议。由`max_prepared_stmt_count`系统变量来控制cache的语句数量。

   对于TiDB的[执行计划缓存](https://docs.pingcap.com/zh/tidb/stable/sql-prepare-plan-cache)，其支持prepared请求与execute请求（session级别），通过LRU链表实现缓存。这里其无法应对参数取值变化情况(如下)，所以TiDB的plan cache适用于 查询编译耗时占比较高且执行计划不容易变的业务场景。（2020-10-22 prepared plan cache选项默认开启）

   > 比如查询过滤条件为 `where a < ?`，假如第一次执行 `Execute` 时用的参数值为 1，此时优化器生成最优的 `IndexScan` 执行计划放入缓存，在后续执行 `Exeucte` 时参数变为 10000，此时 `TableScan` 可能才是更优执行计划，但由于执行计划缓存，执行时还是会使用先前生成的 `IndexScan`

   阿里云的PolarDB-X，其[执行计划管理](https://help.aliyun.com/document_detail/144299.html)也有相应文档。（默认开启plan cache，且这里将SPM与Plan Cache整合在了一起）对于到来的SQL查询，会对SQL做参数化处理，即把常量参数用`?`占位符替换。注意：这里PolarDB-X的**plan cache**和SPM都是用参数化后的SQL语句作为Key（而不是原本的语句）。

   - 与Mysql和TiDB不同，plan cache会缓存所有SQL计划
   - SPM仅对复杂查询SQL进行处理（可能认为这一类查询退化的可能性更大）；<u>如果是简单查询，不会进入SPM这一阶段</u>
   - 在其SPM中，每个SQL对应的一个baseline中包含一个或多个执行计划。实际使用中，会根据当时的参数选择其中代价最小的执行计划来执行。也就是类似于 PQO（参数化查询优化）问题


------

### Aurora PG的[Query Plan Management](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Optimize.html)

1. **Motivation**：迁移到Aurora postgreSQL的企业用户需要在系统升级或更改时 能有**稳定的性能**。

2. 改变查询执行计划的因素（通常不在预料内）：

   - 优化器统计信息的改变
   - planner配置参数的更新
   - schema的改变，比如 增减了索引
   - 绑定变量的改变
   - 数据库版本的更新

3. QPM的目标：

   - **计划稳定性**：当系统发生如上述改变时，防止计划的退化
   - **计划自适应性**：自动检测新的最低代价计划，自动控制新计划何时能够被使用 

4. 工作流程：

   ![](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2019/05/31/intro-to-aurora-1.gif)

   1. 还是先由优化器生成SQL对应的最低代价计划P0
   2. 如果没启用QPM，则直接执行P0；若启用了，则进入下一步
   3. 先判断是否属于 受管理的SQL语句，不属于则直接执行P0
   4. 进入计划捕获阶段
   5. 如果QPM未开启 `use_plan_baselines ` 选项，则执行P0
   6. 如果P0 不属于存储中的计划，优化器会捕获并保存P0 并且 标记为 unapproved 状态，进入第7步；如果属于（且为approved状态），进入第8步
   7. 如果P0不是禁用状态，直接执行P0（用户可以手动设置 **直接执行**任何估计成本低于指定阈值的unapproved计划）；如果是的，进入下一步
   8. 如果这个受管理的SQL语句有对应 已启用且有效的首选计划（**preferred**标记，可能有多个首选计划，会重新估计每个首选计划的代价；注意**preferred**和**approved**是不同的 ），然后优化器会执行最低代价的计划；如果没有首选计划，进入下一步
   9. 如果不存在或无法重新创建首选计划，则优化器会重新计算每个approved计划的代价，然后选择代价最低的approved计划。如果既没有preferred计划，又没有approved计划，那就只能跑P0了

   整体看来，流程和Oracle的SPM基本类似。

5. 受管理的语句 (Managed Statement)

   类似于上文polardb-x中提到的参数化语句，这里同样会对SQL语句进行规格化：

   - 去除头部注释块
   - 去除EXPLAIN相关
   - 去除尾部多余空格
   - 去除字面量
   - 保留token之间的空格 以及原本大小写，方便阅读

   比如，语句`/*Leading comment*/ EXPLAIN SELECT /* Query 1 */ * FROM t WHERE x > 7 AND y = 1;` 会被规格化为 `SELECT /* Query 1 */ * FROM t WHERE x > CONST AND y = CONST; `，最后根据规格化后的语句进行Hash操作。

6. [最佳实践](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Optimize.BestPractice.html)

   **Proactive**：在验证了新计划确实执行更快后，可以手动置为approved状态，从而主动的防止性能退化。实践如下：

   1. 在**开发环境中**，确定对性能或系统吞吐量影响最大的SQL语句，手动捕获特定SQL语句的计划
   2. 从开发环境**导出捕获的计划**，并将其**导入到生产环境**中
   3. 在生产中，运行应用程序并强制使用approved的计划
   4. 对优化器生成的unapproved计划进行分析，**主动将性能更好的计划设置为approved状态**

   **Reactive**：监视运行中的应用，从而检测导致性能下降的计划；检测到时，可以手动**reject掉**或**修复**错误的计划。

   1. 在应用程序运行时，强制使用受管理的计划，并自动将优化器新发现的计划添加为unapproved计划
   2. 监控运行中程序可能发生的性能退化
   3. 发现计划性能退化时，将该计划设置为reject状态。下次优化器运行SQL语句时，会自动忽略reject的计划，并使用其他approved计划
   4. 如果想对退化的计划进行**修复**，则可以考虑使用pg_hint_plan 扩展来尝试改进计划