Perf 

在root用户下执行perf top命令，可以看到当前所有进程的热点函数





FlameGraph

火焰图是常使用的分析性能的工具。通常的使用方式如下

```
1. 通过perf record 选择需要记录的perf events, --pid指定perf监控的端口号，-o表示输出的文件
perf record -ag  --pid 23250 -o perf.data

2.通过FlameGraph工具生成火焰图
perf script -i perf.data &> perf.unfold
./stackcollapse-perf.pl perf.unfold &> perf.folded
./flamegraph.pl perf.folded > perf.svg
```

* Perf_events包括以下几类：通过perf list可以查看
* 硬件事件：CPU性能监控计数
* 软件事件：基于Kernel技术的低级别事件，比如CPU migrations, minor faults, major faults,等

- 内核追踪事件：内核级别的插桩点
- 用户级别定义事件
- 动态追踪: Software can be dynamically instrumented, creating events in any location. For kernel software, this uses the kprobes framework. For user-level software, uprobes.
- 时间采样: Snapshots can be collected at an arbitrary frequency, using `perf record -F*Hz*`. This is commonly used for CPU usage profiling, and works by creating custom timed interrupt events.