## Benchmarking Query Execution Robustness

1. 作者认为 robustness可以分三类：
   - **optimizer** robustness: 当预期条件变化时，优化器仍能选出好的执行计划
   - **execution** robustness: 对于同一个给定的计划，执行引擎可以在不同的运行时场景下 都能高效地处理
   - **workload management** robustness: 当查询实际性能比预期低时，整个系统受影响的程度
2. 

