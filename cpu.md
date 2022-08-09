# CPU

## CPU架构演进（SMP、NUMA）

CPU的频率达到瓶颈后转向多核的发展方向，而CPU多核之间主要存在两种架构体系分别是SMP和NUMA架构，通过lscpu即可查看CPU的numa信息。

* **SMP(Symmetric Multi-Processor)**

对称多处理器结构的主要特征是共享，即所有CPU共享所有的资源（总线，内存，I/O）等。操作系统管理着一个队列，每个处理器依次处理队列中的进程。如果两个处理器同时请求访问一个资源（例如同一段内存地址），由硬件、软件的锁机制去解决资源争用问题。这同时也预示着SMP最大的问题就是扩展性，CPU利用率很快便达到了瓶颈。

* **NUMA(Non-Uniform Memory Access)**

NUMA解决了SMP扩展性的问题。NUMA的主要特征是将CPU进行分组，每组CPU共享内部的内存，I/O资源。不同组别之间的CPU通过CPU内互联网络连接。从而解决了CPU共享的问题。不过这也造成了CPU访问远端内存的速度要远小于访问本地的内存。

## <span id="pipeline">CPU pipeline</span>

**标量处理器**CPU内部一般采用标量流水线技术，即一个CPU指令一般分为五个阶段：取址(IF)，译码(ID)，执行(EX)，访存(MEM)，写回(WB)。所以一般一个CPU指令需要4～5个时钟周期(取决于是否需要访存)。如果指令间没有依赖关系，则多个指令可以并行执行，而当指令间存在依赖时，则后一个指令必须等待前一个指令执行完才可以执行。

![img](./pic/1.png)

**超标量处理器**是在单个处理器内一种称为指令级并行的并行形式的CPU。与每个时钟周期最多可以执行一条指令的标量处理器相比，超标量处理器可以通过同时将多条指令分派到处理器上的不同执行单元来在一个时钟周期内执行多条指令。

## SIMD

CPU除了多核并化的优化外，另一个比较重要的优化便是SIMD。SIMD在很多基础软件中有很多重要的应用，如BloomFilter，加密算法等。

