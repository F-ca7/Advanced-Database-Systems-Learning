## Multi-Core, Main-Memory Joins: Sort vs. Hash Revisited

1. **问题**：随着多核体系架构的到来，人们认为sort-merge join比radix-hash join要更好(参考**[17 parallel join](https://github.com/F-ca7/Advanced-Database-Systems-Learning/tree/master/17%20parallel%20join)**)，因为有 SIMD指令(当SIMD宽度足够时，sort-merge更快) 和 NUMA (*Non Uniform Memory Access Architecture*)架构。

> NUMA中，虽然内存直接attach在CPU上，但是由于内存被平均分配在了各个die上。只有当CPU访问自身直接attach内存对应的物理地址时，才会有较短的响应时间（后称Local Access）。而如果需要访问其他CPU attach的内存的数据时，就需要通过inter-connect通道访问，响应时间就相比之前变慢了（后称Remote Access）。

2. **Contribution**: 

   - 证明了 大部分时候radix-hash 仍然比sort-merge快，只有当 涉及大量数据时，sort-merge的性能才接近radix-hash。

   - 给出了 在现代处理器上 其他数据操作(除join外)的见解
   - 给出了两种join的最快实现算法
   - 解释了影响算法选择的因素

3. **Related Work**: 

   - 一旦硬件支持矢量指令，且宽度足够（如 具有256位及以上AVX的SIMD），sort-merge很容易击败 radix-hash.  参考 *Sort vs. hash revisited: Fast join implementation on modern multi-core CPUs*(2009)
   - 在 基于NUMA的实现中，即使不使用SIMD指令，sort-merge也可以比radix-hash快。参考 *Massively parallel sort-merge joins in main memory multi-core database systems*(2012)
   - **parallel radix join** 
   - no-partitioning 的radix join. 参考 *Design and evaluation of main memory hash join algorithms for multi-core CPUs*(2011)

4. 影响算法的因素：

   - 输入大小 *input size*: 

     表的相对和绝对大小都对性能有很大的影响。随着大小的增长，关于 只有hash才会扫过数据多趟 的假设被打破，实验表明 sort-merge一样要过多趟数据，性能也会相应降低。

   - 并行度 *degree of parallelism*: 

     当并行度增加时，merge tree中的冲突也会变多。

   - cache竞争 cache contention: 

   > **Cache contention** occurs when two or more CPUs alternately and repeatedly update the same **cache** line. This may be due to. memory **contention**, in which two or more CPUs try to update the same variables.

   - SIMD性能:

     硬件特性起着很大的作用，并且发现SIMD寄存器的宽度并不是唯一的相关因素。更复杂的SIMD设计所固有的 **复杂硬件逻辑 hardware logic** 和**信号传播延迟 signal propagation delay** 可能导致延迟大于一个周期，从而限制了SIMD的优势。

5. 并行sort-merge:

   sort-merge join主要的cost就在排序上，通常使用归并排序。

   1. **Run Generation**

      使用 **排序网络** sorting network，奇偶归并排序网络也是一种比较器个数为O(n*logn*logn)的排序网络。它和归并排序几乎相同，不同的只是合并的过程。普通归并排序的O(n)合并过程显然是依赖于数据的，奇偶归并排序可以把这个合并过程改成非数据依赖型，但复杂度将变高。这个合并过程本身也是递归的。

   2. **Merging Sorted Runs**

      使用 双调合并 Bitonic Merge Networks来合并大的列表。

   > 对于两个元素x，y，如果x<=y，则x，y都位于双调序列的递增部分，而递减部分没有元素，如果x>=y，则x，y都位于双调序列的递减部分，而递增部分没有元素，于是x和y构成一个双调序列。因此，任何无序的序列都是由若干个只有2个元素的双调序列连接而成。
   >
   > 于是，对于一个无序序列，我们按照递增和递减顺序合并相邻的双调序列，按照双调序列的定义，通过连接递增和递减序列得到的序列是双调的。最终，我们可以将若干个只有2个元素的双调序列合并成1个有n个元素的双调序列。
   >

6. Cache conscious sort join

   分为三个阶段：

   - **寄存器内排序 in-register**

     对应了 Run Generation

   - **高速缓存内排序 in-cache**

     对应了 双调排序

   - **高速缓存外排序 out-of-cache**

     继续merge直到所有数据有序（超出了cache的大小，需要到内存中排序）

7. Hash join

   随机访问内存，会导致缓存miss（当hash表大于cache时，基本上每次访问hash表都会发生cache miss），可通过partition来解决。

   - **Radix Partitioning**

     在Partition阶段 考虑到 TLB translation look-aside buﬀers(**地址变换高速缓存**，省去了一次访问内存的时间)的影响。*Radix Partitioning*把每个元组写到他们对应的划分中：

       ```c#
     foreach input tuple t do
     	k := hash(t)
     	p[k][pos[k]] = t
     	pos[k]++
       ```

   - **Software-Managed Buffers**

     先分配一组buffer，其中每个buffer供输出的划分使用（可容纳*N*个元组），当buffer满了后再拷贝到最终的目的地。

     有两层好处：1. 更少的趟数能达到同样

     在本文实现中，就使用了Software-Managed Buffers，其中N的大小恰好使buffer装入一个cache line(64 bytes)

8. 具体join算法

   - Sort-Merge **m-way**

   - Sort-Merge **m-pass**

   - Massively Parallel Sort-Merge **mpsm**

     通过Software-Managed Buffer对R进行划分，将R分配给不同NUMA的区域或线程，每个线程对自己区域内的数据进行排序。

   - Radix Hash **radix**

   - No-Partitioning Hash **n-part**

     将关系**R**和**S**划分为大小相等的部分，分配给工作线程。每个worker把负责的关系**R**的元组填充到hash表，再在**S**中找到匹配的元组。