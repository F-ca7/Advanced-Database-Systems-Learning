## I/O

1. I/O设备

   - 块设备：将信息存储在固定大小的块中，每个块都有自己的地址，还可以在设备的任意位置读取一定长度的数据，例如硬盘,U盘，SD卡等
   - 字符设备：在I/O传输过程中以字符为单位进行传输的设备，例如键盘，打印机等

2. /dev

   - 即设备 device，包含了所有Linux系统中使用的外部设备。但是这里并不是放的外部设备的驱动程序，这一点和windows操作系统不一样。它实际上是一个访问这些外部设备的端口
   - **设备都认为是文件**（Unix风格）
   - 常见设备文件：
     - /dev/hd[a-t]：IDE设备
     - /dev/sd[a-z]：SCSI设备
     - /dev/fd[0-7]：标准软驱
     - /dev/loop[0-7]：本地回环设备
     - /dev/ram[0-15]：内存
     - /dev/null：无限数据接收设备,相当于黑洞
     - /dev/zero：无限零资源
     - /dev/tty[0-63]：虚拟终端
     - /dev/ttyS[0-3]：串口

3. I/O调度器：将请求按照对应在块设备上的扇区号进行排列，减少磁头的移动，提高效率。

4. 四种IO调度器 （算法）

   1. Noop：即电梯调度算法，适合那些不希望调度器重新组织IO请求顺序的应用，对于SSD也较为友好
   2. Deadline：保证每个IO请求在一定的时间内一定要被服务到，以此来避免某个请求饥饿。在一些多线程场景下，Deadline比CFQ要好。
   3. Anticipatory：利用局部性原理，**预期** 进程在做完一次IO后 还会在此处继续做IO操作。（后来被CFQ取代，可达到同样的效果）
   4. Completely Fair Queuing ：默认的调度算法（对于通用服务器较好）。

   [Percona这篇博客](https://www.percona.com/blog/2009/01/30/linux-schedulers-in-tpcc-like-benchmark/)反映出了 在tpcc中，Deadline算法 和 noop算法在数据库中更优。

---------

**Java中的I/O**

