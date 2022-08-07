## 定位系统瓶颈

### 常用linux命令

1. Top，top通常是查看系统负载，CPU利用率，进程内存使用情况最常使用的工具

2. vmstat CPU使用率，内存使用，虚拟内存交换，IO使用率

3. iostat IO统计

4. sar: 

   sar  -P 0(cpu编号)  

   sar -r

   sar -B 内存 sar -b io

   



### 定位性能常用工具

#### perf

在root用户下执行perf top命令，可以看到当前所有进程的热点函数

perf stat –e <event list>

perf record –e <event list>

#### FlameGraph



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



#### 定位内存泄漏

##### 1. valgrind

用于jemalloc的内存泄漏检查，常用检查项如

memcheck

massif

##### 2. gperftools

CPUprofiler

Heap Profiler

##### 3. asan检查

编译选项

GCC ：-fsanitize=address：开启内存越界检测



参考文档：

https://brendangregg.com/