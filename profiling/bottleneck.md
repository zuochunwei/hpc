# 性能测试

  在一个复杂的系统中，我们经常需要对程序进行性能测试，针对不同的设计方案选择性能更高的方案，那么首先我们需要一个性能测试的工具，除了通过计时工具来对核心代码进行性能采样，在日常开发中，我们可以通过以下工具或方法来精确测试程序的性能瓶颈。



### CPU Clock/ Timer

当我们要对一段程序进行性能测试或者调优时，通常需要通过计时器来记录程序运行的时间。

在Linux平台上有多种计时工具，常见的如`clock`, `gettimeofday`, `clock_gettime`, `std::chrono::system_clock`, `std::chrono::steady_clock`, `std::chrono::high_resolution_clock`, `rdtsc`等等。

在所有的计时工具中，`clock_gettime`计时器本身的开销大概在1ns(实际测量时间与CPU主频有关)。其与`std::chrono::steady_clock`, `std::chrono::high_resolution_clock`, `std::chrono::system_clock`精度接近。但它的稳定性和精度跨平台性最好(C++11标准)。

`clock_gettime`函数原型是

```
int clock_gettime( clockid_t clock_id,struct timespec * tp );
```

其中`clockid_t`时钟类型取值有`CLOCK_REALTIME`,`CLOCK_MONOTONIC`,`LOCK_PROCESS_CPUTIME_ID`,`CLOCK_THREAD_CPUTIME_ID`等。`CLOCK_REALTIME`代表POSIX系统时间自1970-01-01起经历的绝对时间。这个时间会被用户更新时间打断。`CLOCK_MONOTONIC`代表系统单调时间，表示自开机起经历的时间。它不可以被中断。`LOCK_PROCESS_CPUTIME_ID`代表进程执行的时间。`CLOCK_THREAD_CPUTIME_ID`代表线程启动后执行的时间。所以建议使用更稳定的`CLOCK_MONOTONIC`时钟。

`rdtsc`在相同的CPU主频下精度最高，速度最快，稳定性最好，但并非所有CPU均支持 。

`std::chrono::steady_clock`, `std::chrono::high_resolution_clock`,基本一致，它们记录的是相对时间，并且不会因为修改系统时间而受影响。`std::chrono::system_clock`记录的是绝对时间。可能会被用户打断。它们的最小精度都可以到纳秒级别。

在linux平台下，我们的性能测试数据可以采用`std::chrono::high_resolution_clock`的计时方式来测试的。其精度可以满足我们对性能测试的要求。

```c++
#include <chrono>
#include <iostream>

class Timer
{
public:
    void start() { start_time = std::chrono::high_resolution_clock::now(); }
    void end() {
        end_time = std::chrono::high_resolution_clock::now();
        elapse = end_time - start_time;
        std::cout << "time elapse: " << elapse.count() * 1000 << "ms" <<  std::endl;
    }

private:
    std::chrono::time_point<std::chrono::high_resolution_clock> start_time;
    std::chrono::time_point<std::chrono::high_resolution_clock> end_time;
    std::chrono::duration<double> elapse;

};
```

通过计时器只能采样一次，所以测量结果通常容易因测试环境负载变化而产生偏差。并且计时器通常得到的是执行时间，对于不同频率的 CPU 结果可能是完全不同的。



### Google Benchmark

Google Benchmark 是对 C++ 组件进行benchmark 的框架。 对程序中的部分核心代码进行针对性的性能测试，有助于我们学习和理解性能的瓶颈。

在之前我们讨论过 branch 对程序性能的影响，实际上不同的选择率可能会得到不同的性能结果，通过 Google Benchmark 我们可以非常容易的实现不同参数下程序的性能测试。在下面的程序中，我们有两个测试参数，一个是测试数据量的大小，也就是程序中的 size ，另外一个是 branch 的边界，也就是下面的 boundary。下面的程序实现了不同的 size 和 boundary 组合下的性能测试。