SIMD(**Single instruction, multiple data**),即单指令，多数据，是费林分类法(Flynn's taxonomy)下的一种并行处理技术。从1997年面向x86架构下的MMX指令扩展（80-bits 寄存器），到后来的SSE1-SSE4.2(128bits XMM寄存器)，AVX/AVX2（256bits YMM寄存器）， AVX512（512bits ZMM寄存器），SIMD寄存器的长度每增大一倍，一般相应的性能也得到显著提升。除了Intel平台，AMD也曾经发布了基于x86架构的扩展指令集SSE5。Arm平台在ARMv7也有NEON sets，一种128-bits的固定长度的指令集; 在ARMv8开始支持的SVEand SVE2 instruction sets(最大到2048 bits)可变长度的指令集。

### **自动向量化技术**

在GCC编译器中,指定优化选项 `-O3` or `-ftree-vectorize` 即可开启自动向量化。

\_\_restrict__ 关键字告诉编译器该变量的内存独占且不存在重叠，可以进行SIMD优化。

```c++
void add_restrict(int * __restrict a, const int * b, const int * c) {
    for (int i = 0; i < 4; i++)
        a[i] = b[i] + c[i];
}

void add_norestrict(int * a, const int * b, const int * c) {
    for (int i = 0; i < 4; i++)
        a[i] = b[i] + c[i];
}
```

上面两个函数实现是简单的int数组相加。差别在于a是否增加了__restrict关键字，\_\_restrict关键字表明a的内存是独占的，不会与其它变量b,c重叠或者共享。load以及store指令是独立的，可以并行执行，所以编译器可以进行SIMD优化。如果不指定\_\_restrict,则编译器不能保证a与b,c之间不会存在重叠的部分，所以load和store指令可能会操作同一块内存，所以必须顺序执行。从汇编代码来看，就比较明显了。add_restrict函数直接使用xmm寄存器通过\_mm_add_epi32实现SIMD的优化，但add\_norestrict函数则不能优化，在汇编中rdx表示的是变量c，rsi表示变量b, rdi表示变量a，只能按顺序将rdx寄存器地址的值压入eax累加器中，然后add rsi寄存器地址的值，然后将结果保存到rdi寄存器中，一共计算4次，相比向量化的版本，未加\_\_restrict消耗的cpu instuctions要更多。

```
add_restrict(int*, int const*, int const*):               # @add_restrict(int*, int const*, int const*)
        movdqu  xmm0, xmmword ptr [rsi]
        movdqu  xmm1, xmmword ptr [rdx]
        paddd   xmm1, xmm0
        movdqu  xmmword ptr [rdi], xmm1
        ret
add_norestrict(int*, int const*, int const*):             # @add_norestrict(int*, int const*, int const*)
        mov     eax, dword ptr [rdx]
        add     eax, dword ptr [rsi]
        mov     dword ptr [rdi], eax
        mov     eax, dword ptr [rdx + 4]
        add     eax, dword ptr [rsi + 4]
        mov     dword ptr [rdi + 4], eax
        mov     eax, dword ptr [rdx + 8]
        add     eax, dword ptr [rsi + 8]
        mov     dword ptr [rdi + 8], eax
        mov     eax, dword ptr [rdx + 12]
        add     eax, dword ptr [rsi + 12]
        mov     dword ptr [rdi + 12], eax
        ret
```

Pragma帮助编译器自动向量化，pragma GCC ivdep表示下面的循环没有依赖关系，编译器可以进行自动向量化编译。

```c++
void add (int * a, const int * b, int n) {
  #pragma GCC ivdep
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}
```

pragma GCC unroll n,表示帮助编译器循环展开。

```c++
void add (int * a, const int * b, int n) {
  int i = 0;
  for (i = 0; i < n - 3; i+=4) {
      a[i] += b[i];
      a[i+1] += b[i+1];
      a[i+2] += b[i+2];
      a[i+3] += b[i+3];
  }

  while (i++ < n) { a[i] += b[i]; }
}
```

以下代码等价于上面的实现

```c++
void add (int * a, const int * b, int n) {
  #pragma GCC unroll 4
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}
```

### **SIMD intrinsic**

在SSE1-SSE4.2指令集中，主要使用的是128bits的XMM寄存器，寄存器包括XMM0-XMM15共16个寄存器。在AVX/AVX2中使用的是256bits的YMM寄存器，寄存器也包括YMM0-YMM15共16个寄存器。SSE为了兼容AVX，所以实际上YMM0的前128bits就是XMM0，YMM1的前128bits就是XMM1，以此类推。

所以我们下面主要以AVX/AVX2的SIMD指令集来讲解。SIMD的命名规则一般如下：_mm{128/256}\_{operator}\_{ps/pd/epi{xx}/si128/si256}。

SIMD支持多种数据类型，包括单精度浮点数float，双精度浮点数double，8/16/32/64/128/256位整型。

ps:表示计算类型是单精度浮点数

pd：表示计算类型是多精度浮点数

epi8/16/32/64 表示计算类型是8/16/32/64位整型

si128/256 表示计算类型是128/256位整型

在代码中\_\_m256表示的是浮点数，\_\_mm256i 表示的是整型类型。\_\_mm256d表示的是双精度浮点数。所以比如_\_mm256_set1_ps(float),表示的是用8个相同的单精度浮点数填充整个YMM寄存器。其效果等同于\_mm256_set_ps(float, float,float,float,float,float,float,float)。

**operator**包括多种操作类型，比如add表示两个寄存器相加。load表示读取，store表示存储。extract表示提取寄存器中某一个数值。通常我们可以通过\_mm\_extract\_epiXXX 函数来提取某一个整型值。除此之外，还有gather，shuffle，permutation，mask等多种指令类型可供选择。

```c++
_mm256_set_ps : 设置8个float填充YMM，返回__m256 
_mm256_setr_ps : 以reverse的形式填充YMM，返回__m256
_mm256_add_ps : 计算两个__m256相加的结果
_mm256_movemask_ps: 计算单精度浮点数向量的所有符号位，返回值为int，比如0b00001110 表示第2，3，4为positive
```

第一个SIMD程序

```c++
#pragma GCC target("avx2")
#pragma GCC optimize("O3")
#include <immintrin.h>
#include <iostream>

static std::ostream & operator<<(std::ostream& output, const __m256 & p)
{
    output<<"[";
    for(int i=0;i<(8);++i)⋅
    {
        output << p[i];
        if (i != 7)
            output<<",";
    }
    output << "]";
    return output;
}

int main()
{
    float c[8];
    const __m256 __attribute__((aligned(32))) va = _mm256_setr_ps(1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0);
    const __m256 __attribute__((aligned(32))) vb = _mm256_setr_ps(1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0);

    __m256 __attribute__((aligned(32))) vc = _mm256_add_ps(va, vb);
    
    _mm256_store_ps(&c[0], vc);

    std::cout << vc << std::endl;
    for (size_t i = 0; i < 8; ++i)
    {
    	std::cout << c[i] << std::endl;
    }
}
```

上面的程序展示了如何使用AVX指令集实现两个向量的相加。首先通过\_mm256_setr_ps定义两个\_\_m256的向量，通过_mm256_add_ps将向量相加，最后通过\_mm256_store_ps将相加后的结果拷贝到float数组中。从程序中可以看到我们只用了1个SIMD指令就完成了8个float的相加，这也是SIMD高效的原因。

#### SIMD aggregation

```
  int hsum(__m256i x) {
      __m128i l = _mm256_extracti128_si256(x, 0);
      __m128i h = _mm256_extracti128_si256(x, 1);
      l = _mm_add_epi32(l, h);
      l = _mm_hadd_epi32(l, l);
      return _mm_extract_epi32(l, 0) + _mm_extract_epi32(l, 1);
  }

  int sum_positive_vec(const int *src, size_t count)
  {
      __m256i res = _mm256_setzero_si256();
      const __m256i bound = _mm256_setzero_si256();
      for (size_t i = 0; i < count; i += 8)
      {
          const __m256i __attribute__((aligned(32))) *vsrc = reinterpret_cast<const __m256i *>(src + i);
          __m256i mask = _mm256_cmpgt_epi32(*vsrc, bound);
          res = _mm256_add_epi32(res, _mm256_maskload_epi32(src+i, mask));
      }
      return hsum(res);
  }
```



### Out-of-Order Execution

为了最大化流水线的执行效率，CPU通过乱序执行来优化并发执行的效率。乱序执行分为两种情况：

1. 在编译期，编译器进行指令重排。
2. 在运行期，CPU 进行指令乱序执行。

### branch prediction

Intel的CPU的乱序执行通常会让一些指令提前执行，而分支预测则有可能打破这种秩序。如果分支预测成功，则乱序执行会提升程序的执行效率，而当分支预测失败，则提前计算的CPU指令变为无效。如果保持较高的分支预测效率，分支预测便可以提升流水线的性能。

分支预测成功率主要是为了解决CPU流水线阻塞的问题。一般分为静态分支预测和动态分支预测的方法。

静态分支预测是软件侧的分支预测实现，更准确的说是通过编译器来达到更好的分支预测结果，减少pipeline flush。

动态分支预测主要分为两种，首先是分支结果的预测，对分支结果预测主要是为了减少pipeline stall。其次是分支跳转指令地址预测，即BTA(Branch Target Address)的预测。



程序中通常通过__builtin_expect函数提示编译器来增加分支预测的准确性。

```c++
#if !defined(likely)
#    define likely(x)   (__builtin_expect(!!(x), 1))
#endif
#if !defined(unlikely)
#    define unlikely(x) (__builtin_expect(!!(x), 0))
#endif
```

#### branchless

branchless 通常情况下可以提升系统的性能，尤其是在选择率较低的场景下。下面的程序用于计算数组中大于0的元素个数。通常的写法如下：

```c++
int count_positive(const int *src, 
                     size_t count) 
{
    int res = 0;
    for(size_t i = 0; i < count; ++i) {
        if(src[i] >= 0) {
            res++;
        }
    }
    return res;
}
```

branchless的写法如下

```c++
int count_positive(const int *src, 
                     size_t count) 
{
    int res = 0;
    for(size_t i = 0; i < count; ++i) {
    	  res += (src[i] >= 0);
    }
    return res;
}
```

SIMD的写法如下：

```c++
int count_positive(const int * src, size_t count)
{
    int res = 0;
    for (size_t i = 0; i < count; i += 8) {
        const __m256i *vsrc = reinterpret_cast<const __m256i *>(src + i);
        __m256i mask = _mm256_cmpgt_epi32(*vsrc, _mm256_setzero_si256());
        res += __builtin_popcount(_mm256_movemask_ps((__m256)mask));
    }
    return res;
}
```

branchless 相比branch虽然消耗了更多的CPU Instructions，但实际上在选择率低于50%的场景下，branchless性能更好，所需要的CPU 

Cycles更少。SIMD版本的性能最好，消耗最少的CPU Instructions。

### memory barrier/memory fence/CPU fence

C++11  述了 6 种可以应用于原子变量的内存次序:

- momory_order_relaxed,
- memory_order_consume,
- memory_order_acquire,
- memory_order_release,
- memory_order_acq_rel,
- memory_order_seq_cst.

虽然共有 6 个选项,但它们表示的是三种内存模型:

- sequential consistent([memory_order_seq_cst](https://www.zhihu.com/search?q=memory_order_seq_cst&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A83422523})),

- relaxed(memory_order_seq_cst).
  
  acquire release(memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel),



* CPU缓存一致性storebuffer，InvalidQueue（ARM VS INTEL）

  现代CPU中一般采用的memory hierarchy的方式，越靠近CPU的cache读写速度也越快。多核CPU之间Cache一般是不共享的，所以需要通过缓存一致性协议MESI来保持CPU之间的状态一致。

  MESI包括独占(exclusive)、共享(share)、修改(modified)、失效(invalid)，用来描述该缓存行是否被多处理器共享、是否修改。

  - 独占(exclusive)：仅当前处理器拥有该缓存行，并且没有修改过，是最新的值。
  - 共享(share)：有多个处理器拥有该缓存行，每个处理器都没有修改过缓存，是最新的值。
  - 修改(modified)：仅当前处理器拥有该缓存行，并且缓存行被修改过了，一定时间内会写回主存，会写成功状态会变为S。
  - 失效(invalid)：缓存行被其他处理器修改过，该值不是最新的值，需要读取主存上最新的值。



* context switch

* lock（shared rw lock & unqiue lock） vs atomic

  一般来说锁的开销要大于原子操作，常见的锁分为自旋锁，互斥锁，读写锁。自旋锁通过用于较短时间的锁，因为它会长时间占用CPU，互斥锁的原理是将CPU置于等待队列中，等待唤醒，所以 常用于较长时间的上锁。读写锁更多用于读多写少的场景。



### cpu clock/ timer

同样，我们要对一段程序进行性能测试时，需要记录程序运行的时间，在Linux平台上有多种计时工具，常见的如clock, gettimeofday, clock_gettime, std::chrono::system_clock, std:;chrono::steady_clock, std::chrono::high_resolution_clock, rdtsc。在所有的计时工具中，std::chrono的稳定性和精度均为良好并且跨平台性最好(C++11标准)，rdtsc精度最高，速度最快，稳定性最好 。我们所有的性能测试均采用std::chrono::high_resolution_clock的计时方式来测试性能。

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
        std::cout << "time elapse: " << elapse.count()*1000 << "ms" <<  std::endl;
    }

private:
    std::chrono::time_point<std::chrono::high_resolution_clock> start_time;
    std::chrono::time_point<std::chrono::high_resolution_clock> end_time;
    std::chrono::duration<double> elapse;

};
```



**reference** ：

https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html （Build_in 内置指令）

https://www.ibm.com/docs/en/zos/2.4.0?topic=pragmas-individual-pragma-descriptions (Pragma优化)

http://svmoore.pbworks.com/w/file/fetch/70583970/VectorOps.pdf

https://sites.cs.ucsb.edu/~tyang/class/240a17/slides/SIMD.pdf

http://web.eecs.utk.edu/~jplank/plank/classes/cs494/494/notes/SIMD/index.html

https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX

http://ftp.cvut.cz/kernel/people/geoff/cell/ps3-linux-docs/CellProgrammingTutorial/BasicsOfSIMDProgramming.html
