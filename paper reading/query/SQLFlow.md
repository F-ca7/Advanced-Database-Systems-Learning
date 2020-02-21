## SQLFlow: A Bridge between SQL and Machine Learning

1. **场景**：工业中的AI系统通常是 end-to-end 的机器学习工作流。这里的流程是包括：从web UI开始，再是日志收集器 log collector (如 [Fluentd](https://www.fluentd.org/))，日志流聚集器 log streams aggregator (如 Kafka)，接着到 数据库，最后是机器学习系统 (如TensorFlow, PyTorch等)。

   注意区分 **end-to-end解决方案** 与 **end-to-end学习**

   > Generally, end-to-end solutions are used with vendors that offer comprehensive systems that keep pace with a business’s ever-changing infrastructure requirements, and the changing demands of the IT sector itself. End-to-end suppliers generally handle all of a system's hardware and software, including installation, implementation, and maintenance. An end-to-end solution might cover everything from the client interface to data storage.
   >
   > end-to-end学习，其实就是从一端（输入，原始数据）到另一端（输出，结果）的意思。也就是说，端到端意味着，模型的输入是原始的数据，模型的输出是我们想要的结果。通过缩减人工预处理和后序处理，尽可能是模型从原始输入到最终输出，**使模型有更多根据数据 自动调节的空间，增加模型的整体契合度**。

   目前有些数据库系统 扩展了自己的dialect 来支持机器学习，比如谷歌的BigQuery 增加了 CREATE MODEL语句 用来训练模型。但这些存在缺点如下：

   - 把 end-to-end ML 用户固定在特定的数据库系统上，如果要切换数据库，需要重写整个ML方案
   - 大部分不开源，难有社区贡献
   - 都基于现存 数据库的计算基础结构，无法使用向 Kubernetes 的集群计算技术做到容错和可扩展

2. **Contribution**:

   - SQLFlow 使用sql语言（而不是Python之类的语言，这样适合所有编程人员），高效地描述了 这样的机器学习流程
   - SQLFlow可以桥接多种数据库系统（Mysql, Hive, Alibaba MaxCompute） 和 多种机器学习引擎（TensorFlow, XGBoost, sklearn）
   - 发明了一种 collaborative parsing算法
   - SQLFlow 将一个sql程序 编译为 Kubernetes原生的工作流，以方便容错处理 与云端部署

3. **实现细节**：

   - **Collaborative Parsing**

     可以使用任何开源的parser，如MySQL/TiDB, HiveQL, Clacite Parser. 使用goyacc编写，只能理解 TO TRAIN, PREDICT, EXPLAIN从句。

   - **Two-Tier Compilation**

     SQLFlow表现为 两层编译的架构。上层将 SQL程序 翻译为 YAML，其中每条SQL语句为一个step。而当每个step在容器中执行时，将SQL翻译为一个Python程序并运行。

     如果是普通SQL语句，生成的Python程序会 将其直接提交到数据库系统中；如果是经过SQLFolw扩展的语句，则生成的Python程序会 获取数据、提取特征、启动ML任务。