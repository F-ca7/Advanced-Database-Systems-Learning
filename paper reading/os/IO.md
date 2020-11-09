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
     
       SCSI硬盘和普通IDE硬盘相比有很多优点：接口速度快，并且由于主要用于服务器，因此硬盘本身的性能也比较高，硬盘转速快，缓存容量大，CPU占用率低，扩展性远优于IDE硬盘，并且支持热插拔
     
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

5. 读写的本质

   - **读**：read也好，recv也好只负责把数据从底层缓冲copy到我们指定的位置。
   - **写**：用户态的数据copy到系统底层去，然后再由系统进行发送操作，返回成功只表示数据已经copy到底层缓冲，而不表示数据以及发出,更不能表示对端已经接收到数据。

6. 五种IO模型

   1. 阻塞IO：读写数据过程中会发生阻塞现象。

      read时候如果**发现没有数据**，**会一直等待**；**如果读取到的数据比参数指定长度小的话，也是立即返回**，故read完一次需要判断一下读取到的长度。

      write时，会一直等到写完所有数据再返回（与读不同）。

      阻塞IO在性能方面是很低下的，如果要使用阻塞IO完成一个Web服务器的话，那么对于每一个请求都必须启用一个线程进行处理。

   2. 非阻塞IO：

      在非阻塞时，read如果发现没有数据会立即返回，如果有数据则是有多少 读多少，（与阻塞的区别在于 没有数据到达时是否立刻返回）。

      write时，能写多少就写多少。

      

   3. 多路复用IO（比如Java的NIO）

      在多路复用IO模型中，会有一个线程不断去轮询多个socket的状态，只有当socket真正有读写事件时，才真正调用实际的IO读写操作。比如在Java NIO中，是通过**selector.select()**去查询每个通道是否有到达事件，如果没有事件，则一直阻塞在那里，因此这种方式会导致用户线程的阻塞。

   4. 信号驱动IO：当用户线程发起一个IO请求操作，会给对应的socket注册一个信号函数，然后用户线程会继续执行，当内核数据就绪时会发送一个信号给用户线程，用户线程接收到信号之后，便在信号函数中调用IO读写操作来进行实际的IO请求操作

   5. 异步IO：IO操作的两个阶段都不会阻塞用户线程，这两个阶段都是由内核自动完成，然后发送一个信号告知用户线程操作已完成（和*信号驱动*的区别在于：在信号驱动模型中，当用户线程接收到信号表示数据已经就绪，然后需要**用户线程调用IO函数进行实际的读写操作**；而在异步IO模型中，收到信号表示IO操作已经完成，不需要再在用户线程中调用IO函数进行实际的读写操作）；需要操作系统的底层支持，在Java 7中，提供了Asynchronous IO

7. 两种设计模式

   负责工作都是将单独的IO事件通知到上层模块。

   - Reactor: 理解为反应器（被动的），实现了以下功能：

     - 供应用程序注册和删除关注的事件句柄
     - 运行事件循环
     - 有就绪事件到来时，分发事件到之前注册的回调函数上处理

     本质就是当IO事件(如读写事件)触发时，通知我们主动去读取，也就是要我们主动将socket接收缓存中的数据读到应用进程内存中（就很像epoll）。对于**耗时短的处理场景处理高效**，

   - Proactor: 主动的，只负责发起异步读写操作。IO操作本身由操作系统来完成（所以说需要系统级别的异步支持）。我们需要指定一个应用进程内的buffer(内存地址)，交给系统，当有数据包到达时，则写入到这个buffer并通知我们收了多少个字节

     能够处理**耗时长的并发场景**，性能也较好。

8. **iostat**命令

   从盘的角度看IO，`iostat -x 1 -m`

   给出指标有

   - r/s: 读的次数
   - w/s: 写的次数
   - rMB/s: 读流量
   - wMB/s: 写流量
   - await: 平均每次I/O操作等待的时间
   - r_await: 读操作等待时间，如超过5ms级别，可认为比较高，即读压力大
   - w_await: 写操作等待时间，如超过5ms级别，可认为比较高，即写压力大

9. **iotop**命令

   从线程的角度看IO，可以看各个线程的IO累积量，有没有超出预期，也可反应fsync

