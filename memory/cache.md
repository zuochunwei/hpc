## CACHE
Cache并非一开始就有，在286时代，没有Cache，CPU和内存都很慢，CPU直接访问内存，但随着CPU技术的发展，CPU越来越快，时钟频率能够达到数G，一个简单操作能在一个时钟周期内（0.x纳秒）完成，而内存的访问时延达到50纳秒，两者之间有巨大的速度差距，因而内存访问速度成为制约系统性能的瓶颈。

Cache介于CPU与内存之间，Cache是存储子系统的一个部件，存放着程序经常/最近访问的指令和数据，Cache的出现填充了CPU和内存之间的速度鸿沟，掩盖了CPU对内存的访问延时，访问cache的速度比访问内存快，但cache容量有限，单位成本相比内存更高，cache不是为了替代内存，而是被设计用来作为内存的高速缓存。

根据局部性原理，当前被访问数据的临近数据有可能在接下来的时间里被访问，当前访问的数据很可能在接下来的时间里被再次访问，所以，虽然cache不足以装下整个内存数据，得益于程序的局部性行为，CPU对内存的访问很多会命中cache，更快的获得数据，从而获得性能的提升。

广义上，用于虚拟地址转换的TLB（缓存MMU常用页表项）、MOB（Memory Ordering Buffers)、指令流水线中的ROB、甚至寄存器文件也是一种cache，但狭义上的cache主要指L1/L2/L3 Cache。

cache对应用程序员是透明的，应用程序员无须关心数据是在内存还是cache中，无须关心CPU如何处理cache一致性，系统底层全权负责处理好这些细节，cache悄无声息的工作着，但理解cache及其工作原理，编写对Cache友好的程序，对性能敏感的软件（例如数据库内核/人工智能引擎）非常必要。

- 为什么需要CACHE比内存快
因为内存的物理介质是DRAM（Dynamic RAM），而CACHE的物理介质是SRAM（Static RAM），SRAM的传输速度高、延迟低、密度低、成本高。
L1-L2 CACHE每个核心一个，直接做到了CPU Core里，跟Core的物理距离短，也是Cache快的原因之一。
L1指令Cache（ICache）通常是放在CPU核心的指令预取单远附近的，L1数据Cache（DCache）通常是放在CPU核心的load/store单元附近。

- 三级CACHE结构：L1-L2 CPU内core独占，L3 CPU内Core共享
    - 386时代，首次出现Cache，也就是L1-Cache的雏形，而当时的cache内容更新都会直接写回内存，即Write-Through策略。
    - 486时代，Intel在CPU里加入了8KB的L1-Cache（内部cache），该cache叫unified-cache，不区分代码和数据，且增加了Write-Back策略，即Cache内容更新不立即写回内存。
    - 586时代，L1-Cache被一分为二，分为指令cache和数据cache。
    - 后来，随着多CPU多Core技术的出现和发展，L3-Cache也被加入了Cache层级。
    - L1/L2被设计为每个Core独占，L2也叫Middle Level Cache，L3被设计为核间共享，是跨CPU Core的，也被称为Last-Level-Cache。
    - 现代计算机系统中，L1 Cache一般几十k，L2 Cache几百K，而L3 Cache容量高达数兆。

- cache对性能的影响
	- 访问延迟对比
        - 寄存器/MBO：1 cycle， < 1ns（纳秒）
        - L1 cache：3 cycle，~= 1ns
        - L2 cache：12 cycle，~= 3ns
        - L3 cache: 40 cycle, ~= 12ns
        - DRAM（内存）：100 cycle，~=50ns
    可见，寄存器和MBO最快，L1的时延跟CPU Core在一个数量级，内存的访问时延跟L1差了接近2个数量级。
	- CacheLine
        - Cache以固定大小为单位（Cache Entry）存储数据，这个单位叫Cache Line，一般是64字节
        - CPU和Cache之间按字长（Word）为单位传输数据，64位系统上字长是64位，32位系统上字长是32位
        - Cache和内存之间是按Cache Line为单位传输数据
        - 内存跟磁盘间传输数据是按IO Block为单位，一般是4096字节
        - 数据按Cache Line对齐对程序性能有好处，假设CACHE从内存里取一个Cache Line，但只访问其中的4个字节，然后因为cache容量空间有限，随后对新Cache Line的加载，导致旧的Cache Line从Cache中驱逐，这样的话，实际上只读取其中的4字节，却需要加载整个Cache Line，从而导致内存带宽的浪费，这是需要努力避免的。
        - 考虑另一种情况，如果一个8字节的数据，被放置在2个相邻的Cache Line，则对该数据的访问，讲导致2条Cache Line被加载。
	- Cache替换   
        - Cache中存放的是内存数据的一个拷贝，Cache容量相比内存小很多，当数据装满之后，如果需要存入新数据，就需要淘汰旧的数据条目，给新条目腾出空间，这个过程叫Cache驱逐。
        - 缓存管理单元通过一定的算法决定哪些数据留在Cache里，哪些数据被淘汰，这个策略叫替换策略，最简单的替换策略是LRU（Least Recently Used），实际使用的策略一直在不断演进。
    - Cache hit/miss
        - 因为Cache的引入，CPU需要读取一个地址的时候，会先去Cache中查找，如果需要访问的数据在Cache中，叫Cache命中（cache hit）
        - 如果数据不在Cache中，称为Cache命失（cache miss），这时候，就需要从内存中把这个地址所在的那个Cache line上的数据加载到Cache中，然后再把数返回给CPU。
        - 针对写操作，有两种写入策略：write back和write through。
            - write through策略：数据直接同时被写入到Memory中。
            - write back策略：数据仅写到Cache中，此时Cache中的数据与Memory中的数据不一致，Cache中的数据就变成了脏数据(dirty)。
            - 如果其他部件（DMA， 另一个核）访问这段数据的时候，就需要通过Cache一致性协议(Cache coherency protocol)保证取到的是最新的数据。
            - Cache替换的时候，Cache Line被驱逐替的时候需要写回到内存中。
    - Cache Miss与CPU Stall
        - 一旦Cache miss发生，就需要从内存取数，因为内存的访问时延多达几十纳秒，这个时间足够CPU执行几十上百条指令，这个时候如果CPU什么也不做干等着的话，就浪费了宝贵的计算资源。
        - CPU Stall：CPU执行的时候，所需的数据不在寄存器和CACHE中，而需要去内存加载，加载过程中造成CPU停顿（无事可做）的现象叫CPU Stall。
        - CPU Stall主要由数据依赖、资源竞争造成，前述的访存停顿属于典型的CPU Stall，也叫Memory Stall，另外，由于执行资源的不可用，比如功能单元、指令buffer、寄存器竞争也会导致CPU Stall。
        - 有多种优化CPU使用效率的技术被广泛采用：
            - 乱序执行（out of order execution）指当前执行流中不依赖当前指令执行结果的后续指令被提前执行的情况。如果指令X执行时停顿了，指令X之后的指令Y的输入，如果不依赖于指令X的执行结果，那么指令Y可能先于指令X被执行，分发/执行单元负责识别出这样依赖，并调度指令执行。
            - 分支预测：算法预测分支结果，并取合适的指令提前执行，如果预测成功，则继续执行，如果预测失败，则重刷指令流水线，retire单元会删除错误结果，错误的预测会引起额外计算开销并增加停顿时间。
            - 超线程技术，即把另一个线程的指令拿过来执行。
