## FoundationDB

参考：

1. FoundationDB: A Distributed Unbundled Transactional Key Value Store
2. FoundationDB Record Layer: A Multi-Tenant Structured Datastore
3. https://apple.github.io/foundationdb/index.html

FoundationDB是苹果内部使用的分布式数据库，主要特性包括：

- 多模：可以存储各种格式类型的数据
- 易扩展：部署、管理、扩容方便
- 高性能：商业化的产品级硬件即可支持高负载
- 弹性应用架构：应用即可以DB直接交互，也可以与上层的无状态层进行交互（提供更多特性）

FDB本质是一个分布式、有序的KV存储，并在此之上提供了基于乐观并发控制的事务特性（NoSQL的扩展性 + ACID -> NewSQL）。系统逻辑上由两个集群组成（两层均可独立地扩展）：

- 数据层(Data Plane): 负责存储数据、处理事务
- 控制层(Control Plane): 作为集群的协调者，包含了元数据管理、集群监控、与事务号分配等工作

> FDB 的模拟测试框架是它的特别之处，可以做到在单线程的条件下 快速模拟集群的各种故障，所以能保证系统是确定性的(deterministic)，也就是所有问题都是可复现的（在同样输入的情况下）；从而保证了在快速发布新版本、引入新特性时，系统仍能保持稳定。

一个产品化的数据库需要考虑什么问题？

- 数据持久性
- 数据分片
- 负载均衡
- 集群管理
- 宕机检测与恢复
- 副本分布
- 集群同步
- 过载状态下的运行
- 横向扩展
- 并发与任务调度
- 系统部署与升级、配置管理

为了从更高层次来阐述这些问题的解决方案，得先了解FDB架构的设计理念：

- Divide-and-Conquer: 将 事务管理(写入路径) 与 分布式存储(读取路径)分离开来，并且二者可以单独扩展，还有其他不同的组件负责不同的任务，如时间戳管理、冲突检测、日志记录等
- Make failure a common case: **在分布式系统中，故障认为是常态而不是一个小概率的异常**。在FDB的事务管理中，并不是尝试解决所有的失败场景，而是主动地进行重启恢复
- Fail fast and recover fast: 为了提升可用性，FDB尽可能地降低MTTR（平均恢复时间），也就是上一点所提到的重启恢复【论文中提到，生产环境下的恢复用时少于5s】
- Simulation testing: 引入了随机化、确定性的测试框架，不仅更容易发现深层次的bug，也能改进代码质量和开发效率

------

### 事务

MVCC读 + 乐观并发写，可序列化的事务



-----------

[未解决的一些问题](https://apple.github.io/foundationdb/known-limitations.html)：

- 大事务：
- 长事务：目前不支持执行时间超过5s的长事务，如果需要长事务，建议在应用层进行乐观验证；如果需要长读事务，建议在上层维护快照版本；
- 