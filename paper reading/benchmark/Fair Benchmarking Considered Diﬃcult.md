## Fair Benchmarking Considered Difficult: Common Pitfalls In Database Performance Testing

1. **Motivation**: 性能测试表面上看上去很客观公正，但事实上有 许多不同的方法可以让基准测试结果 在一个系统上比另一个更优（可能是有意的，也可能是无意的）。

2. **Contribution**: 

   - 对于软件系统可再现的基准测试方法 进行了研究
   - 基于以上研究的方法，探索了如何将其应用到数据库测试场景，并且讨论了在数据库测试中通常可能犯的问题
   - 讨论了如何 辨别并避免以上问题，从而可以在不同的系统、算法间进行公平的比较

3. 相关工作：

   - 通用Benchmarking

     通常实验过程包括这几个步骤：实验计划与建立、实验执行、数据收集、数据分析和结果表示。而在统计型数据分析中，常见的问题包括：“精心挑选”的数据集、隐藏一些极端情况。

     在计算机科学领域，*The Art of Computer Systems Performance Analysis: Techniques for Experimental Design, Measurement, Simulation and Modeling* 一书中给出了这么两个定义

     - Mistake: 对于Benchmark执行不利的选择，但是是无意的。 比如，测试workload不能很好地反映真实生产环境的情况。
     - Game: 故意操作实验过程 来得到一个特定的结果。比如，实验时的软硬件环境不同。

     而在高性能计算(HPC)领域，*Scientific Benchmarking of Parallel Computing Systems (2015)* 提出了12条Benchmark测试规则。其作者也强调了 实验结果的可解释性；因为这样才提供了足够的信息，让学者能够理解实验，得出自己的结论，评估其确定性，从而对结果进行归纳。

     在 系统安全领域， *Benchmarking Crimes: An Emerging Threat in Systems Security (2018)*指出了Benchmark中的错误会威胁到实验结果的有效性，一定程度阻碍研究的发展。

   - 数据库Benchmarking

     *Benchmark Handbook: For Database and Transaction Processing Systems (1992)* 提出了一些测试时的要点：

     - 数据应以指定的方式生成
     - 结果报告应包括资源利用率和数据库磁盘空间
     - 准确的硬软件配置
     - 有效的查询计划
     - 收集统计信息的开销
     - 随时间变化的内存使用情况
     - 运行时间、CPU时间、IO操作次数

     现今的TPC测试则成为了新标准，参考TPC Benchmark C Standard Specification、TPC Benchmark H (Decision Support) Standard Specification。而对于特定的数据库，比如SQLite， *The Dangers and Complexities of SQLite Benchmarking (2017)* 中也指出 通过一个参数的设置，事务吞吐量的变化可达到28倍，而相关的16篇论文没有一篇报告了它们实验结果的所需参数。

4. ***Checklist***
   - Benchmark选择
     - 能覆盖整个评估空间
     - 证明选取Benchmark子集的合理性
     - Benchmark能够突出相应评估的功能
   - 可重现
     - 硬件配置
     - DBMS参数与版本
     - 源代码 / 二进制文件
     - 数据、表模式、查询
   - 优化
     - 编译选项
     - 系统参数
   - 可比较的调优
     - 不同的数据
     - 多种workload
   - 程序预热（Cold / Warm / Hot runs）
     - 区分冷运行和热运行
     - 冷运行：清空OS和CPU的缓存
     - 热运行：初始的运行结果直接忽略掉
   - 预处理
     - 确保系统之间的预处理相同
     - 注意自动的索引创建
   - 保证正确性
     - 验证结果
     - 测试不同的数据集
     - 边界情况的讨论
   - 收集结果
     - 多运行几次防止其他干扰
     - 检查多次运行结果的方差/标准差
     - 报告具有稳健性的指标（如中位数、置信区间）