# 系统底层基础
计算机基础： 
	- 多CPU、多核
	- NUMA
	- 内存总线、PCI总线
	- 编译器优化、乱序执行
	- 流水线
	- 置换
	- PIPELINE
kernel相关：调度（context swap）

# 内存管理和优化
- cache：L1-L2-L3
- page fault
- 虚拟地址空间：MMU、段页式管理
- kernel -> malloc库 -> 内存池层次
- 内存池和内存复用

# 数据结构和算法
- hash
- roaringbitmap
- hll
- skiplist

# 并发编程
- 并发和并行
- 数据分片：murmurhash、一致性哈希

# 多线程和多线程同步
- 异步和同步
- 原子操作
- 内存屏障
- 免锁
- SIMD等
- 生产者/消费者 mutex+cv
- 如何设置线程数？

# 定位系统瓶颈
- 查看内存带宽、内存占用、cache命中
- 工具：perf、火焰图

# 优化系统性能
- 分析法
	- USE
	- TMAM
- 策略
	- 空间置换时间
