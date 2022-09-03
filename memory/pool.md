## 内存池

### 为什么内存池更快？(因为放弃了中间free，本质上是以空间换时间)
- 池化/池技术是一种通过复用，从而降低运行时开销的通用技术，池的本质是空间换时间。
- 系统启动时，按设定预留资源；运行中，向池动态申请资源；用完后，将资源返还池。
- 池技术能减少资源创建和回收的开销，提升系统的并发和吞吐，广泛应用于低延迟系统。

既然系统已经有了动态内存分配器，而且新的动态内存分配器层出不穷，比如tcmalloc/jemalloc，为什么还需要内存池呢？

换言之，内存池为什么能做到比通用内存分配器快很多，究其根本，是因为通用内存分配器，要支持每个分配的块的独立回收，通过malloc分配的内存块，可以在程序运行中的任何时间，通过free返回系统，就这一个要求，决定了动态内存分配器快不到哪里去，它需要在释放的时候处理碎片合并，需要在分配的时候处理找空闲块以及多线程等问题，而池可以不用处理多线程问题，池可以只记录一个当前分配位置的游标，分配的时候，简单的移动游标即可，它不需要为分配的块分配额外头部，分配出去的块不会单独回收，这才是它快的秘诀。

- 批发转零售的策略
- 经典内存池的实现案例

## doris是如何做内存管理的？
Apache Doris是一个基于MPP架构的高性能、实时的分析型数据库，我们分析一下它的内存管理方案。

**doris的内存管理方案分三层：**
- 系统分配器（SystemAllocator）：封装系统/标准接口，提供统一的allocate/free接口
    - allocate(size)根据size调用posix_memalign()或者mmap()分配内存，会在分配的时候做内存对齐
    - free()接口调用munmap()或者free()释放内存
    - mmap()/munmap()配对，posix_memalign()/free()配对，通过配置二选一
- 块分配器（ChunkAllocator）
    - 块（Chunk）：Chunk表示通过SystemAllocator::allocate接口分配的内存块，Chunk包含内存块首地址、尺寸、core_id等信息
    - 为每个CPU core维护一个chunk_arena，chunk_arena类似ptmalloc里arena的概念，不同的是ptmalloc中的arena对应到线程，每个线程一个arena
    - 每个chunk_arena包含一个chunk_list的数组
    - chunk_list为每个size维护一个该size的chunk集合，为了减少各种size的数量，只维护固定size的chunk集合，比如8、16、32、64、128、256...，所以如果分配请求的大小是34字节，那么会向上圆整到64字节，通过size求对数就得到该size的块所属chunk_list（在chunk_list数组中的下标）
    - 块分配器会使用上述的系统配置器分配/回收内存，块配置器是单件（唯一实例），分配Chunk的接口是线程安全的
- 内存池（MemPool）
    - 提供allocate()、clear()、free_all()等操作接口
    - 维护通过allocate接口分配的ChunkInfo的列表，ChunkInfo在Chunk上增加了一个已分配字节数
    - 内存池会通过块分配器分配大块，每次分配的大块的大小会按X2（策略决定）增加，从而确保不会频繁调用块分配器的allocate接口
    - 通过内存池的allocate接口分配的内存，不支持单块free，不支持中途free，只支持统一释放free_all()
    - 内存复用：clear()接口会重置ChunkInfo上Chunk的已分配字节数，clear并不会真正回收内存
    - 内存池的操作接口是非线程安全的
---
### SystemAllocator
屏蔽了动态内存管理相关的底层系统调用和标准C/Posix编程接口，上层应用不再直接调用底层接口，而是调用SystemAllocator封装的编程接口：allocate/free。

- ChunkAllocator是怎么工作的？
    - ChunkAllocator处于3层结构的中间层
    - ChunkAllocator在SystemAllocator之上，会使用SystemAllocator的allocate/free接口申请和释放内存块
    - ChunkAllocator在MemPool之下，提供allocate和free接口供MemPool使用
    - ChunkAllocator减少了多线程竞争，ChunkAllocator维护core_num个ChunkArena对象，ChunkArena内维护一个chunk_list数组，为size=2^n的固定size块维护一个free list，内存申请的时候，会对请求的size向上圆整

