## JIT-Compiling SQL Queries in PostgreSQL Using LLVM

> 来源于：2017 PGCon
>
> 可参考https://llvm.org/devmtg/2016-09/slides/Melnik-PostgreSQLLLVM.pdf

1. Motivation: 

   在CPU计算密集的场景中，比如对对大批量的数据进行filter，则实际在内核执行时 每行元组都要进行一次表达式的函数调用（要进行interpret）。

   比如这个例子：`SELECT COUNT(*) FROM t1 WHERE (x+y)>20`，采用解释的方式执行，则解释的开销占到了总执行时间的56%；若使用LLVM生成代码，则该部分开销只会占6%。核心思想就是减少函数的反复调用，而是通过LLVM把这部分反复调用的代码在一个函数内进行实现，从而完成运算。

2. 主要针对 一些复杂的分析性查询，其瓶颈不在磁盘IO 反而在CPU上。

3. 解释执行的示例：

   以filter `X+Y < 1`为例 

   ![](https://cchw-1257198376.cos.ap-chengdu.myqcloud.com/test/clipboard_20210217_121809.png)

   其对应的函数调用代码为

   ```c
   P = indirect call ExecEvalFunc() {
       X = indirect call ExecEvalFunc();
       Y = indirect call ExecEvalFunc();
       return int8pl(X, Y)
   };
   C = indirect call ExecEvalConst(1);
   return int8lt(P, C)
   ```

   每次的indirect call就是get_next()虚函数，而JIT可以通过内联来避免函数调用。

4. JIT执行的流程

   \*.c文件（pg的backend目录下）通过**clang**生成\*.bc的LLVM中间代码文件，在通过**llvm-link**生成backend.bc，在经过**opt**阶段 来防止全局状态变量的重复。PostgreSQL服务启动时，将中间代码装载进buffer中；JIT阶段再将LLVM模块编译为nayive code来执行。

5. 执行模型的变化：

   从原先volcano的pull-based，变为push-based的模型，从底层结点向上推送元组。

6. 

