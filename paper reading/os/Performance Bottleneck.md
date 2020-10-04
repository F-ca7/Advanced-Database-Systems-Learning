## Performance Bottleneck

> 【High Performance TiDB】Lesson 03：通过工具寻找 TiDB 瓶颈

1. 性能的定义

   性能 = 产出 ÷ 资源消耗

   产出：

   - 每秒查询率 (QPS)
   - 数据吞吐量 (Bytes)

   资源消耗：

   - CPU
   - 磁盘IO
   - 网络IO
   - 流量

2. 资源消耗的设计目标：

   让最紧缺的资源消耗满。

3. 流水线：达到 资源-时间轴 重叠的效果

   比如以下三种情况的对比：

   1. 整个大任务分为 Encode、Write，其中Encode消耗CPU，耗时100s；Write消耗IO，耗时70s，共170s
   2. 整个大任务划分为10个小任务，交替进行，即 Encode(Block0) 10s、Write(Block0) 7s、Encode(Block1) 10s、Write(Block1) 7s... 共170s
   3. 小任务间流水线进行，即 Encode(Block0) 10s、Write(Block0) + Encode(Block1) 10s、Write(Block1) + Encode(Block2) 10s ... Write(Block9)，共107s

4. 流水线Batch的粒度

   - 小粒度
     - batch总量相对变多了，锁竞争可能开销变大，总的提交代价相对也变大了
     - 运行时占用资源小，整体资源使用峰值低
     - 单次失败代价更低
   - 大粒度
     - 整体占用资源较多
     - 单次失败代价更大
     - 首尾的batch不能受益
     - 通常吞吐更好，但latency更高

   例一. 于HDD上进行文件拷贝，HDD寻道时间3-6ms，随机读写的最大QPS为 400-800，读写带宽为 100 MB/s —— 100MB / 800 ≈ 100KB，所以Batch的粒度不能小于100KB，否则造成性能下降（吞吐达不到100MB/s）。

   例二. 将数据划分块，并发进行编码，使用互斥锁对块状态 加锁，在高竞争状态，mutex的QPS约为 10-20万，则总吞吐不会高于 10万*batch粒度；若batch块大小为4KB，则整体吞吐不高于 400MB/s。

5. 瓶颈类型

   - CPU

     - CPU usage高

       只能减少计算的开销：比如用更优的算法，通过cache中间结果减少重复计算

     - CPU load高

       - 线程太多，频繁切换导致：考虑跨线程的交互是否必要
       - 等IO，可以在top中看到**iowait**比较高：说明IO是限制性能的瓶颈

   - IO

     - IOPS上限

       减少读写次数，提高cache的hit率

     - IO带宽上限

       减少磁盘读写的流量：比如采用 更紧凑的数据存储格式、更小的读写放大（）

     - fsync次数上限（通过工具来查看）

       减少fsync次数：可以采用group commit

   - Network

     - 网卡带宽上限（千兆网卡、万兆网卡）：降低传输的数据量（紧凑的数据格式、计算下推）
     - send/recv系统调用太频繁：批量进行发送与接收

6. 工具使用可参考其他几篇