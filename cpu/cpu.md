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



### Parallel

CPU硬件支持的并行主要包括线程并行，指令级并行，数据并行。

线程并行

线程并行是指在同一时间内，不同的线程同时执行。线程之间没有依赖关系。现代CPU一般具有多个核心，开启超线程后，每个核心可以开启两个线程，每个线程可以独立的处理不同的任务。从而同一时间点，由多个CPU核心同时工作。

指令级并行

指令级并行依赖于**指令级流水技术，**通过指令级的流水达到在同一时间点(指令周期，更准确的说)内，无数据依赖的指令同时执行。

数据并行

数据并行主要指`SIMD`(Single -Instruction ,Multple -Data)单指令多数据流技术。通过一个指令，对多个相同类型的数据(也叫"数据向量”)进行同样的操作。



## SIMD

在费林分类法(Flynn's taxonomy)下，根据指令流和数据流来分类，共分为四种类型的计算平台

单指令流单数据流机器（SISD）：单核的串行数据流，早期的冯诺.依曼架构机器都是SISD架构。

单指令流多数据流机器（SIMD）：单核的并行数据流，一般支持ISA的单核计算机是这个架构。

多指令流单数据流机器（MISD）：多核的串行数据流，理论模型，还没有应用过。

多指令流多数据流机器（MIMD）：多核的并行数据流，目前的大多数多核计算机都是这个分类。

SIMD(**Single instruction, multiple data**),即单指令，多数据，是费林分类法下的一种并行处理技术。SIMD可以提高单个核心并行执行的性能，所以是提升程序并行性能的一个重要的优化方法。SIMD在很多基础软件中有很多重要的应用，如BloomFilter，哈希函数，加密算法等。

从1997年开始出现了面向x86架构下的MMX指令扩展，该指令集使用的是80-bits 寄存器。后来出现了SSE系列指令集SSE1-SSE4.2，使用的是128bits XMM寄存器，再后来是AVX/AVX2指令集，使用256bits YMM寄存器，目前Intel平台上支持指令最长的指令集是 AVX512，使用的是512bits ZMM寄存器，一次指令支持512bits的数据计算。SIMD的寄存器长度越长，单个指令可以处理的数据量也就越多，所以提高了程序的并行执行效率，从而提升了性能。SIMD寄存器的长度每增大一倍，一般相应的性能也得到显著提升。

除了Intel平台，AMD也曾经发布了基于x86架构的扩展指令集SSE5。Arm平台在ARMv7也有NEON sets，一种128-bits的固定长度的指令集; 在ARMv8开始支持的SVEand SVE2 instruction sets(最大到2048 bits)可变长度的指令集。指令集架构简写为`ISA(Instruction Set Architecture)`,大多数Linux服务器中均支持SSE4_2，AVX/AVX2，AVX512等指令集。通过`lscpu`可以通过Flags查看当前CPU支持的指令集架构。

预备知识：IPC,FLOPS

在科学计算中，一般使用FLOPS都是衡量处理器的算力，FLOPS即Floating Point Operations Per Second，表示每秒钟执行的单精度浮点数操作数。一般理论上最大的FLOPS计算如下:
$$
\begin {aligned}
peak\ FLOPS = &FP\ operators\ per\ instructions\ (SIMD\ width) \\  
						&\times instructions\ per\ cycle \\
						&\times cycles\ per\ second\ (frequency)
\end {aligned}
$$
其中，IPC即instructions per cycle，表示每个CPU时钟周期内的指令数，处理器并行度越高则IPC越大。可以看出在现代CPU时钟频率趋于不变时，增大SIMD的width便可以增加处理器的算力。前提是程序中是否可以利用向量化的技术

### **自动向量化技术**

在GCC编译器中,指定优化选项 `-O3` or `-ftree-vectorize` 即可开启自动向量化。除此之外，我们可以通过编译器支持的指令来提示编译器进行自动向量化技术编译代码。通常自动向量化编译的代码性能都不错。

* **__restrict**

GCC或LLVM编译器都提供了\_\_restrict关键字来帮助程序实现自动向量化。\_\_restrict 关键字的作用是告诉编译器该变量的内存独占且不存在重叠，可以进行SIMD优化。

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

上面两个函数实现是简单的int数组相加。差别在于a是否增加了__restrict关键字，\_\_restrict关键字表明变量a的内存是独占的，不会与其它变量b,c重叠或者共享。因此load以及store指令是互相独立的，可以并行执行，所以编译器可以进行SIMD优化。如果不指定\_\_restrict,则编译器不能保证a与b,c之间不会存在重叠的部分，所以load和store指令可能会操作同一块内存，所以store必须等待load执行完成后才可以执行。

从下面的汇编代码来看，就比较明显了。add_restrict函数直接使用xmm寄存器通过\_mm_add_epi32实现SIMD的优化，但add\_norestrict函数则不能优化，在汇编中rdx表示的是变量c，rsi表示变量b， rdi表示变量a，执行时按顺序将rdx寄存器地址的值压入eax累加器中，然后add指令与 rsi寄存器地址的值相加，然后将eax累加器的结果保存到rdi寄存器，也就是变量a中，一共计算4次，相比向量化的版本，未加\_\_restrict消耗的cpu instuctions要更多。

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

* **simd pragma**

pragma是编译器提供的帮助编译器自动向量化的指令，比如: pragma GCC ivdep表示下面的循环没有依赖关系，GCC编译器可以进行自动向量化编译。

```c++
void add (int * a, const int * b, int n) {
  #pragma GCC ivdep
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}
```

对于pragma GCC ivdep而言，GCC编译器会进行编译时的依赖关系检查决定是否进行自动向量化。除了依赖编译器提供的编译优化选项来优化也可以利用openmp来提示编译器自动向量化。OpenMP是跨平台共享内存的API，支持C, C++, fortran语言。而`#pragma omp simd`可以强制编译器进行自动向量化优化。`#pragma omp simd`告诉编译器不必检查依赖关系，由用户保证，只需要强制自动向量化即可。需要注意的是，如果想要使用openmp，在编译时需要加入编译选项-fopenmp。并且可以通过-mavx2来制定使用AVX2指令集。

```c++
void add (int * a, const int * b, int n) {
  #pragma omp simd
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}
```

在程序中，循环展开可以提示编译器更好的进行自动向量化优化。我们可以通过手写下面的代码来手动进行循环展开。在循环内展开有助于减少程序的循环次数，更好的利用cache line优化等。在较新的c++11之后版本中提供了`pragma unroll n`的编译优化提示。GCC编译器也提供了`pragma GCC unroll n`提示编译器进行循环展开优化。在clang编译器也可以使用`#pragma clang loop unroll_count(n)`来指定循环展开层数。Clang编译器种n可以指定为full，由编译器来决定循环展开的层数。

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

以下代码等价于上面的实现，通过pragma 提示编译器以下循环可以展开，并且可以提示展开的大小，当n=0或者1时，表示阻止编译器循环展开。这样的写法使得程序更加简洁。同样，你也可以使用GCC的function attribute`__attribute__((optimize("unroll-loops", "O3")))`或者`#pragma GCC optimize ("unroll-loops")`指定某一函数是否进行循环展开。需要注意的是以上编译优化是GCC编译器提供的循环展开优化。循环展开的层数一般不能超过寄存器的长度，比如YMM寄存器的最大长度是256bits，所以unroll loops一般超过8之后可能不会有加速效果。

```c++
void add (int * a, const int * b, int n) {
  #pragma GCC unroll 4
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}

__attribute__((optimize("unroll-loops", "O3")))
void add (int * a, const int * b, int n) {
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}

#pragma GCC push_options
#pragma GCC optimize ("unroll-loops")
void add (int * a, const int * b, int n) {
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}

#pragma GCC pop_options
```

* **GCC vector type**

c++中有valarray类型执行向量化计算。GCC中也可以通过\_\_attribute\_\_(vector_size(32))定义一个vector类型，这里的32表示vector的大小，如果是int类型，则vector可以表示8个int。并且vector之间支持加减乘除等运算符。所以对于vector type我们可以有下面的用法。

```
typedef int int8 __attribute__((vector_size(32)));

void add (const int * a, const int * b, int *c, int n) {
	size_t i = 0;
  for (i = 0; i < n - 7; i++)
      *((int8*)&c[i]) = *((int8*)&a[i]) + *((int8*)&b[i]);
  for (; i < n; ++i)
      c[i] = a[i] + b[i];
}
```

对于自动向量化的代码，如何确定是否生成了向量化的代码呢？可以通过以下两种方式：

1. 在GCC编译器中， 可以通过开启以下编译选项来获取自动向量化结果的详细信息。

- `-fopt-info-vec`或`-fopt-info-vec-optimized`：编译器将记录哪些循环取得自动向量化优化。
- `-fopt-info-vec-missed`：有关未向量化的循环的详细信息，以及许多其他详细信息。
- `-fopt-info-vec-note`：有关所有循环和优化的详细信息。
- `-fopt-info-vec-all`：包括以上所有的信息。

> **注意：**其他编译器优化也有类似`-fopt-info-[options]-optimized`的标志，例如`inline`：`-fopt-info-inline-optimized`

```
$cat opt.cpp
#include <immintrin.h>

void add(int* a, int* b, int n,  int c)
{
    for (int i = 0; i < n; ++i)
        a[i] = b[i] + c;
}
$g++ -fopt-info-vec-optimized -O3 -o opt opt.cpp
opt.cpp:5:23: optimized: loop vectorized using 16 byte vectors
opt.cpp:5:23: optimized:  loop versioned for vectorization because of possible aliasing
```

2. 可以在代码中插入汇编指令，通过gcc -S来查找插入汇编代码间是否存在带有xmm，ymm，zmm等寄存器的汇编指令来判断。

```
asm volatile ("# xxx loop begin");
for (i = 0; i < n; i++) {
    ... /∗ hope to be vectorized ∗/
}
asm volatile ("# xxx loop end");
```

例如对于上面的add函数，这里为了方便展示，把循环的次数设置为4。在汇编代码中可以看到xmm寄存器，证明生成的代码使用了SIMD的自动向量化代码生成优化。

```
$cat opt.cpp
#include <immintrin.h>

void add(int* a, int* b, int c)
{
asm volatile ("add loop begin");
    for (int i = 0; i < 4; ++i)
        a[i] = b[i] + c;
asm volatile ("add loop end");
}

$g++ -S opt.s -O3 -march=native opt.cpp
$cat opt.s
#APP
# 5 "opt.cpp" 1
	add loop begin
# 0 "" 2
#NO_APP
	leaq	4(%rsi), %rcx
	movq	%rdi, %rax
	subq	%rcx, %rax
	cmpq	$8, %rax
	jbe	.L2
	movdqu	(%rsi), %xmm1
	movd	%edx, %xmm2
	pshufd	$0, %xmm2, %xmm0
	paddd	%xmm1, %xmm0
	movups	%xmm0, (%rdi)
.L3:
#APP
# 8 "opt.cpp" 1
	add loop end
# 0 "" 2
```

### **SIMD intrinsic**

在SSE1-SSE4.2指令集中，主要使用的是128bits的XMM寄存器，寄存器包括XMM0-XMM15共16个寄存器。在AVX/AVX2中使用的是256bits的YMM寄存器，寄存器也包括YMM0-YMM15共16个寄存器。SSE为了兼容AVX，所以实际上YMM0的前128bits就是XMM0，YMM1的前128bits就是XMM1，以此类推。

所以我们下面主要以AVX/AVX2的SIMD指令集来讲解。SIMD的命名规则一般如下：_mm{128/256}\_{operator}\_{ps/pd/epi{xx}/si128/si256}。

SIMD支持多种数据类型，包括单精度浮点数float，双精度浮点数double，8/16/32/64/128/256位整型。

ps:表示计算类型是单精度浮点数

pd：表示计算类型是多精度浮点数

epi8/16/32/64 表示计算类型是有符号8/16/32/64位整型

Epu8/16/32/64 表示计算类型是无符号8/16/32/64位整型

si128/256 表示计算类型是128/256位整型

在代码中\_\_m256表示的是浮点数，\_\_mm256i 表示的是整型类型。\_\_mm256d表示的是双精度浮点数。所以比如_\_mm256_set1_ps(float),表示的是用8个相同的单精度浮点数填充整个YMM寄存器。其效果等同于\_mm256_set_ps(float, float,float,float,float,float,float,float)。

**operator**包括多种操作类型，比如add表示两个寄存器相加。load表示读取，store表示存储。extract表示提取寄存器中某一个数值。通常我们可以通过\_mm\_extract\_epiXXX 函数来提取某一个整型值。除此之外，还有gather，shuffle，permutation，mask等多种指令类型可供选择。

```c++
_mm256_set_ps : 设置8个float填充YMM，返回__m256 
_mm256_setr_ps : 以reverse的形式填充YMM，返回__m256
_mm256_add_ps : 计算两个__m256相加的结果
_mm256_movemask_ps: 计算单精度浮点数向量的所有符号位，返回值为int，比如0b00001110 表示第2，3，4为positive
```

让我们从下面的SIMD程序来揭开手写SIMD的序幕吧。

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

上面的程序展示了如何使用AVX2指令集实现两个向量的相加。首先通过\_mm256_setr_ps定义两个\_\_m256的向量，通过_mm256_add_ps将向量相加，最后通过\_mm256_store_ps将相加后的结果拷贝到float数组中。从程序中可以看到我们只用了1个SIMD指令就完成了8个float的相加，这也是SIMD高效的原因。

上面的程序中主要用到\_mm256\_setr\_ps，\_mm256\_add\_ps，\_mm256\_store\_ps三个指令。

\_mm256\_setr\_ps表示使用8个float类型以reverse的方式填充YMM寄存器。在寄存器中高位字节存在高位地址，低位字节存在低位地址。但Intel的内存字节序为小端序。高位字节在高位地址，低位字节在低位地址。所以对于数组中的元素，前面的元素存在低位地址。比如`int a[] = {1,2,3,4}`在内存中1的地址是最小的，所以当存入寄存器时，应该存在YMM寄存器的低位地址上。也就是_mm256_set_ps的最后一个元素。这与我们的直觉相反，所以我们可以使用\_mm256_setr_ps来代替，寄存器中的顺序跟数组的顺序就是一致的。

\_mm256\_add\_ps表示将两个寄存器中的\_\_m256相加，并将结果保存在寄存器中，这一过程只需要一个Instructions即可完成，对应的汇编指令是`        vaddps  ymm0, ymm0, ymm1`。

\_mm256\_store\_ps表示将寄存器中的结果保存到内存中，同样的也可以使用\_mm256\_stream\_ps表示绕过cache直接写入内存中，如果当前数据没有立即被使用时其性能会更好。



#### Reductions

AVX2中支持丰富的指令集，利用这些指令我们可以手动实现一些向量化的函数来提升性能。比如对于如下的条件判断函数，其作用是求源数组中所有正数的和，通常我们可以有如下实现:

```c++
int sum_positive_scalar(const int * src, size_t count)
  {
      int res = 0;
      for (size_t i = 0; i < count; ++i) {
          if(src[i] >= 0) {
              res += src[i];
          }
      }
      return res;
  }
```

如果开启自动向量化，上面的程序如果打开O3编译选项一般可以进行自动向量化编译，但如果分支条件变得更加复杂，编译器无法自动向量化时，我们可以通过手动向量化该函数来加速计算。

下面的程序中首先通过\_mm256_cmpgt_epi32来获取大于0的mask标记位，mask实际上是一个0和1组成的__m256i变量，通过\_mm256_and_si256得到满足mask条件的所有result结果，并将其与res累加起来，最后通过hsum函数将result中的8个int累加起来，实现的方式是想将高128位和低128位提取出来相加，然后执行一次128位的hadd，此时只需要将前64位的两个int相加，即可得到整个向量的和。通过下面的向量化实现，我们可以获得与编译器自动向量化版本相等的性能结果。比不开启自动向量化优化性能有6倍的提升。

![image-20220817164313346](./pic/2.png)

```c++
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
          res = _mm256_add_epi32(res, _mm256_and_si256(vsrc, mask));
      }
      return hsum(res);
  }
```

#### shuffle

filter是数据库中常见的算子，filter实现的功能是将满足一定条件的input数据输出到output中。非向量化的代码实现如下：

```c++
int filter(int * input, int bond, int * output) {
    int k = 0;

    for (int i = 0; i < N; i++)
        if (input[i] < bond)
            output[k++] = input[i];

    return k;
}
```



我们可以使用`_mm256_permutevar8x32_epi32`来实现filter的向量化。向量化实现时需要建立一个lookup table。lookup table中保存的是0-255的二进制对应的偏移量数组，比如对于13，二进制是00001101，对应的数组是{0,2,3,0,0,0,0,0}。这里只有前三个值是有效的。

可以按照以下步骤对上述的filter函数进行向量化实现。

1. `_mm256_cmpgt_epi32`对8个整型值与P进行比较，得到比较后的mask向量
2. `_mm256_movemask_ps`讲mask向量转换成8-bit的整型值。
3. 根据8-bit整型值查找lookup table得到得到需要排序的permutation向量

4. `_mm256_permutevar8x32_epi32`函数通过permutation向量来重新排列input向量。比如对于一个数组{1,2,3,4,5,6,7,8}通过{0,2,3,0,0,0,0,0}数组重排得到新的值为{1,3,4,1,1,1,1,1}。只有前三位是有效的。
5. 将结果保存在output向量中，其中有部分结果是不需要的。
6. `__builtin_popcount`计算出新的偏移量k,下一次保存到output时以第k地址所在位置开始保存。

向量化版本的实现中无论选择率的大小，性能都比较稳定。因为向量化实现中去除了条件选择判断，并且实现了比非向量化版本更好的性能。

```
struct Precalc {
    alignas(64) int permutation[256][8];

    constexpr Precalc() : permutation{} {
    for (int m = 0; m < 256; m++) {
        int k = 0;
        for (int i = 0; i < 8; i++)
        if (m >> i & 1)
        		permutation[m][k++] = i;
        }
    }
};

static constexpr Precalc T;
int filter(int * input, int bond, int * output) {
    int k = 0;
    const __m256i p = _mm256_set1_epi32(bond);
    for (int i = 0; i < N; i += 8)
    {
        __m256i x = _mm256_load_si256( (__m256i*)&input[i]);

        __m256i m = _mm256_cmpgt_epi32(p, x);
        int mask = _mm256_movemask_ps((__m256) m);
        __m256i permutation = _mm256_load_si256( (__m256i*) &T.permutation[mask] );

        x = _mm256_permutevar8x32_epi32(x, permutation);
        _mm256_storeu_si256((__m256i*) &output[k], x);

        k += __builtin_popcount(mask);
    }

    return k;
} 
```



### BloomFilter的SIMD实现

*Bloom Filter*是由Bloom在1970年提出的一种空间效率高的概率型数据结构。它可以用来查找某一个元素是否存在于集合中。BloomFilter有多种变种，我们要介绍的是Block BloomFilter，由Cache-, Hash- and Space-Efficient Bloom Filters论文首次提出。BloomFilter初始化时是一个bit位组成的集合，集合中所有的bit位均置为0。BloomFilter的最重要的两个函数即为Add函数和Find函数。通过Add函数可以将一个元素通过hash函数计算后映射到集合中的某一个bit位置为1。一般需要经过多次hash函数计算。Find函数则通过将元素通过多次hash计算后比较集合中对应的bit位是否都为1，如果不全为1，则该元素一定不存在，反之，则可能存在。

下面是BlockBloomFilter的非SIMD版本实现。我们主要关注其中的算术运算。

```
void BlockBloomFilter::BucketInsert(const uint32_t bucket_idx, const uint32_t hash) noexcept {
  // new_bucket will be all zeros except for eight 1-bits, one in each 32-bit word. It is
  // 16-byte aligned so it can be read as a __m128i using aligned SIMD loads in the second
  // part of this method.
  uint32_t new_bucket[kBucketWords] __attribute__((aligned(16)));
  for (int i = 0; i < kBucketWords; ++i) {
    // Rehash 'hash' and use the top kLogBucketWordBits bits, following Dietzfelbinger.
    new_bucket[i] = (kRehash[i] * hash) >> ((1 << kLogBucketWordBits) - kLogBucketWordBits);
    new_bucket[i] = 1U << new_bucket[i];
  }
  for (int i = 0; i < 2; ++i) {
    __m128i new_bucket_sse = _mm_load_si128(reinterpret_cast<__m128i*>(new_bucket + 4 * i));
    __m128i* existing_bucket = reinterpret_cast<__m128i*>(
        &DCHECK_NOTNULL(directory_)[bucket_idx][4 * i]);
    *existing_bucket = _mm_or_si128(*existing_bucket, new_bucket_sse);
  }
}

bool BlockBloomFilter::BucketFind(
    const uint32_t bucket_idx, const uint32_t hash) const noexcept {
  for (int i = 0; i < kBucketWords; ++i) {
    BucketWord hval = (kRehash[i] * hash) >> ((1 << kLogBucketWordBits) - kLogBucketWordBits);
    hval = 1U << hval;
    if (!(DCHECK_NOTNULL(directory_)[bucket_idx][i] & hval)) {
      return false;
    }
  }
  return true;
}
```

非SIMD实现中的kBucketWords为8，kLogBucketWordBits为5，可以看出在for循环内部需要计算一些乘法以及移位运算，在判断元素是否存在时需要与运算，这样的计算非常适合使用SIMD来加速。比如对于Add函数和Find函数中的`(kRehash[i] * hash) >> ((1 << kLogBucketWordBits) - kLogBucketWordBits)`计算可以使用SIMD来加速。kRehash实际上是一个静态数组，我们通过`_mm256_setr_epi32`的指令存储在YMM寄存器中。通过`_mm256_set1_epi32`将hash值复制8次并存储在另一个YMM寄存器中。`(kRehash[i] * hash)`的乘法运算通过`_mm256_mullo_epi32`计算一次得到8个乘法的结果，`_mm256_mullo_epi32`表示乘法计算后只保留低32位结果。这是因为两个32位整型相乘会得到一个64位整型。但实际上我们只需要低32位结果。之后是`_mm256_srli_epi32(hash_data, 27)`右移27位。即完成上面的复杂表达式的计算。同时`hval = 1U << hval;`计算也可以通过`_mm256_sllv_epi32`SIMD右移指令实现。这样对于原来的for循环，我们可以使用SIMD指令进行并行计算加速。

对于Add函数，对集合中计算的bit位置为1，可以通过或运算实现，在SIMD intrinsic中通过`_mm256_or_si256`来实现两个YMM计算器的或运算。对于Find函数，通过比较集合中对应的bit位是否全为1来判断该元素是否存在，在SIMD intrinsic中使用`int _mm256_testc_si256 (__m256i a, __m256i b)`来实现，该指令表示如果两个256bits向量中先对a进行NOT运算，然后与b进行AND运算，如果结果为0，则返回CF标记位为1，否则返回CF标记位为0。假设集合a中某一元素为0，NOT计算得到1，如果b中对应的元素为1，则结果一定不为0，表示a中的bit位没有被标记过，但b中的元素是被标记过的，所以b一定不存在a中，所以返回CF标记位为0。通过以上的SIMD intrinsic计算即可完成BloomFilter的SIMD实现。通过SIMD加速Block BloomFilter计算，性能相比非SIMD版本提升数倍。

```
static inline __attribute__((__target__("avx2"))) __m256i MakeMask(
    const uint32_t hash) {
  const __m256i ones = _mm256_set1_epi32(1);
  const __m256i rehash = _mm256_setr_epi32(BLOOM_HASH_CONSTANTS);
  __m256i hash_data = _mm256_set1_epi32(hash);
  hash_data = _mm256_mullo_epi32(rehash, hash_data);
  hash_data = _mm256_srli_epi32(hash_data, 27);
  return _mm256_sllv_epi32(ones, hash_data);
}

void BlockBloomFilter::BucketInsertAVX2(const uint32_t bucket_idx, const uint32_t hash) noexcept {
  const __m256i mask = MakeMask(hash);
  __m256i* const bucket = &(reinterpret_cast<__m256i*>(directory_)[bucket_idx]);
  _mm256_store_si256(bucket, _mm256_or_si256(*bucket, mask));
  // For SSE compatibility, unset the high bits of each YMM register so SSE instructions
  // dont have to save them off before using XMM registers.
  _mm256_zeroupper();
}

bool BlockBloomFilter::BucketFindAVX2(const uint32_t bucket_idx, const uint32_t hash) const
    noexcept {
  const __m256i mask = MakeMask(hash);
  const __m256i bucket = reinterpret_cast<__m256i*>(directory_)[bucket_idx];
  const bool result = _mm256_testc_si256(bucket, mask);
  _mm256_zeroupper();
  return result;
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

CPU的乱序执行可以提高指令的并行度，但遇到分支判断失败时，则提前执行的指令需要重新执行。大大降低了程序的执行效率，对于分支跳转不确定的场景下，分支判断失败不能享受乱序执行的优势。这时我们可以通过无分支的代码来避免分支预测失败带来的副作用。branchless 通常情况下可以提升系统的性能，尤其是在选择率较低的场景下。下面的程序用于计算数组中大于0的元素个数。通常的带有分支判断的写法如下：

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

在带有分支判断的实现中，分支判断成功时，res加1，分支判断失败时，res不变。实际上我们可以直接用`res += (src[i] >= 0)`来代替分支判断的结果，这样便可以去掉分支判断对性能的影响。不带有分支判断实现的好处在于无论分支判断的选择率如何，性能会比较稳定，不会出现较大的波动。具体实现如下面的代码所示：

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

实际上，对于这类简单的分支判断，我们也可以使用SIMD Intrinsic来完成向量化实现。SIMD可以使程序在没有自动向量化的情况下也可以实现更高的性能。SIMD的写法如下：

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

branchless 相比branch虽然消耗了更多的CPU Instructions，但实际上在选择率低于50%的场景下，branchless性能更好，所需要的CPU Cycles更少。SIMD版本的性能最好，因为消耗最少的CPU Instructions。

### memory barrier/memory fence/CPU fence

C++11 中的std::atomic描述了 6 种可以应用于原子变量的内存次序:

- momory_order_relaxed,
- memory_order_consume,
- memory_order_acquire,
- memory_order_release,
- memory_order_acq_rel,
- memory_order_seq_cst.

但它们表示的是三种内存模型:(memory_order_consume目前被当成memory_order_acquire处理)

- sequential consistent(memory_order_seq_cst),
- relaxed(memory_order_relaxed)
- acquire release(memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel),

下面我们分别来介绍一下std::atomic原子变量的内存模型

1. **sequential consistent memory order**

   sequential consistent叫做顺序一致性。是最强的内存序，也是std::atomic默认的内存序。它要求所有线程之间的全局同步。因此在多线程间的同步开销较大，相比于其它线程模型性能会稍差一点。

   * 在`store()`前的所有读写操作，不允许被移动到这个`store()`的后面。
   * 在`store()`后的所有读写操作，不允许被移动到这个`store()`的前面。
   * 在`load()`前的所有读写操作，不允许移动到这个`load()`的后面。
   * 在`load()`后的所有读写操作，不允许移动到这个`load()`的前面。

   在下面的程序中z可能等于1，或者2，但z一定不等于0，所以z != 0的断言不会失败。

   ```c++
   #include <thread>
   #include <atomic>
   #include <cassert>
    
   std::atomic<bool> x = {false};
   std::atomic<bool> y = {false};
   std::atomic<int> z = {0};
    
   void write_x()
   {
       x.store(true, std::memory_order_seq_cst);
   }
    
   void write_y()
   {
       y.store(true, std::memory_order_seq_cst);
   }
    
   void read_x_then_y()
   {
       while (!x.load(std::memory_order_seq_cst))
           ;
       if (y.load(std::memory_order_seq_cst)) {
           ++z;
       }
   }
    
   void read_y_then_x()
   {
       while (!y.load(std::memory_order_seq_cst))
           ;
       if (x.load(std::memory_order_seq_cst)) {
           ++z;
       }
   }
    
   int main()
   {
       std::thread a(write_x);
       std::thread b(write_y);
       std::thread c(read_x_then_y);
       std::thread d(read_y_then_x);
       a.join(); b.join(); c.join(); d.join();
       assert(z.load() != 0);  // will never happen
   }
   ```

2. **relaxed memory order**

   relaxed内存序要求`load()`和`store()`都要是`memory_order_relaxed`,relaxed语义只能保证同一个原子变量的修改是顺序的，也就是说当前线程修改后再读一定读取的是最新修改的值，但其它线程不一定读取的是最新的值。relaxed内存序不能提供跨线程的同步。

   比如下面的程序中，是允许D->A->B->C的执行顺序的，relaxed并不具备跨线程同步的语意，只能保证A->B的顺序。所以下面的程序有可能输出为x=y=42的结果

   ```c++
   //initially
   x = 0; y = 0;
   
   // Thread 1:
   r1 = y.load(std::memory_order_relaxed); // A
   x.store(r1, std::memory_order_relaxed); // B
   // Thread 2:
   r2 = x.load(std::memory_order_relaxed); // C 
   y.store(42, std::memory_order_relaxed); // D
   ```

   但relaxed内存序对于有循环依赖的情况下并不允许

   ```c++
   // Thread 1:
   r1 = y.load(std::memory_order_relaxed);
   if (r1 == 42) x.store(r1, std::memory_order_relaxed);
   // Thread 2:
   r2 = x.load(std::memory_order_relaxed);
   if (r2 == 42) y.store(42, std::memory_order_relaxed);
   ```

   relaxed内存序最典型的使用场景就是递增计数器，比如 `fetch_add(1, std::memory_order_relaxed);`

3. **Acquire release memory order**

   acquire release模型下`store()`使用的是`memory_order_release`,而`load()`使用的是`memory_order_acquire`.acquire release虽然不同线程可以看到不同的排序，但排序的顺序是受限制的，限制条件如下:

   - 在`store()`之前的所有读写操作，不允许被移动到这个`store()`的后面。
   - 在`load()`之后的所有读写操作，不允许被移动到这个`load()`的前面。

   在下面的程序中，thread2中`ptr.store(p, std::memory_order_release)`之前的所有store()操作都一定会完成，而thread1中的`while (!(p2 = ptr.load(std::memory_order_acquire)))`之后的load()也不会提前执行，所以`*p2 == "Hello"`和`data == 42`的断言一定不会失败。

   ```
   //thread1
   std::string* p2;
   while (!(p2 = ptr.load(std::memory_order_acquire)))
   ;
   assert(*p2 == "Hello"); // never fires
   assert(data == 42); // never fires
   
   //thread2
   std::string* p  = new std::string("Hello");
   data = 42;
   ptr.store(p, std::memory_order_release);
   ```

### CPU缓存一致性协议

现代CPU中一般采用的memory hierarchy的方式，越靠近CPU的cache读写速度也越快。多核CPU之间Cache一般是不共享的，所以需要通过缓存一致性协议MESI来保持CPU之间的状态一致。

MESI包括独占(exclusive)、共享(share)、修改(modified)、失效(invalid)，用来描述该缓存行是否被多处理器共享、是否修改。

- 独占(exclusive)：仅当前处理器拥有该缓存行，并且没有修改过，是最新的值。
- 共享(share)：有多个处理器拥有该缓存行，每个处理器都没有修改过缓存，是最新的值。
- 修改(modified)：仅当前处理器拥有该缓存行，并且缓存行被修改过了，一定时间内会写回主存，会写成功状态会变为S。
- 失效(invalid)：缓存行被其他处理器修改过，该值不是最新的值，需要读取主存上最新的值。



### cpu clock/ timer

当我们要对一段程序进行性能测试或者调优时，通常需要通过计时器来记录程序运行的时间。

在Linux平台上有多种计时工具，常见的如`clock`, `gettimeofday`, `clock_gettime`, `std::chrono::system_clock`, `std::chrono::steady_clock`, `std::chrono::high_resolution_clock`, `rdtsc`等等。

在所有的计时工具中，`clock_gettime`计时器本身的开销大概在1ns(实际测量时间与CPU主频有关)。其与`std::chrono::steady_clock`, `std::chrono::high_resolution_clock`, `std::chrono::system_clock`精度接近。但它的稳定性和精度跨平台性最好(C++11标准)。`clock_gettime`函数原型是`int clock_gettime( clockid_t clock_id,struct timespec * tp );`其中`clockid_t`时钟类型取值有`CLOCK_REALTIME`,`CLOCK_MONOTONIC`,`LOCK_PROCESS_CPUTIME_ID`,`CLOCK_THREAD_CPUTIME_ID`等。`CLOCK_REALTIME`代表POSIX系统时间自1970-01-01起经历的绝对时间。这个时间会被用户更新时间打断。`CLOCK_MONOTONIC`代表系统单调时间，表示自开机起经历的时间。它不可以被中断。`LOCK_PROCESS_CPUTIME_ID`代表进程执行的时间。`CLOCK_THREAD_CPUTIME_ID`代表线程启动后执行的时间。所以建议使用更稳定的`CLOCK_MONOTONIC`时钟。

`rdtsc`在相同的CPU主频下精度最高，速度最快，稳定性最好，但并非所有CPU均支持 。

`std::chrono::steady_clock`, `std::chrono::high_resolution_clock`,基本一致，它们记录的是相对时间，并且不会因为修改系统时间而受影响。`std::chrono::system_clock`记录的是绝对时间。可能会被用户打断。它们的最小精度都可以到纳秒级别。

在linux平台下，我们的性能测试数据是采用`std::chrono::high_resolution_clock`的计时方式来测试的。其精度可以满足我们对性能测试的要求。

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



**reference** ：

https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html （Build_in 内置指令）

https://www.ibm.com/docs/en/zos/2.4.0?topic=pragmas-individual-pragma-descriptions (Pragma优化)

https://gcc.gnu.org/onlinedocs/gcc-4.5.2/gcc/Optimize-Options.html（optimize options）

http://svmoore.pbworks.com/w/file/fetch/70583970/VectorOps.pdf

https://sites.cs.ucsb.edu/~tyang/class/240a17/slides/SIMD.pdf

http://web.eecs.utk.edu/~jplank/plank/classes/cs494/494/notes/SIMD/index.html

https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX

http://ftp.cvut.cz/kernel/people/geoff/cell/ps3-linux-docs/CellProgrammingTutorial/BasicsOfSIMDProgramming.html
