# 性能测试

## Google Benchmark

在一个复杂的系统中，我们经常需要对程序进行性能测试，针对不同的设计方案选择性能更高的方案，那么首先我们需要一个性能测试的工具，除了通过计时工具来对核心代码进行性能采样，但通过计时器只能测样一次，所以测量结果通常容易因环境变化而产生偏差。Google Benchmark 可以对程序中的核心代码进行针对性的性能测试，有助于我们学习和理解性能的瓶颈。



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

Google Benchmark 的 PMU计数是通过pfmlib库实现的。



## Perf

perf 是基于 perf_event_open 的性能分析工具，所以我们可以通过选取perf event，通过perf_event_open的系统调用采集PMU数据。perf先从proc文件系统中获取内核支持的perf event，然后使用系统调用perf_event_open和ioctl控制内核中perf_event的使能，并从文件描述符中读取数据。

perf_event_open函数定义如下：

```
static int perf_event_open(struct perf_event_attr *evt_attr, pid_t pid,
                                int cpu, int group_fd, unsigned long flags)
{
    int ret;
    ret = syscall(__NR_perf_event_open, evt_attr, pid, cpu, group_fd, flags);
    return ret;
}
```

