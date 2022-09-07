## 定位系统瓶颈

新的系统或者功能在上线前一般会遇到各种各样的瓶颈，或是BUG,内存泄漏抑或是性能瓶颈，这就需要我们能够了解整个系统的运行状况并快速定位系统的瓶颈。俗话说：工欲善其事，必先利其器。linux提供了丰富的性能定位分析工具，比如perf，ptrace，eBPF等，除此之外还有丰富的内存泄漏定位工具可供使用，比如valgrind,Sanitizer,gperftools等。

### 系统监控

1. 查看系统配置

* lscpu

Linux操作系统下，cpu相关的信息一般存在于proc文件夹下面的cpuinfo文件中。通过cat /proc/cpuinfo可以查看。更常用的命令是lscpu。在lscpu中我们可以看到CPU的Architecture表示架构信息是x86_64架构，CPU(s)共96个核，NUMA nodes表示2个numa组，其中0-47个核属于numa0，48-95核属于numa1。CPU MHz表示CPU的主频，L1d cache，L1i cache,表示L1cache的大小，L1cache分别指令缓存和数据缓存。L1/ L2 cache为各个CPU独自拥有的。L3 cache是每个numa组内共享的，Flags表示支持的ISA指令集等等信息。

```
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              96
On-line CPU(s) list: 0-95
Thread(s) per core:  2
Core(s) per socket:  24
Socket(s):           2
NUMA node(s):        2
Vendor ID:           GenuineIntel
CPU family:          6
Model:               85
Model name:          Intel(R) Xeon(R) Platinum 8361HC CPU @ 2.60GHz
Stepping:            11
CPU MHz:             2593.906
BogoMIPS:            5187.81
Hypervisor vendor:   KVM
Virtualization type: full
L1d cache:           32K
L1i cache:           32K
L2 cache:            1024K
L3 cache:            36608K
NUMA node0 CPU(s):   0-47
NUMA node1 CPU(s):   48-95
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl cpuid tsc_known_freq pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single pti fsgsbase bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 avx512_bf16 arat avx512_vnni
```

* free

内存及IO相关的系统一般存在于proc文件夹下面的meminfo文件中，通过cat /proc/meminfo可以查看。更常用的命令是free -h，free中可以看到total代表总的内存大小，used代表已使用的内存大小，buffer/cache的内存大小，available的内存大小。可以发现total的大小等于free的内存加上buff/cache的内存大小再加上used的大小。这里的buff/cache包括的是page cache和buffer cache的大小之和。page cache用作文件系统的文件缓存。比如进程对文件read/write操作或者mmap调用时使用。buffer cache也是block cache是用作系统对块设备读写时的缓存。在Linux内核中通过/proc/sys/vm/drop_caches来清理buff/cache。可以通过echo 3>/proc/sys/vm/drop_caches来清理buff/cache的内存。需要注意的是tmpfs的内存占用也是显示在buff/cache中的。swap表示的是交换内存的信息，如果未开启，则显示的都是0。

```
              total        used        free      shared  buff/cache   available
Mem:          278Gi        13Gi       134Gi       1.0Gi       129Gi       262Gi
Swap:            0B          0B          0B
```

* top

​		top通常是查看系统负载，CPU利用率，进程内存使用情况最常使用的工具。Load average中的三个数字分别代表1分钟，5分钟，15分钟的正在运行或者等待运行的进程个数。top -u user可以查看某个用户的进程信息。top -p pid查看某个进程的信息。如果想要查看某个cpu的信息，可以使用mpstat -P coreid来查看。

top 中最常关注的还有RSS，一般指的是物理内存的占用情况。RSS越大就证明进程的内存占用越多，如果持续增长有可能是内存泄漏的问题。

* vmstat

​	vmstat也是监控系统的常用指令之一。vmstat中包括了 CPU使用，内存使用，虚拟内存交换swap，IO使用情况，每秒上下文切换cs等信息。其中cs代表线程切换时进程上下文切换的次数。cs越大表示cs对性能的影响越大，可能是影响系统性能的瓶颈。vmstat常用于分析进程上下文切换对性能的影响。

