# 系统底层基础
计算机基础： 
	- 多CPU、多核
	- NUMA
	- 内存总线、PCI总线
	- 编译器优化、乱序执行
	- 流水线
	- PIPELINE
kernel相关：调度（context swap）

# 内存管理和优化
- cache：L1-L2-L3 & cache miss/hit
- page fault
- 虚拟地址空间：MMU、段页式管理
- kernel -> malloc库 -> 内存池层次
- 内存池和内存复用
- 内存泄漏定位

# 数据结构和算法
- hash / 一致性hash
- roaringbitmap
- hll
- skiplist
- ...

# 并发和并行
- 并发和并行的关系
- 数据分片：murmurhash、一致性哈希
- 任务并行
- 多进程/多线程/协程
- SIMD

# 多线程和多线程同步
- 异步和同步
- 原子操作
- 内存屏障
- 免锁
- 伪共享
- 生产者/消费者模式 mutex+cv
- 如何设置线程数？

# 定位系统瓶颈
- 查看内存带宽、内存占用、cache命中
- CPU利用率
- I/O
- 工具：perf、火焰图、prof2dot.py、PMU-Tools

# 优化系统性能
- 基本现实
    - 内存拷贝速度
	- 访存时延
	- 锁的耗时
	- cs耗时
	- 虚函数开销..
- 分析法
	- USE
	- TMAM
- 策略
	- 置换：空间置换时间、带宽置换计算
	- 查表法
	- 延迟计算、预计算
	- 数据预取
	- CPU offload：RDMA