```
#include <benchmark/benchmark.h>
#include <vector>
#include <random>
int count_branch(const int *src,
                 size_t count,
                 int bound)
{
    int res = 0;
    for(size_t i = 0; i < count; ++i) {
        if(src[i] >= bound) {
            res++;
        }
    }
    return res;
}

int count_branchless(const int *src,
                     size_t count,
                     int bound)
{
    int res = 0;
    for(size_t i = 0; i < count; ++i) {
        res += (src[i] >= bound);
    }
    return res;
}

static void init_vec(std::vector<int32_t> & vec)
{
    int size = vec.size();
    std::random_device rand_dev;
    std::mt19937       generator(rand_dev());
    std::uniform_int_distribution<int32_t>  distr(-100,100);
    for (size_t i = 0; i < size; i++) {
        vec.push_back(distr(generator));
    }

}

static void bench_count_branch(benchmark::State& state)
{
    int size = state.range(0);
    int boundary = state.range(1);
    std::vector<int32_t> x(size);
    init_vec(x);
    for (auto _: state) {
        count_branch(x.data(), x.size(), boundary);
    }
}
BENCHMARK(bench_count_branch)->Ranges({{1<<10, 1<<20}, {8, 50}});

static void bench_count_branchless(benchmark::State& state)
{
    int size = state.range(0);
    int boundary = state.range(1);
    std::vector<int32_t> x(size);
    init_vec(x);

    for (auto _: state) {
        count_branchless(x.data(), x.size(), boundary);
    }
}
BENCHMARK(bench_count_branchless)->Ranges({{1<<10, 1<<20}, {8, 50}});

BENCHMARK_MAIN();
```

上面的代码在编译选项中通过  -lbenchmark 来指定连接 Google Benchmark 库。Google Benchmark 可以通过 github 下载安装。



下面是执行的结果。首先前面列出了 CPU 的核数、频率信息，CPU Caches 的信息，当前的 Load Average 信息。接的是 Benchmark 执行结果。从输出结果来看，branchless 在相同的数据量下，不同选择率下性能不会有明显的变化，而 branch 则与选择率相关选择率越靠近 50%，性能越差。branchless 在相同的数据量，相同的选择率下可以得到更好的性能。

```
Run on (96 X 2593.91 MHz CPU s)
CPU Caches:
  L1 Data 32 KiB (x96)
  L1 Instruction 32 KiB (x96)
  L2 Unified 1024 KiB (x48)
  L3 Unified 36608 KiB (x2)
Load Average: 0.41, 0.33, 0.25
----------------------------------------------------------------------------
Benchmark                                  Time             CPU   Iterations
----------------------------------------------------------------------------
bench_count_branch/1024/8               4787 ns         4787 ns       145969
bench_count_branch/4096/8              30922 ns        30922 ns        21892
bench_count_branch/32768/8            321406 ns       321403 ns         2186
bench_count_branch/262144/8          2577650 ns      2577624 ns          271
bench_count_branch/1048576/8        10343072 ns     10342944 ns           68
bench_count_branch/1024/50              5003 ns         5003 ns       100000
bench_count_branch/4096/50             23476 ns        23475 ns        29977
bench_count_branch/32768/50           250767 ns       250767 ns         2792
bench_count_branch/262144/50         2014366 ns      2014368 ns          348
bench_count_branch/1048576/50        8084391 ns      8084330 ns           86
bench_count_branchless/1024/8           3716 ns         3716 ns       188545
bench_count_branchless/4096/8          14836 ns        14836 ns        47270
bench_count_branchless/32768/8        118311 ns       118311 ns         5911
bench_count_branchless/262144/8       961045 ns       961046 ns          726
bench_count_branchless/1048576/8     3899768 ns      3899768 ns          179
bench_count_branchless/1024/50          3716 ns         3716 ns       188912
bench_count_branchless/4096/50         14812 ns        14812 ns        47211
bench_count_branchless/32768/50       117920 ns       117920 ns         5898
bench_count_branchless/262144/50      960105 ns       960106 ns          729
bench_count_branchless/1048576/50    3904627 ns      3904629 ns          179
```

Google Benchmark 也可以开启 PMU 计数，需要在编译时开启 *BENCHMARK_ENABLE_LIBPFM* ，在执行时

```
./a.out --benchmark_perf_counters=CYCLES,INSTRUCTIONS,IDQ_UOPS_NOT_DELIVERED:CORE
```

Google Benchmark 的 PMU 计数依赖 pfmlib 库，通过系统调用perf_event_open 实现计数和采样的。



### perf_event_open