对于context switch，可以通过pidstat -p ${pid} -w 命令在查看指定进程的内存上下文切换情况。pidstat命令中cswch 与 nvcswch 是重点关注的对象。cswch 表示每秒自愿上下文切换（voluntary context switches）的次数，nvcswch 表示每秒非自愿上下文切换（non voluntary context switches）的次数。

所谓自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。

而非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

* iostat

iostat输出了 IO统计信息, iostat -c输出CPU信息，包括user, nice, system,iowait. Steal, idle信息，iostat -d {sda,sdb}显示设备信息，iostat -d -p {sda, sdb} 显示设备分区信息，设备信息提供每个物理设备或分区的统计信息。iostat -x 命令可以查看IO utils，观察IO是否达到了系统瓶颈，如果IO一直处于较高的利用率，可以考虑使用RAID磁盘阵列或者更换更快的SSD盘来提升IO的吞吐。

**user**
进程在用户地址空间中消耗 CPU 时间的百分比。像 shell 程序、各种语言的编译器、数据库应用、web 服务器和各种桌面应用都算是运行在用户地址空间的进程。这些程序如果不是处于 idle 状态，那么绝大多数的 CPU 时间都是运行在用户态。

**system**
进程在内核地址空间中消耗 CPU 时间的百分比。所有进程要使用的系统资源都是由 Linux 内核处理的。当处于用户态(用户地址空间)的进程需要使用系统的资源时，比如需要分配一些内存、或是执行 IO 操作、再或者是去创建一个子进程，此时就会进入内核态(内核地址空间)运行。事实上，决定进程在下一时刻是否会被运行的进程调度程序就运行在内核态。对于操作系统的设计来说，消耗在内核态的时间应该是越少越好。在实践中有一类典型的情况会使 sy 变大，那就是大量的 IO 操作，因此在调查 IO 相关的问题时需要着重关注它。

**iowait**
CPU 等待磁盘 IO 操作的时间。和 CPU 的处理速度相比，磁盘 IO 操作是非常慢的。有很多这样的操作，比如：CPU 在启动一个磁盘读写操作后，需要等待磁盘读写操作的结果。在磁盘读写操作完成前，CPU 只能处于空闲状态。Linux 系统在计算系统平均负载时会把 CPU 等待 IO 操作的时间也计算进去，所以在我们看到系统平均负载过高时，可以通过 iowait 来判断系统的性能瓶颈是不是过多的 IO 操作造成的。

**idle**
CPU 处于 idle 状态的百分比。一般情况下， user + nice + idle 应该接近 100%。



### 定位性能常用工具

系统性能中最常用的两个指标是吞吐量和延时。对于服务器而言，延时代表的是一次访问所需要的耗时。而吞吐代表的是一段时间内的访问的总次数。一般来说，延时越低，吞吐量越大，系统的性能就越好。

#### perf

预备知识：PMU,PMC

PMU,即perf monitor unit，性能监控单元。每个 PMU 模型包含一组寄存器：性能监视配置 (PMC) 和性能监视数据 (PMD)。这两个寄存器都是可读的，但只有 PMC 是可写的。这些寄存器用于储存配置信息和数据。

Perf,也叫perf events，或perf tools,是Linux内核提供的性能分析工具。perf是基于perf_event_open的内核系统调用。支持硬件性能计数(HPC),软件性能计数，kprobe，uprobe, tracepoints等多种事件。以PMC为例，perf执行的原理如下：

1. 通过sys_perf_event_open()的系统调用向内核注册一个PMC的计数器并在内核中通过mmap申请内存用于保存采样结果。
2. PMC通过专用的寄存器随CPU cycles的增长而累加，当PMC溢出时，PMU会产生一个PMI硬件中断。
3. 在中断函数中完成一次采样：采样信息包括HPC计数值，触发中断的指令地址，时间戳，PID等信息。并存入perf_event_open()申请的内存中
4. perf通过read读取采样信息，根据PID等信息找到对应进程，并根据进程ELF符号表解析为对应的函数调用信息。



