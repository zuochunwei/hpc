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

预备知识：PMU,PMC

PMU,即perf monitor unit，性能监控单元。每个 PMU 模型包含一组寄存器：性能监视配置 (PMC) 和性能监视数据 (PMD)。这两个寄存器都是可读的，但只有 PMC 是可写的。这些寄存器用于储存配置信息和数据。

Perf,也叫perf tools，或perf tools,是Linux内核提供的性能分析工具。perf是基于perf_event_open的内核系统调用。支持硬件性能计数(HPC),软件性能计数，kprobe，uprobe, tracepoints等多种事件。以PMC为例，perf执行的原理如下：

1. 通过sys_perf_event_open()的系统调用向内核注册一个PMC的计数器并在内核中通过mmap申请内存用于保存采样结果。
2. PMC通过专用的寄存器随CPU cycles的增长而累加，当PMC溢出时，PMU会产生一个PMI硬件中断。
3. 在中断函数中完成一次采样：采样信息包括HPC计数值，触发中断的指令地址，时间戳，PID等信息。并存入perf_event_open()申请的内存中
4. perf通过read读取采样信息，根据PID等信息找到对应进程，并根据进程ELF符号表解析为对应的函数调用信息。

Perf list

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
```

* Perf_events包括以下几类：通过perf list可以查看
* 硬件事件：PMU产生的CPU性能监控计数
* 软件事件：基于Kernel技术的事件，比如CPU migrations, minor faults, major faults,等

- 内核追踪事件：内核级别的插桩点
- 用户级别定义事件
- 动态追踪: Software can be dynamically instrumented, creating events in any location. For kernel software, this uses the kprobes framework. For user-level software, uprobes.
- 时间采样: Snapshots can be collected at an arbitrary frequency, using `perf record -F*Hz*`. This is commonly used for CPU usage profiling, and works by creating custom timed interrupt events.

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