---
- 因为每个core都有一个ChunkArena对象，所以上层应用代码申请内存的时候，先获取当前线程正在哪个核上运行，从而找到对应的ChunkArena对象，再通过size找到对应的free list，再从该free list上摘除一个块，返回给提出内存申请的上层。

- 多个逻辑线程依然可能调度到同一个核上执行，虽然多个线程不会在一个核上同时执行申请动态内存，但多个线程在一个核上交错执行（申请内存）的情况，依然会引发对free list的数据竞争（虽然这种情况出现的概率很小），这时候只需要用test_and_swap原子操作不停尝试就行了，如果尝试一定次数还不成功，则执行线程主动yield，让出CPU，从而让另一个在该核上执行内存分配的线程有机会继续执行，进而修改atomic_flag，然后之前yield CPU的线程被重新调度执行。

- TAS（test and swap）是很快的，且冲突概率变得非常小（因为每个核都有一个atomic_flag，不会所有线程竞争一个锁），这样的免锁设计，让分配内存变得很高效。

- ChunkAllocator也做了一层cache，通过ChunkAllocator::free释放的内存块，并不一定会真正调用底层的free，只在预留size超过配额（2G）的情况下，才会调用SystemAllocator的free()，这样进一步减少了对系统底层动态内存管理相关API的调用频次。

ChunkAllocator是单件，唯一实例，被所有MemPool对象共享。

---
### MemPool
- 咱们进一部分分析MemPool的设计，先给一张MemPool的图：

首先，我们来看MemPool的作用：

- 内存池在SystemAllocator/ChunkAllocator/MemPool的层次结构中，位于顶层，它依赖于下层ChunkAllocator，间接依赖SystemAllocator，下层的类不反向依赖于MemPool。

- 先说Chunk和ChunkInfo。
    - Chunk就是底层接口单次分配的内存块，Chunk持有内存块首地址data，内存块大小size，以及分配的时候执行线程在哪个core上执行。
    - ChunkInfo包含Chunk，同时多了一个int allocated_size，这是因为，为了减少对SystemAllocator::allocate()的调用次数，所以单次分配的chunk会比较大，几K，几十K，甚至XX M（兆），这个大的size记录在chunk->size上，但是，上层应用一次分配的内存可能比较小，几十字节之类，所以，该chunk还有多少字节可用（已经使用了多少字节），需要有一个记录，这就是allocated_size，相当于一个游标，每次从该chunk分配x字节，那就把allocated_size这个游标往增长的方向移动x字节（实际上会考虑到对齐）。

- 所以，对SystemAllocator::allocate()的调用，相当于批发进货；对MemPool::allocate()的调用，相当于零售。效果上，就是减少了底层API的调用频率，减少了多线程竞争。

- MemPool持有一个next_chunk_size，它表示下次调用ChunkAllocator分配接口allocator的时候，需要分配多大，它被初始化为4K，下次分配的时候，会增加到8K，当然如果下次申请的size大于8K，则会取max。

- next_chunk_size会一直增加，直到触达最大配置值，这样的设计，目的还是为了减少底层分配次数。

- 每次ChunkAllocator::allocate()都会返回一个Chunk，进而包装为ChunkInfo，被MemPool管理起来，所以MemPool会有多个ChunkInfo，用chunk_index标识chunk。

- MemPool记录一个current_chunk_idx，这个idx记录了上次成功分配的ChunkInfo，下次分配的时候，先从current_chunk_idx指向的chunkInfo里尝试分配，如果该ChunkInfo的剩余内存空间不够，则会查找其他ChunkInfo，直到找到能满足分配请求的ChunkInfo，如果现有的所有ChunkInfo都不满足，那就走ChunkAllocator的allocate，并把新申请的Chunk，放入ChunkInfo list。

- MemPool不支持单次分配的内存free，但是支持free_all，这会free该MemPool的所有Chunk。

- MemPool::Clear()接口不会真正free Chunk，而是会重置allocated_size，复用原内存chunk。

- 一个细节，关于ChunkAllocator，分配的时候，会首先从线程运行的core上的ChunkArena分配，如果没有合适的，会从其他Core的ChunkArena里分配，再分配不到，才会从system_allocate，这样做的目的，是减少内存cache量。

我们做内存池有几个目标：

- 吞吐，吞吐越大越好，能满足各种不同size，各种内存分配场景的大吞吐最好。
- 提高存储空间利用率，千方百计减少碎片（内碎片+外碎片）。