10. **iosnoop**工具

    https://github.com/brendangregg/perf-tools/blob/master/iosnoop

    检测磁盘延迟的毛刺

    可以结合[HeatMap](https://github.com/brendangregg/HeatMap)来可视化查看指标

11. fio

    可以用于测试磁盘的指标，包括：

    - 读写带宽
    - IOPS
    - 每秒 fsync的上限

---------

**Java中的文件I/O** (文件IO实践 https://www.cnkirito.moe/file-io-best-practise/)

1. 原生的文件IO读写方式

   - 普通IO: java.io.FileWriter,   java.io.FileReader

   - FileChannel: java.nio.channels.FileChannel，注意 NIO 并不一定意味着非阻塞，这里的FileChannel 就是阻塞的

     `FileChannel channel = new RandomAccessFile(new File("test"), "r").getChannel();`

   - MMAP (内存映射): 由 FileChannel 调用 map 方法衍生出来的一种特殊读写文件的方式

     `MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());`

2. **FileChannel**

   大部分时候是与 **ByteBuffer**类交互，ByteBuffer可视为 字节数组的一个封装类：

   ```java
   public abstract class ByteBuffer
       extends Buffer
       implements Comparable<ByteBuffer>
   {
       // These fields are declared here rather than in Heap-X-Buffer in order to
       // reduce the number of virtual method invocations needed to access these
       // values, which is especially costly when coding small buffers.
   
       final byte[] hb;                  // Non-null only for heap buffers
       final int offset;
       boolean isReadOnly;                 // Valid only for heap buffers
   }
   ```

   FileChannel的read和write方法是线程安全的，因为在真正读写会通过 synchronize(positionLock) 来进行互斥。

3. 为什么FileChannel比普通IO快？

   1. FileChannel 采用了 ByteBuffer 这样的内存缓冲区，让我们可以非常精准的控制写盘的大小；所以当 FileChannel 在一次写入 4kb 的整数倍（具体数值受 磁盘结构、操作系统、CPU的影响）时，可以发挥出更高的性能
   2. FileChannel不是把ByteBuffer直接写入磁盘，write方法是先写入PageCache缓存，最后由操作系统将PageCache的数据落盘（所以  FileChannel 也提供了一个 force() 方法，通知操作系统进行及时的刷盘）

4. mmap的MappedByteBuffer 

   断点执行完 `fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, 1024 * 1024)` 后，发现即使不写入数据，磁盘上也会生成一个1M大小的文件（字节全为0）。

   原理如下：

   > mmap 把文件映射到用户空间里的虚拟内存，**省去了从内核缓冲区复制到用户空间的过程**，文件中的位置在虚拟内存中有了对应的地址，可以像操作内存一样操作这个文件，相当于已经把整个文件放入内存，但在真正使用到这些数据前却不会消耗物理内存，也不会有读写磁盘的操作，只有真正使用这些数据时，虚拟内存管理系统 VMS 才根据缺页加载的机制从磁盘加载对应的数据块到物理内存。这样的文件读写文件方式少了数据从内核缓存到用户空间的拷贝，效率很高

5. Java MappedByteBuffer 的三个注意事项

   - MMAP 使用时必须实现指定好内存映射的大小，并且一次 map 的大小限制在 1.5G 左右，重复 map 又会带来虚拟内存的回收、重新分配的问题，对于文件不确定大小的情形不友好
   - MMAP 使用的是虚拟内存，和 PageCache 一样是由操作系统来控制刷盘的，虽然可以通过 force() 来手动控制，但这个时间把握不好，在小内存场景下很麻烦
   - 当 MappedByteBuffer 不再需要时，可以手动释放占用的虚拟内存

6. 不管是机械硬盘还是固态硬盘，顺序读写总比随机读写快

   FileChannel多线程的顺序读写 需要在调用时额外进行同步，如

   ```java
   private synchronized static void syncWrite(FileChannel channel, byte[] data) throws IOException {
       channel.write(ByteBuffer.wrap(data), writePos);
       writePos += data.length;
   }
   ```

   这是因为多线程并发write且不加同步会导致文件空洞，其执行顺序可能如下：

   1. 线程1写入偏移位置[0, 1024)
   2. 线程3写入偏移位置[2048, 3072)
   3. 线程2写入偏移位置[1024, 2048)

   导致不是绝对的顺序写入。

7. 堆内内存 vs 堆外(Direct)内存

   |              | 堆内内存                          | 堆外内存                                                     |
   | ------------ | --------------------------------- | ------------------------------------------------------------ |
   | **实现**     | 数组，使用的是JVM堆内存           | unsafe.allocateMemory(size) 返回直接内存                     |
   | **大小限制** | JVM启动参数-Xms -Xmx              | -XX:MaxDirectMemorySize，以及及其本身的虚拟内存              |
   | **GC**       | 由垃圾回收器处理                  | 当 DirectByteBuffer 不再被使用时，会出发内部 cleaner 的钩子，保险起见，可以考虑手动回收：((DirectBuffer) buffer).cleaner().clean(); |
   | **内存复制** | 堆内内存 -> 堆外内存 -> pageCache | 堆外内存 -> pageCache                                        |

   最佳实践：

   - 当需要申请大块的内存时，堆内内存会受到限制，只能分配堆外内存
   - 堆外内存适用于生命周期中等或较长的对象。(如果是生命周期较短的对象，在 YGC 的时候就被回收了，就不存在大内存且生命周期较长的对象在 FGC 对应用造成的性能影响)
   - 堆内内存刷盘的过程中，还需要复制一份到堆外内存
   - 还可以使用池 + 堆外内存 的组合方式，来对生命周期较短，但涉及到 I/O 操作的对象进行堆外内存的再使用 (Netty 中就使用了该方式)。在比赛中，尽量不要出现在频繁 `new byte[]` ，创建内存区域再回收也是一笔不小的开销，使用 `ThreadLocal<ByteBuffer>` 和 `ThreadLocal<byte[]>`
   - 创建堆外内存的消耗要大于创建堆内内存的消耗，所以当分配了堆外内存之后，**尽可能复用**它

8. 

   

