# 内存管理和优化

## CACHE
- 为什么需要CACHE？局部性原理
- 三级CACHE结构，L1-L2 CPU内core共享，L3 CPU独立
- 内存和CACHE的关系
- cache对性能的影响
	- 访问延迟对比
	- CacheLine
	- Cache miss / Cache hit，颠簸
- 分析和测量CACHE命中率
- 编写CACHE友好代码

## 虚拟地址空间
- C进程内存布局：stack内存/heap内存
- 段页式内存管理
- MMU + 硬件合作
- page fault：minor page fault / major page fault
- segment fault
- 相关linux命令：pmap、vmstat、free，虚拟内存占用和RSS

## 动态内存分配
- 动态内存分配器的历史（多线程慢及改进）
- 动态内存分配器介于os与应用程序之间
- 内存碎片：内碎片 / 外碎片
- 动态内存分配器的目标
    - 高吞吐
	- 高有效内存使用率
	- 低时延
- 一次malloc要耗费多少cpu时间
- 几种经典的内存分配器：ptmalloc / tcmalloc / jemalloc
- 动态内存分配器的挑战（引出为什么需要MemPool）

## 内存池
- 为什么内存池更快？(因为放弃了中间free，本质上是以空间换时间)
- 批发转零售的策略
- 经典内存池的实现案例
	- Nginx内存池
	- doris内存池
	- loki小对象分配器
- 延伸：对象池

## 内存泄漏
何谓内存泄漏？动态申请的内存丢失引用
- C++ operator new/delete重载
- C wrap malloc/free
- 地址消毒器
- valgrind/cachegrind/asan

## 其他

- mmap
- Shared Memory
- 内存对齐及影响：posix\_memalign 
- 对内存的优化不应该作为项目的高优先级目标

## 参考资料