为了提高速度，我们经常要做cache，但是cache多了，会造成宝贵的内存资源的浪费，所以，需要balance。

---

#### NGINX内存池
NGINX是高性能高并发服务器的典范，NGINX常被用来作为HTTP和反向代理Web服务器，作为HTTP服务器，NGINX的工作模式是：接收一个来自client的request，处理该request，然后向client吐出response。

这样的工作模式非常适合用内存池优化动态内存分配，NGINX内存池也是NGINX优秀设计的一个典型案例。

NGINX为每个连接创建一个内存池对象，在处理该连接请求的过程中，所有的动态内存分配请求，都向内存池提交，返回结果后，再清理该连接的内存池，清理内存池不会把内存真正返回系统，会根据策略把一定量的内存缓存起来复用。

因为请求的处理过程都在一个线程内进行，所以该连接的内存池不需要处理多线程同步（免锁），中途也不需要释放内存（返回结果后统一释放），避免了内存泄漏的隐患，安全性也得到了提升。

- NGINX内存池的设计概要：
    - request处理过程中，所有动态内存分配，都向连接专属的内存池申请。
    - 大尺寸内存按需分配，走malloc/free或mmap/munmap，大块内存块用链表串起来。
    - 小尺寸内存从内存页分配，批发转零售：先通过malloc/mmap申请一个4K（大小可配）的内存页（批发），再从内存页里细分（零售），内存也也会用链表串起来。
    - 内存池记着当前页指针，下次分配先从当前页尝试，内存页多次不满足分配请求后，才会修改当前页指针。

- NGINX内存池的设计考虑：  
    - 对动态内存分配请求，根据参数size，区别对待：
    - 如果大于等于某个阈值，则被视为大块内存，大块内存分配直接调用标准C的malloc函数或系统调用mmap，内存池为大块内存维护一个单独的链表，用于统一释放，大块也支持单独释放。
    - 如果小于某个阈值，会先判断当前页剩余的内存大小能否满足本次分配的尺寸要求
        - 如果满足，则简单的移动游标（实际上会考虑对齐要求），并返回移动前的游标位置（地址），这种情况概率高、效率高
        - 如果不满足，再次通过底层接口分一内存页，并从该页划分一块满足本次分配请求，新分配的内存页，会通过链表串起来。
    - 关于当前页指针：
        - 假设当前页还剩500字节，但接下来的分配请求512字节，因为剩余尺寸不够，那么内存池会分配一个4K的新内存页，从新内存页划分出512字节返回，并把新内存页串到内存页链表。
        - 当前页指针还是指向剩余500字节的内存页，内存池下次分配请求依然从旧内存页开始尝试。
        - 只有一个内存页在多次不满足分配尺寸要求后，才会修改内存池的当前页指针，这样做是为了减少内存浪费。因为如果多次尝试后，依然不满足，那么这个内存页的剩余空间大概率会比较小，这时候跳过它，也就合情合理了。
    - 大多数的分配是请求小块内存，这部分高频请求用简单的移动游标满足，性能非常高，因为分配出去的小块内存不支持中途释放，所以它消除了通用内存分配器为每个内存块增加的头部/尾部，提高了有效载荷，内存利用率高。
    - 大块内存的分配是小概率事件，因为大块被单独的链表串起来，所以既支持统一释放，也可以通过遍历链表的方式中途释放大内存块，因为大块数量不多，所以遍历不会很耗费。之所以要支持大块内存的中途释放，是为了避免内存占用过度膨胀（大块内存的一块就可能很大），提高内存复用。

- NGINX内存池小结:
    - 为每个连接配一个内存池，使得无锁成为可能。
    - NGINX内存池专注于做好小块内存的分配，大块的分配只是简单转交给malloc/mmap，它在处理当前内存页指针的细节上做的比较好，提升了内存利用率。
    - 跟一般内存池一样，NGINX内存池通过牺牲小块内存的中途释放能力，换取通过移动游标分配内存的高性能和高内存利用率；通过缓存内存的提升复用，本质上是以空间换时间，这些都体现了TradeOff的思想。
    - NGINX用很少的代码量，达到了很好的实际效果，值得学习借鉴。

#### loki小对象分配器
#### 延伸：对象池