perf 是基于 perf_event_open 的性能分析工具。通过 perf_event_open 系统调用分配 perf_event 之后，会返回一个文件句柄 fd，perf_event 数据结果可以通过 read/prctl/ioctl/mmap/fcntl 系统调用接口来操作。

perf_event 提供两种类型的 trace 数据：count 和 sample 。

* count 只是记录了 perf event 的发生次数，

* sample记录了大量信息(比如：IP、ADDR、TID、TIME、CPU、BT)。

如果需要使用sample功能，需要给 perf_event 分配 ringbuffer 空间，并且把这部分空间通过 mmap 映射到用户空间。

perf_event_open 函数可以通过`__NR_perf_event_open`的系统调用实现。定义如下：

```
static long
perf_event_open(struct perf_event_attr *hw_event, pid_t pid,
                int cpu, int group_fd, unsigned long flags)
{
    int ret;
    ret = syscall(__NR_perf_event_open, hw_event, pid, cpu,
                   group_fd, flags);
    return ret;
}
```

perf_event_open 函数调用的参数详细介绍如下：

**perf_event_attr**

perf_event_attr 是内核中的数据结构，提供了要创建的 perf event 的详细配置信息。 主要介绍其中的 type 和 read_format 。

type 字段表示该 perf event 事件的类型。其取值如下：

* PERF_TYPE_HARDWARE: 该事件是内核提供的硬件事件。
* PERF_TYPE_SOFTWARE: 该事件是内核提供的软件定义的事件。
* PERF_TYPE_TRACEPOINT: 该事件是内核跟踪点基础结构提供的跟踪点。
* PERF_TYPE_HW_CACHE: 该事件是硬件高速缓存事件。 在config字段定义中具有特殊的编码。
* PERF_TYPE_RAW: 该事件是 config 字段中发生的“原始”特定于实现的事件。
* PERF_TYPE_BREAKPOINT:该事件是 CPU 提供的硬件断点。 断点可以是对地址的读/写访问，也可以是指令地址的执行。

```
enum perf_type_id { /* perf 类型 */
	PERF_TYPE_HARDWARE			= 0,    /* 硬件 */
	PERF_TYPE_SOFTWARE			= 1,    /* 软件 */
	PERF_TYPE_TRACEPOINT		= 2,    /* 跟踪点 /sys/bus/event_source/devices/tracepoint/type */
	PERF_TYPE_HW_CACHE			= 3,    /* 硬件cache */
	PERF_TYPE_RAW				= 4,        /* RAW/CPU /sys/bus/event_source/devices/cpu/type */
	PERF_TYPE_BREAKPOINT		= 5,    /* 断点 /sys/bus/event_source/devices/breakpoint/type */

	PERF_TYPE_MAX,				/* non-ABI */
};
```

read_format 字段指定 perf_event_open() 返回的文件描述符上通过 read(2) 返回的数据格式。

* `PERF_FORMAT_TOTAL_TIME_ENABLED`添加64位启用时间的字段。 如果 PMU 过量使用并且正在发生多路复用，则可将其用于计算估计总数。
* `PERF_FORMAT_TOTAL_TIME_RUNNING` 添加64位 time_running 字段。 如果 PMU 过量使用并且正在发生多路复用，则可将其用于计算估计总数。
* `PERF_FORMAT_ID` 一个与事件组相对应的64位唯一值。
* `PERF_FORMAT_GROUP` 允许通过一次读取来读取事件组中的所有计数器值。

**pid** 

- 如果pid为0，则在**当前进程**上进行测量；
- 如果pid大于0，则对**pid指示的进程**进行测量；
- 如果pid为-1，则对**所有进程**进行计数。

**cpu**

- 如果`cpu>=0`，则将测量限制为指定的CPU；否则，将限制为0。
- 如果`cpu=-1`，则在所有CPU上测量事件。

需要注意的是`pid == -1`和`cpu == -1`的组合是不允许的。

**group_fd**

group_fd 参数指定 perf_event 的 group_leader：

* group_fd >= 0 指定对于的 perf_event 为当前 group_leader。
* group_fd == -1 创建新的 group_leader。

**flags**

flags默认为0， 有以下几种取值：

* PERF_FLAG_FD_CLOEXEC：该参数表示在事件的文件描述符创建时是能 close-on-exec 标识。文件描述符在调用 execev 时自动关闭。

