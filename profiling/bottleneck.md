# 性能测试

1. Google Benchmark

在一个复杂的系统中，我们经常需要对程序进行性能测试，针对不同的设计方案选择性能更高的方案，那么首先我们需要一个性能测试的工具，除了通过计时工具来对核心代码进行性能采样，但通过计时器只能测样一次，所以测量结果通常容易因环境变化而产生偏差。Google Benchmark可以对程序中的核心代码进行针对性的性能测试，有助于我们学习和理解性能的瓶颈。



在之前我们讨论过branch对程序性能的影响，实际上不同的选择率可能会得到不同的性能结果，通过Google Benchmark我们可以非常容易的实现不同参数下程序的性能测试。在下面的程序中，我们有两个测试参数，一个是测试数据量的大小，也就是程序中的size，另外一个是branch的边界，也就是下面的boundary。下面的程序实现了不同的size和boundary组合下的性能测试。

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

上面的代码在编译选项中通过 -lbenchmark来指定连接Google Benchmark库。Google Benchmark可以通过github下载安装。



下面是执行的结果。首先前面列出了CPU的核数、频率信息，CPU Caches的信息，当前的Load Average信息。接的是Benchmark执行结果。从输出结果来看，branchless在相同的数据量下，不同选择率下性能不会有明显的变化，而branch则与选择率相关选择率越靠近50%，性能越差。branchless在相同的数据量，相同的选择率下可以得到更好的性能。

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

