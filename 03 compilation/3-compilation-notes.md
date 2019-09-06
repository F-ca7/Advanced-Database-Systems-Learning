## Query Compilation

1. in-memory database(内存数据库)：相比于传统的寄语磁盘的DBMS，IMDB要快很多
   - 使用IMDB后，唯一增加throughput的方法就是**减少执行的指令数**(比如想增加10倍的吞吐量，DBMS需要减少90%的指令)
2. **code specialization**: 生成的代码对应DBMS的一个特定任务。生成代码的方法有两种：
   - Transpilation: 先把关系型查询计划转化为C/C++ 代码，然后用传统的编译器来生成本地代码
   - JIT Compilation：针对查询生成**Intermediate Representation** (中间语言)，IR可以 (通过即时编译器) 快速编译生成本地代码
3. 查询处理：
   - tuple-at-a-time: 每个操作符通过调用子结点的***next()***方法来获取下一个要处理的**tuple**
   - operator-at-a-time: 每个操作符会把它所有的输出结果物化(materialiation)，供它的父结点操作符使用
   - vector-at-a-time: 每个操作符通过调用子结点的***next()***方法来获取下一个要处理的**数据块**
4. 流程一览：
   - **SQL查询**经过***Parser***，得到**AST**(抽象语法树)
   - **AST**传到***Binder***，***Binder***与***System catalog***(元数据)交互，处理后得到**Annotated AST**(标注语法树)
   - **Annotated AST**发送至***Optimizer***(优化器)，***Optimizer***通过***System catalog***的信息 计算估计代价，得到**Physical plan**(物理执行计划)
   - **physical plan**发送至**Compiler**，得到**Native code**

5. 关系型操作符很方便用来解释查询过程，但执行起来不是最高效的。
6. 使用**LLVM**(底层虚拟机)将内存中的queries编译为本地代码
   - LLVM是一个模块化和可重复使用的编译器和工具技术的集合
   - 能够进行程序语言的编译期优化、链接优化、在线编译优化、代码生成
   - LLVM的编译时间随着<u>query的大小</u>(join/predicate/aggregation的数量) 超线性增长—— 对于OLTP应用无所谓，对于OLAP是大问题。

7. Adaptive Execution 
   - 生成query的LLVM中间语言
   - 在解释器中执行中间语言
   - 在后台对query进行编译(编译耗时相对较长)
   - 编译完成后，从解释执行过程无缝切换到编译后本地代码的执行

8. 查询的编译很有用，但是实现起来不容易
9. MemSQL2016版的query编译实现最好。——任何新的DBMS想有竞争力，必须实现query编译。
10. 