* PERF_FLAG_FD_NO_GROUP：忽略group_fd参数，除非使用 PERF_FLAG_FD_OUTPUT 进行重定向
* PERF_FLAG_FD_OUTPUT：将事件的采样结果重定向到 group_fd 指定的mmap缓冲区内。（在内核2.6.35后已弃用）
* PERF_FLAG_PID_CGROUP：该标志使能 cgroup 的系统范围内的监控。 cgroup 对CPU，内存等资源以进行更精细的控制 在这种模式下，仅当在受监视的CPU上运行的线程属于指定的容器（cgroup）时，才测量该事件。 

**ioctl**

`ioctl`可以通过 group_fd 启动事件的采样或计数。ioctl 的定义如下：

```
#include <sys/ioctl.h>
int ioctl(int fd, unsigned long request, ...);
```

ioctl 调用常见的取值如下：

* `PERF_EVENT_IOC_ENABLE`启用文件描述符参数指定的单个事件或事件组。如果ioctl参数中的PERF_IOC_FLAG_GROUP 置为 1， 组内的所有事件都将开启。

* `PERF_EVENT_IOC_DISABLE`禁用文件描述符参数指定的单个计数器或事件组。如果ioctl参数中的PERF_IOC_FLAG_GROUP 置为 1， 组内的所有事件都将禁止。

* `PERF_EVENT_IOC_RESET`将文件描述符参数指定的事件计数重置为零。这只会重置计数；无法重置多路复用的time_enabled或time_running值。如果ioctl参数中的PERF_IOC_FLAG_GROUP 置为 1， 组内的所有事件都将重置。

* `PERF_EVENT_IOC_SET_OUTPUT`这告诉内核将事件通知报告给指定的文件描述符，而不是默认的文件描述符。文件描述符必须全部在同一CPU上。

* `PERF_EVENT_IOC_SET_FILTER`（从Linux 2.6.33开始）这将向该事件添加ftrace过滤器。

* `PERF_EVENT_IOC_ID`  （从Linux 3.12开始）表示给定文件描述符返回对应的 event id。

* `PERF_EVENT_IOC_SET_BPF` 表示允许将BPF程序附加到已有的 kprobe tracepoint event。但需要CAP_PERFMON (since Linux 5.8) or CAP_SYS_ADMIN 权限。



**prctl**

prctl 定义如下: 

```
#include <sys/prctl.h>
int prctl(int option, unsigned long arg2, unsigned long arg3,
          unsigned long arg4, unsigned long arg5);
```

prctl（2）系统调用通过 PR_TASK_PERF_EVENTS_ENABLE 和 PR_TASK_PERF/EVENTS_DISABLE 操作启用或禁用所有当前打开的事件组。这仅适用于调用进程创建的事件。这不适用于通过 attach 到其他进程创建的事件或从父进程继承的事件。启用和禁用只对group_leader 有效。



通过以上的介绍，我们可以通过 perf_event_open 采样 PMU 硬件事件的简单示例如下：

```
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/ioctl.h>
#include <linux/perf_event.h>
#include <asm/unistd.h>

static long
perf_event_open(struct perf_event_attr *hw_event, pid_t pid,
                int cpu, int group_fd, unsigned long flags)
{
    int ret;

    ret = syscall(__NR_perf_event_open, hw_event, pid, cpu,
                   group_fd, flags);
    return ret;
}

int
main(int argc, char *argv[])
{
    struct perf_event_attr pe;
    long long count;
    int fd;

    memset(&pe, 0, sizeof(pe));
    pe.type = PERF_TYPE_HARDWARE;
    pe.size = sizeof(pe);
    pe.config = PERF_COUNT_HW_INSTRUCTIONS;
    pe.disabled = 1;
    pe.exclude_kernel = 1;
    pe.exclude_hv = 1;

    fd = perf_event_open(&pe, 0, -1, -1, 0);
    if (fd == -1) {
       fprintf(stderr, "Error opening leader %llx\n", pe.config);
       exit(EXIT_FAILURE);
    }

    ioctl(fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(fd, PERF_EVENT_IOC_ENABLE, 0);

    printf("Measuring instruction count for this printf\n");

    ioctl(fd, PERF_EVENT_IOC_DISABLE, 0);
    read(fd, &count, sizeof(count));

    printf("Used %lld instructions\n", count);

    close(fd);
}
```

