## Linux Storage

1. VFS (Virtual File System)

   Linux内核里提供文件系统接口 给**用户态应用程序**的一个虚拟文件系统层。也就是不管是访问什么文件，也不管使用的是什么文件系统（ext2/ ext3/ ext4/ Minix），基本都可以统一地使用诸如open(), read()和write()这样的接口

2. 概念

   - SuperBlock: 文件系统的第一个Block，描述文件系统的元信息
   - Mount Tree: 访问文件系统必须通过mount point去访问
   - File System: 文件系统必须向VFS注册自己的类型，并实现mount方法
   - Inode: 也就是Index node，每个有一个全局唯一(对于一个文件系统)的inode号
   - dentry: 目录也是一种文件，包含文件与子目录，以及它们对应的inode号
   - Opened file: 当进程打开一个文件，VFS会创建一个文件实体，并在进程的open file table中分配一个对应的index，即文件描述符fd

3. 文件系统的组成

   - superblock
   - inode bitmap: 作为inode table的索引，保存了 已分配 / 未分配的inode信息
   - data block bitmap
   - inode tables
   - data blocks: 真正存储数据的数据块

4. inode

   - 文件元信息
   - 文件数据的存储位置

   对于大文件的支持：

   - ext2: 通过多级指针的方式，但是对大文件还是不太友好，因为要用很多指针去指向
   - ext4: 通过extent树

   琐碎文件导致inode超出范围，可能会创建文件失败。

5. dentry

   

6. 链接

   - 硬链接：不能跨分区，因为它只相当于文件的别名 （inode在同一个文件系统中才唯一）

     对于 `ln 1.txt 3.txt`，dentry如下：

     | name  | inode   |
     | ----- | ------- |
     | 1.txt | 123     |
     | 2.txt | 456     |
     | 3.txt | **123** |

     假设进程A 与 进程B打开了同一个文件1.txt，进程C打开了硬链接3.txt；则A和B的file指针**包含的是同一个dentry**，C的file指针**是另一个dentry**，但**两个dentry的成员inode是同一个**。

   - 软链接：又叫符号链接，这个文件包含了另一个文件的路径名；可以链接不同文件系统的文件，可以理解为windows的快捷方式

     对于 `ln -s 1.txt 3.txt`，dentry如下：

     | name  | inode   |
     | ----- | ------- |
     | 1.txt | 123     |
     | 2.txt | 456     |
     | 3.txt | **789** |

     比如用"rm"命令删除文件时，删除的只是原文件的路径和inode之间的关联，而不是这个inode本身，文件的内容依然存在于磁盘中，，因而只能算是"unlink"。所以直接关联inode的hard link不受影响，而关联原文件路径的soft link此时相当于是一个dangling reference

   注意：硬链接不能link目录，软链接可以。

7. 不要混用buffer I/O和 direct I/O，比如进程A用buffer IO 读取文件，会有page cache，从而接着不用进行磁盘的IO；但是进程B通过direct IO修改了磁盘上的文件，会导致进程A仍需要进行磁盘的IO。

8. buffer IO 从磁盘中读取到缓冲区，会将对应的page标记为dirty；每次做这个操作时，会检查系统的脏页数量是否过多，如果超出阈值，会执行balance_dirty_page的操作，此时原本异步的IO也会变成同步的。

9. Block层

   - BIO: 包含了 数据在磁盘上的位置 以及 在内存上的位置信息
   - Request
   - cmd

10. IO调度器

11. 三种队列类型

    1. blk-sq（以前磁盘速度慢，瓶颈在磁盘上）
    2. blk-mq：支持随CPU核数的线性扩展
    3. raw

12. 工具：

    - perf stat
    - BPF tool（效率高）
    - iostat
    - blktrace

