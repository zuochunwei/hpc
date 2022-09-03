# 内存管理和优化

- 存储程序原理奠定了现代计算机的基础，将程序像数据一样存储在计算机内存，计算机一条接着一条指令执行。内存不仅存储数据，而且存储程序，是计算机系统的核心功能组件之一。
- 围绕局部性原理，计算机系统构建起“寄存器->缓存(SRAM)->内存(DRAM)->磁盘(HDD/SSD)”的存储层次结构：离CPU越近，容量越小、速度越快、单元价格越贵，离CPU越远，容量越大，速度越慢，单元价格越便宜。
- 内存可被为磁盘的高速缓存，因为CPU的速度比磁盘快很多，所以，通过在CPU与磁盘之间架设内存这座桥梁，填平了CPU和磁盘之间的速度鸿沟。
- 内存管理和优化对性能影响很大，是编写高性能程序的关键，但内存管理和优化牵扯很多系统底层知识，本章将梳理相关内容，从软硬件结合出发，力求简明扼要讲清楚这个主题的内容，如需更深入的理解，则需扩展阅读。

## 减少内存拷贝
- CPU Offload
    - CPU的最主要工作是计算，而不是进行数据复制
    - DMA：DMA全称为Direct Memory Access，即直接内存访问。意思是外设对内存的读写过程可以不用CPU参与而直接进行。
    - RDMA:RDMA（ Remote Direct Memory Access ）意为远程直接地址访问，通过RDMA，本端节点可以“直接”访问远端节点的内存。所谓直接，指的是可以像访问本地内存一样，绕过传统以太网复杂的TCP/IP网络协议栈读写远端内存，而这个过程对端是不感知的，而且这个读写过程的大部分工作是由硬件而不是软件完成的。

### 编译器和编程语言支持
c语言支持hook malloc、free、realloc等接口，这样你可以从比较低的层次干预和统计内存分配。
c++支持operator new/new[]、operator delete/delete[]、以及类的operator new/delete重载。
c++的标准库容器，比如vector、list、map等，都支持传入自定义allocator，你可以接管内存配置，而不限于默认分配器。
COW（Copy On Write）写时拷贝是一项能节省拷贝的技术，fork出来的进程也用到了cow，如果要全量拷贝，那fork的返回会延迟很多。
为了防止内存泄漏，有时候会借助RAII技术。
引用计数是实现智能指针的关键技术，需要区分弱引用和强引用，以及shared和unique，所有权的概念。
ptmalloc可以开启一些统计选项，这可以为排错提供帮助。
libc的动态内存分配器默认是ptmalloc，你也可以用google的tcmalloc，以及jemalloc等动态内存分配器替换。
监控和查找内存泄漏问题，你可以借助valgrind工具，不过valgrind对代码有要求，只有符合它期望的程序才能valgrind干净，不然误报会比较多。

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

