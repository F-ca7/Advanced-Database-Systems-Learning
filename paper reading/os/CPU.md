## CPU

1. 多任务操作系统分为非抢占式 和 抢占式多任务，大多数现代操作系统是 **抢占式**的（也就是说 调度器会决定什么时候停止一个进程，同时挑选出另一个进程运行）。

2. 现有的Linux调度策略（也就是选择一个新进程的策略）

   ```c
   #define SCHED_NORMAL	0 	// 默认的调度策略
   #define SCHED_FIFO		1 	// 针对实时进程的先进先出调度,适合对时间性要求比较高但每次运行时间比较短的进程
   #define SCHED_RR		2	// 针对的是实时进程的时间片轮转调度,适合每次运行时间比较长得进程
   #define SCHED_BATCH		3	// 针对批处理进程的调度，适合那些非交互性且对cpu使用密集的进程
   /* SCHED_ISO: reserved but not implemented yet */
   #define SCHED_IDLE		5	// 适用于优先级较低的后台进程
   ```

3. 普通进程的CFS算法

   在 pick_next_entity函数中，调用了wakeup_preempt_entity，其主要作用是 根据进程的**虚拟时间**以及**权重**的结算进程的粒度，以判断其是否需要抢占（返回0或1）；同时会保证每个进程可以运行一个最小的时间粒度。

4. Nice值

   ```
   F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
   4 S     0 20312 20304  0  80   0 - 29050 wait   pts/0    00:00:00 bash
   0 R     0 20356 20312  0  80   0 - 37234 -      pts/0    00:00:00 ps
   
   ```

   > F：flag，标识了进程拥有的权限，4即为superuser
   >
   > S：stat，状态（R-Running，S-Sleep，T-Stop，Z-Zombie）
   >
   > UID：执行者的身份
   >
   > PID：进程ID
   >
   > PPID：父进程ID
   >
   > C：CPU百分比
   >
   > PRI：priority，优先级，越小越优先
   >
   > NI：nice，进程优先级的修正数值
   >
   > ADDR：address，程序在内存的哪个部分
   >
   > SZ：使用的内存大小
   >
   > WCHAN：是否运行中
   >
   > TTY：终端位置
   >
   > TIME：使用的CPU时间
   >
   > CMD：运行的指令

   通过`ps -el`可以看到进程NI列即为nice值；PRI列表示的是进程的优先级，是一个整数，它是调度器选择进程运行的基础。

   CFS直接分配的不是时间片，而是**CPU使用比**，这个比例会收到nice值影响，nice值低比重就高，nice高比重就低。

   普通进程的优先级分为

   - 静态优先级：不会随时间而变化。只能通过*系统调用nice(value)* 进行修改
   - 动态优先级：即PRIO。

   总的来说可以通过 nice这个系统调用来改变进程的优先级。

5. 