Perf events：Perf_events包括以下几种类型：

* 硬件事件：PMU产生的CPU性能监控计数
* 软件事件：基于Kernel技术的事件，比如CPU migrations, minor faults, major faults,等

- 内核追踪事件：内核级别的插桩点
- 动态追踪: 包括内核态kprobe和用户态的uprobe事件。
- 时间采样: perf可以指定采用频率，通过 `perf record -F*Hz*`命令。



通过perf list可以查看当前perf支持的perf events。

```
List of pre-defined events (to be used in -e):

  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
  cpu-migrations OR migrations                       [Software event]
  dummy                                              [Software event]
  emulation-faults                                   [Software event]
  major-faults                                       [Software event]
  minor-faults                                       [Software event]
  page-faults OR faults                              [Software event]
  task-clock                                         [Software event]

  duration_time                                      [Tool event]

  msr/tsc/                                           [Kernel PMU event]

  rNNN                                               [Raw hardware event descriptor]
  cpu/t1=v1[,t2=v2,t3 ...]/modifier                  [Raw hardware event descriptor]
   (see 'man perf-list' on how to encode it)

  mem:<addr>[/len][:access]                          [Hardware breakpoint]
  
  ......
  sched:sched_stat_blocked                           [Tracepoint event]
  sched:sched_stat_iowait                            [Tracepoint event]
  sched:sched_stat_runtime                           [Tracepoint event]
  sched:sched_stat_sleep                             [Tracepoint event]
  sched:sched_stat_wait                              [Tracepoint event]
  skb:consume_skb                                    [Tracepoint event]
  skb:kfree_skb                                      [Tracepoint event]
  skb:skb_copy_datagram_iovec                        [Tracepoint event]
  ......
```

以上是perf version 5.4.119版本的输出结果，从中我们可以看到Software event，比如常用的cpu-clock, context-switches, page-faults, Hardware breakpoint, 比如mem, Tracepoint event 比如常见的sched, skb等等。不同版本的perf event可能不一样，如果发现某些events不存在可以通过升级perf来解决。

在root用户下执行perf 命令，可以看到指定-e 来指定需要跟踪的perf events，使用方式如下

```
perf top -e <event list>
perf stat –e <event list>
perf record –e <event list>
```



#### FlameGraph

