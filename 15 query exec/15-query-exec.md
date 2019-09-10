## Query Execution & Processing

1. 当不考虑磁盘时，(即in memory)，DBMS查询可优化性能的地方还有哪些:
   - 减少指令数
   - 减少每条指令的时钟周期(增加cache命中，减少系统的stalls)
   - 并行化(对于每个查询使用多线程)

2. CPU(可参考计算机组织与体系结构):
   - CPU将指令组织为流水线(pipeline stages)
   - Super-scalar(大型?) CPU支持多流水线: 在一个时钟周期内，如果多条指令之间相互独立，则可它们并行执行。
   - DBMS和CPU面临的相同问题：1. 如果一条指令依赖另一条指令，那么它不能立即进入同一条流水线 2. 分支预测：在DBMS中最经常执行的分支代码是**顺序扫描时的过滤操作**(比如 where price>500)，但几乎不可能可以准确预测

3. 分支预测：对于下面这个SQL有两种代码实现

   ```
   SELECT * FROM table
   WHERE key >= $(low)
   	AND key <= $(high)
   ```

   - 分支型：
   
   ```
   i = 0
   for t in table:
   	key = t.key
   	if (key>=low) && (key<=high):
   		// 把符合条件的元组放在输出的数组中
   		copy(t, output[i])
   		i = i + 1
   ```
   
   - 非分支型：
   
   ```
   i = 0
   for t in table:
   	// 不管符不符合条件
   	// 先直接把元组放在输出的数组中
   	copy(t, output[i])
   	key = t.key
   	// 这里原ppt的代码可能有误
   	m = (key>=low && k<=high)? 1:0
   	// 符合条件就向数组下个位置移动
   	// 不符合不动，等待被覆盖
   	i = i + m
   ```
   
   经测试对比可发现，非分支型的所花费的平均CPU时钟周期比分支型**少**。

4. DBMS在对一个值进行操作前，需要检查其数据类型：通过很长的switch语句实现，也带来了更多难以预测的分支
5. **processing model**：决定了DBMS如何执行一个查询计划(对于不同类型有不同的trade-offs)
   - Iterator model: (又称为Pipeline model)
     - 每个查询操作符(如选择，连接，投影等)对应有一个**next()**方法
     - 每次调用时，返回**一个(single)元组**或空标记(null marker, 说明没有更多的了)
     - 查询操作符在一个for循环里，调用其子结点的next()方法，来其emit()的元组，然后对获取到的元组进行处理
     - 一些操作符(连接，子查询，order by)可能会**阻塞**直到子结点输出**所有的元组**
   - Materialization model:
     - 每个查询操作符**一次性**处理其输入，并**一次性**发送所有输出
     - 操作符将其输出作为一个结果**物化(materialize)**
     - DBMS会将一些hint下推/下移，避免扫描过多元组(比如有比较严格的选择时，优先执行)
     - 输出结果可以是 whole tuples(NSM) 或 subsets of columns(DSM)——参考([Flexible-Storage-Model](https://github.com/F-ca7/Advanced-Database-Systems-Learning/blob/master/paper%20reading/Flexible-Storage-Model.md))
     - 适合OLTP的工作(一次只会查询少量的元组，更少的函数调用)，不适合OLAP查询(产生过大的中间表/中间结果)
   - Vectorized/Batch model:
     - 每个查询操作符对应有一个**next()**方法
     - 每次发送**一批(batch)元组**，而不是单个
     - 一批的大小可由硬件决定，或查询的性质决定
     - 适合OLAP查询(比起*Iterator*减少了每个操作符的函数调用 )

6. Plan Processing Direction: 
   - 自顶向下：
     - 从根结点开始，从子结点拉取(pull)数据
     - 元组通过函数调用传递
   - 自底向上：
     - 从叶结点开始向父结点推送(push)数据

7. 查询内的并行：两种技术之间不互斥
   - 操作符内(Horizontal)：比如一个选择操作符，可以根据数据的不同子集，分解(decompose)为多个独立的选择操作符，最后在通过**exchange**操作合并选择结果。 
   - 操作符间(Vertical)：在流式处理系统中更常见，又叫**流水线并行**(pipelined parallelism)