- Load/Store
    - CPU从内存中读取数据到cache line的操作被称为"load"
    - cache line中的数据写回到内存对应位置的操作则被称为"store"
    - CPU通过总线与内存相连，所以Load和Store都需要经过总线，总线被多个CPU所共享

- Cache一致性
    - 单核系统的Cache一致性指内存与Cache之间的数据一致性问题。

    - 随着多CPU多Core已经成为主流硬件，而每个Core有独立的L1-L2级Cache，所以，一个数据行，有可能被同时加载到多个core的Cache中，一个数据在多个地方被缓存，因此多核系统的“cache一致性”既包括Cache和内存之间的一致性，还包括各个CPU的各Cache之间的一致性，也就是说，对内存同一位置的数据，不同CPU的Cache Line不应该有不同(不一致)的值。
    
    - 如果是多个core上并行/并发执行的执行流，只是读内存数据，倒没什么问题，如果其中一个core上执行的线程，对数据进行修改，则这个修改将需要反应到内存上，并将这个信息同步给其他Core上的缓存，不然，就会引起不一致问题，这就是多核系统的Cache一致性问题。

    先看看CACHE怎么引入不一致：

    - 上下级cache之间。例如，CPU core将值写入了L1 cache，然而内存中对应的数据并没有得到更新。这种情况一般出现在单进程的程序控制其他非CPU的agent的情况。解决这种问题的思路一般是程序显式或者隐式地清空相应的cache。显式的途径包括发flush/invalidate命令，隐式的途径一般是直接将对应的内存空间标记为 non-cache，或者write-through.
    
    - 同一级cache之间。例如，在一个4核处理器中，每个core都拥有自己的L1 cache。那么 一份数据可能在这4个L1 Cache中都有相应的拷贝。这种情况一般出现在多线程的程序 设计中。这种情况下，多线程程序需要显式提供必须的信息，以帮助SMP正确执行程序。 这也是本篇文章关注的焦点。

    - 多核把一个内存数据加载到各自的Cache，然后其中一个核修改了该数据，其他核的cacheline的对应数据将被更新为新的值，这个机制叫写传播（Write Propagation），最常见的实现方式就是总线嗅探（Bus Snooping）。

    - 某个CPU核心里对数据的操作顺序，必须在其他核心看起来顺序是一样的，这个称为事务的串形化（Transaction Serialization），有一个协议（MESI）基于总线嗅探机制实现了事务串行化。
    
    - MESI就是CPU上采用的缓存一致性协议，它将CPU中每个缓存行标记为M(Modified)、E(Exclusive)、S(Shared)、I(Invalid)四种状态之一。

    - 对应程序员而言，重要的需要理解的一点就是，如果某个core上的线程修改了某个内存数据，则需要通过总线发送消息，通知其他核心，其他核心监听到这个消息，从而把对应的CacheLine标注为无效，以反应该内存数据已经被其他核心修改的事实，从而再次从内存中加载数据到缓存。
    
    - 如果一个Cache Line中的2个数据，分别被不同核心修改，则会导致频繁的核间消息传递，这就是伪共享导致的性能低下问题，可以通过在2个数据之间插入无效数据，让这2部分数据，分布到不同的cache line，从而避免频繁的缓存失效/更新。
    
    - NUMA系统中，不同node的CPU之间不再是通过共享的总线相连，而是通过基于消息传递(message‐passing)的interconnect相连。

- 分析和测量CACHE命中率

- 编写CACHE友好代码

- linux命令：lscpu可以查看机器的L1-2-3级缓存信息

