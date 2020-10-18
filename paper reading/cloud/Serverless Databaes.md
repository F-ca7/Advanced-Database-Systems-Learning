## Serverless Databaes

1. 分布式数据库 + 云计算资源管理（k8s这一套）

   软件系统：

   - 分布式数据库（OpenGauss还不是真正的分布式数据库，目前只是主从）

   - 云存储（Share disk，有状态的东西放在云存储；使用的是**Ceph**分布式存储集群）

     共享存储的一致性怎么保证？——这里他们只有了一份实际的存储，只需要保证和cache一致就行了

   虚拟化资源管理

   - 虚拟化平台：k8s
   - **资源调度器**：软件监测（怎么知道服务器压力大了，），自适应的资源分配方案

2. K8s集群中:

   - 最上层是 **Cluster Operator**
     - 根据租户需求来创建MasterSlave集群
     - 为MasterSlave集群分配共享存储
     - 对不同的租户进行资源隔离
   - 中间是 **MasterSlave Controller**
     - 周期性检测pod资源情况
     - 支持**主从节点的scale-up**
     - 支持**从节点的scale-out**
   - 最底层是Ceph分布式存储

3.  监控方法

   - USE法（utilization, saturation, errors）
   - perf: Linux的性能事件
   - 火焰图（CPU profiling）
   - 冷火焰图（off CPU profiling）

4. 监控的资源

   **硬件资源**

   - CPU
   - 内存
   - IO
   - 网络
   - 存储设备

   **软件资源**

   - 文件描述符
   - Kernel mutex
   - process/thread capacity
   - User mutex 

5. 资源分配方法

   如何做到自适应？

   1. 根据建模直接分配（Quasar）
   2. Auto-Scaling
      - 阈值法
      - 强化学习
      - 控制论
      - 时间序列分析

   