perf提供了丰富的采用工具，但对于性能分析来说不够直观，而火焰图是更常使用的分析性能的工具。火焰图又叫做[FlameGraph](https://github.com/brendangregg/FlameGraph)，是由Brendan Gregg开源的一个性能分析工具。火焰图可以从横向和纵向两个方向来分析，

纵向表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。

横向表示不同的采样信息，如果一个函数在横向占据的宽度越宽，就表示它的采样次数越多。

**火焰图就是看最上层的哪个函数占据的宽度最大。宽度越大，就表示该函数在整个性能采样中占用的时间最长，更可能是系统的瓶颈点，需要重点关注。**

火焰图的使用方式如下

```
1. 通过perf record 选择需要记录的perf events，采集生成perf.data, --pid指定perf监控的端口号，-o表示输出的文件 -a表示所有cpu， -g表示加入调用堆栈信息
perf record -ag  --pid 23250 -o perf.data

2.通过FlameGraph工具生成火焰图
perf script解析采用的perf.data生成堆栈信息
perf script -i perf.data &> perf.unfold

3. 解析perf script生成的堆栈信息列表，输出逗号分隔的堆栈即count计数。
./stackcollapse-perf.pl perf.unfold &> perf.folded

4. 输出为svg图
./flamegraph.pl perf.folded > perf.svg
```

on-cpu & off-cpu

```x86asm
On-CPU：线程在 CPU 上运行的时间。
Off-CPU：计时器、分页/交换等上阻塞时所花费的时间。
```

Off-CPU 是一种用于测量和研究 CPU时间之外以及上下文堆栈跟踪的性能分析方法。它不同于 On-CPU 仅分析在 CPU 上执行的线程。它的目标是分析被阻止线程状态，Off-CPU 外分析是对 On-CPU 分析的补充。此方法也不同于应用程序阻塞的跟踪技术，因为此方法针对内核调度程序被阻塞原因的分析，比应用程序阻塞分析的应用场景更广。

造成线程Off-CPU 的原因有很多，包括 I/O 和锁，但也有一些与当前线程的执行无关的原因，包括由于对 CPU 资源的高需求而导致的非自愿上下文切换和中断。无论出于什么原因，如果在工作负载请求（同步代码路径）期间发生这种情况，则造成延迟。





#### BPF

BPF,eBPF,BCC

BPF全称是**Berkeley Packet Filter**，翻译过来是**伯克利包过滤器**，BPF诞生在1992年，最初是为了解决 Unix 内核实现网络数据包过滤的问题。BPF的主要原理是在内核中设计了一个新的BPF虚拟机可以有效地工作在基于寄存器结构的 CPU 之上，应用程序使用缓存只复制与过滤数据包相关的数据，不会复制数据包的所有信息，最大程度地减少BPF 处理的数据，提高处理效率。常用的抓包工具tcpdump就是基于BPF技术实现的。

eBPF全称是**enhanced Berkeley Packet Filter**。eBPF的原理是用户程序中编译生成BPF字节码指令，并加载到 BPF JIT 模式的虚拟机中，在虚拟机中将字节码指令转成内核可执行的本地指令运行。eBPF提供可基于系统或程序事件高效安全执行特定代码的通用能力，且具有很高的执行效率。其使用场景不再仅仅是网络分析，可以基于eBPF开发性能分析、系统追踪、网络优化等多种类型的工具和平台。

BCC全称是**BPF Compiler Collection**，是python封装的eBPF工具集。基于 BCC 实现的各种跟踪 BPF 程序，可以查看到必要的内核的内部结构。BCC 提供了内置的 Clang 编译器，可以在运行时编译 BPF 代码，以实现运行在目标主机上的特定内核中。

在BCC中有很多工具可以使用，比如常见的off-cpu定位工具，可以得到off-cpu的火焰图。

```
#/usr/share/bcc/tools/offcputime -df -p `pgrep -nx mysqld` --state=2 30 > out.stacks
# /usr/share/bcc/tools/offcputime -df -p `pgrep -nx mysqld` 30 > out.stacks
[...copy out.stacks to your local system if desired...]
# git clone https://github.com/brendangregg/FlameGraph
# cd FlameGraph
# ./flamegraph.pl --color=io --title="Off-CPU Time Flame Graph" --countname=us < out.stacks > out.svg
```

![image-20220826105932146](./pic/3.png)

#### 定位内存泄漏

##### 1. valgrind

*Valgrind*是一款用于内存调试、内存泄漏检测以及性能分析的软件开发工具。它的原理是**Valgrind** 通过运行时软件翻译二进制指令的执行获取相关的信息。所以valgrind在定位问题时性能会有大幅的下降。常用检查项如

（1）Memcheck。这是valgrind应用最广泛的工具，一个重量级的内存检查器，能够发现开发中绝大多数内存错误使用情况，比如：使用未初始化的内存，使用已经释放了的内存，内存访问越界等。

（2）Callgrind: 检查程序中函数调用过程中出现的问题。

（3）Cachegrind: 检查程序中缓存使用出现的问题。

（4）Helgrind: 检查多线程程序中出现的竞争问题。

（5）Massif: 检查程序中堆栈使用中出现的内存泄漏问题。

valgrind的使用方法如下

```
valgrind --tool=memcheck --leak-check=full ./main
```

##### 2. gperftools

gperftools是Google开源的性能分析工具

（1）CPU profiler：捕捉程序运行时的CPU profilter信息

（2）Heap Profiler：捕捉程序运行时出现的堆栈内存使用信息

（3）Heap Checker：捕捉程序运行时出现的堆栈内存泄漏问题

（4）TCMalloc：thread-cached 内存管理工具。

（5）pprof : 用于生成可视化和分析堆栈图

gperftool的使用方法如下

```
LD_PRELOAD=/usr/local/lib/libprofiler.so CPUPROFILE=test.prof ./main

pprof -gv ./main test.prof
```

##### 3. **Sanitizer**

**Sanitizer** 则是通过编译时插入代码来捕获相关的信息，性能下降幅度比 Valgrind 小很多。大概在2-4倍性能下降。

LLVM 以及 GNU C++ 有多个 Sanitizer：

- AddressSanitizer（ASan）可以发现内存错误问题，比如 use after free，heap buffer overflow，stack buffer overflow，global buffer overflow，use after return，use after scope，memory leak，super large memory allocation；
- AddressSanitizerLeakSanitizer （LSan）可以发现内存泄露；
- MemorySanitizer（MSan）可以发现未初始化的内存使用；
- UndefinedBehaviorSanitizer （UBSan）可以发现未定义的行为，比如越界数组访问、数值溢出等；
- ThreadSanitizer （TSan）可以发现线程的竞争行为；

Sanitizer在使用时，只需要在编译时添加如下编译选项即可。

```
-fsanitize=address -fsanitize=undefined -fno-sanitize-recover=all -fsanitize=float-divide-by-zero -fsanitize=float-cast-overflow -fno-sanitize=null -fno-sanitize=alignment
```

在上述编译选项中，-fsanitize的常见取值有`address,memory,undefined,thread`，但`-fsanntize=memory`和`-fsantize=address`不能同时使用。

下面的程序中存在use-after-free的内存错误问题，在编译时加入-fsanitize=address表示开启ASAN检查，从而得到a.out输出文件。

```
% cat use-after-free.c
#include <stdlib.h>
int main() {
  char *x = (char*)malloc(10 * sizeof(char*));
  free(x);
  return x[5];
}

%compile
clang -fsanitize=address -fno-omit-frame-pointer -g use-after-free.c
```

通过执行a.out文件，可以得到如下的报错信息。可以看到AddressSanitizer检测出heap-use-after-free的内存问题。heap-use-after-free问题表示堆内存释放后再次访问，此时可能访问到空指针或者脏数据。报错信息接下来依次是READ堆栈，内存释放堆栈，内存申请堆栈，通过以上堆栈信息，我们可以得到heap-use-after-free发生在use-after-free.c的第五行。

```
=================================================================
==3666426==ERROR: AddressSanitizer: heap-use-after-free on address 0x607000000025 at pc 0x0000004cc9a3 bp 0x7ffce22c95c0 sp 0x7ffce22c95b8
READ of size 1 at 0x607000000025 thread T0
    #0 0x4cc9a2 in main /tests/use-after-free.c:5:10
    #1 0x7f19470e8f92 in __libc_start_main (/lib64/libc.so.6+0x26f92)
    #2 0x41e449 in _start (/data/ljw/temp/a.out+0x41e449)

0x607000000025 is located 5 bytes inside of 80-byte region [0x607000000020,0x607000000070)
freed by thread T0 here:
    #0 0x49a992 in free (/tests/a.out+0x49a992)
    #1 0x4cc965 in main /tests/use-after-free.c:4:3
    #2 0x7f19470e8f92 in __libc_start_main (/lib64/libc.so.6+0x26f92)

previously allocated by thread T0 here:
    #0 0x49abfd in __interceptor_malloc (/data/ljw/temp/a.out+0x49abfd)
    #1 0x4cc958 in main /tests/use-after-free.c:3:20
    #2 0x7f19470e8f92 in __libc_start_main (/lib64/libc.so.6+0x26f92)
```

需要注意的是ASAN符号解析时需要用到llvm-symbolizer，否则报错信息只有地址信息没有符号信息。可以通过安装llvm后指定环境变量ASAN_SYMBOLIZER_PATH来解决。





参考文档：

https://brendangregg.com/

https://github.com/google/sanitizers/wiki/AddressSanitizer