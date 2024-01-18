# 向量化处理技术

## SIMD

在费林分类法（Flynn's taxonomy）下，根据指令流和数据流来分类，共分为四种类型的计算平台

单指令流单数据流机器（SISD）：单核的串行数据流，早期的冯诺.依曼架构机器都是SISD架构。

单指令流多数据流机器（SIMD）：单核的并行数据流，一般支持ISA的单核计算机是这个架构。

多指令流单数据流机器（MISD）：多核的串行数据流，理论模型，还没有应用过。

多指令流多数据流机器（MIMD）：多核的并行数据流，目前的大多数多核计算机都是这个分类。

SIMD(**Single instruction, multiple data**) ,即单指令，多数据，是费林分类法下的一种并行处理技术。SIMD 可以提高单个核心并行执行的性能，所以是提升程序并行性能的一个重要的优化方法。SIMD在很多基础软件中有很多重要的应用，如 BloomFilter，哈希函数，加密算法等。

从 1997 年开始出现了面向 x86 架构下的MMX指令扩展，该指令集使用的是 80-bits 寄存器。后来出现了 SSE 系列指令集 SSE1-SSE4.2，使用的是 128bits XMM 寄存器，再后来是 AVX/AVX2 指令集，使用 256bits YMM 寄存器，目前 Intel 平台上支持指令最长的指令集是 AVX512，使用的是 512bits ZMM 寄存器，一次指令支持 512bits 的数据计算。SIMD 的寄存器长度越长，单个指令可以处理的数据量也就越多，所以提高了程序的并行执行效率，从而提升了性能。SIMD 寄存器的长度每增大一倍，一般相应的性能也得到显著提升。

除了 Intel 平台，AMD 也曾经发布了基于 x86 架构的扩展指令集 SSE5。Arm 平台在 ARMv7 也有 NEON sets，一种128-bits的固定长度的指令集; 在 ARMv8 开始支持的 SVEand SVE2 instruction sets (最大到2048 bits)可变长度的指令集。指令集架构简写为`ISA(Instruction Set Architecture)`,大多数Linux服务器中均支持 SSE4_2，AVX/AVX2，AVX512 等指令集。通过`lscpu`中的 flags 查看当前 CPU 支持的指令集架构。

预备知识：IPC,FLOPS

在科学计算中，一般使用 FLOPS 都是衡量处理器的算力，FLOPS 即 Floating Point Operations Per Second，表示每秒钟执行的单精度浮点数操作数。一般理论上最大的 FLOPS 计算如下:
$$
\begin {aligned}
peak\ FLOPS = &FP\ operators\ per\ instructions\ (SIMD\ width) \\  
						&\times instructions\ per\ cycle \\
						&\times cycles\ per\ second\ (frequency)
\end {aligned}
$$
其中，IPC 即 instructions per cycle ，表示每个 CPU 时钟周期内的指令数，处理器并行度越高则 IPC 越大。可以看出在现代 CPU 时钟频率趋于不变时，增大 SIMD 的 width 便可以增加处理器的算力。前提是程序中是否可以利用向量化的技术

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

上面两个函数实现是简单的 int 数组相加。差别在于a是否增加了 __restrict 关键字，\_\_restrict 关键字表明变量a的内存是独占的，不会与其它变量 b,c 重叠或者共享。因此 load 以及 store 指令是互相独立的，可以并行执行，所以编译器可以进行 SIMD 优化。如果不指定\_\_restrict ,则编译器不能保证 a 与 b,c 之间不会存在重叠的部分，所以 load 和 store 指令可能会操作同一块内存，所以 store 必须等待load 执行完成后才可以执行。

从下面的汇编代码来看，就比较明显了。add_restrict 函数直接使用 xmm 寄存器通过 \_mm_add_epi32 实现 SIMD 的优化，但add\_norestrict 函数则不能优化，在上述程序的汇编代码中 rdx 表示的是变量 c，rsi 表示变量 b， rdi 表示变量 a，执行时按顺序将 rdx 寄存器地址的值压入 eax 累加器中，然后 add 指令与 rsi 寄存器地址的值相加，然后将 eax 累加器的结果保存到 rdi 寄存器，也就是变量a 中，一共计算 4 次，相比向量化的版本，未加 \_\_restrict 消耗的 cpu instuctions 要更多。

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

pragma 是编译器提供的帮助编译器自动向量化的指令，比如: pragma GCC ivdep 表示下面的循环没有依赖关系，GCC 编译器可以进行自动向量化编译。

```c++
void add (int * a, const int * b, int n) {
  #pragma GCC ivdep
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}
```

对于 pragma GCC ivdep 而言，GCC 编译器会进行编译时的依赖关系检查决定是否进行自动向量化。除了依赖编译器提供的编译优化选项来优化也可以利用openmp来提示编译器自动向量化。OpenMP 是跨平台共享内存的 API，支持 C, C++, fortran 语言。而 `#pragma omp simd` 可以强制编译器进行自动向量化优化。`#pragma omp simd` 告诉编译器不必检查依赖关系，由用户保证，只需要强制自动向量化即可。需要注意的是，如果想要使用 openmp，在编译时需要加入编译选项 -fopenmp 。并且可以通过 -mavx2 来制定使用 AVX2 指令集。

```c++
void add (int * a, const int * b, int n) {
  #pragma omp simd
  for (int i = 0; i < n; i++)
      a[i] += b[i];
}
```

在程序中，循环展开可以提示编译器更好的进行自动向量化优化。我们可以通过手写下面的代码来手动进行循环展开。在循环内展开有助于减少程序的循环次数，更好的利用 cache line 优化等。在较新的 c++11 之后版本中提供了 `pragma unroll n` 的编译优化提示。GCC编译器也提供了 `pragma GCC unroll n` 提示编译器进行循环展开优化。在 clang 编译器也可以使用 `#pragma clang loop unroll_count(n)` 来指定循环展开层数。Clang 编译器中 n 可以指定为 full，由编译器来决定循环展开的层数。

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

以下代码等价于上面的实现，通过 pragma 提示编译器以下循环可以展开，并且可以提示展开的大小，当 n=0 或者 1 时，表示阻止编译器循环展开。这样的写法使得程序更加简洁。同样，你也可以使用 GCC 的 function attribute`__attribute__((optimize("unroll-loops", "O3"))) `或者 `#pragma GCC optimize ("unroll-loops")` 指定某一函数是否进行循环展开。需要注意的是以上编译优化是GCC 编译器提供的循环展开优化。循环展开的层数一般不能超过寄存器的长度，比如 YMM 寄存器的最大长度是 256bits，所以 unroll loops 一般超过 8 之后可能不会有加速效果。

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

void add(int* a, int* b, int n, int c)
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

```c++
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

3. 检查向量化条件

   在 clang 编译器中，我们可以在编译时加入`-Rpass=loop-vectorize`, `-Rpass-missed=loop-vectorize`  和 `-Rpass-analysis=loop-vectorize `编译参数来查看编译器是否进行了向量化。

   在 gcc 编译器中，可以在编译时加入 `-ftree-vectorizer-verbose` 编译参数来查看是否进行了向量化。

* **\_\_attribute\_\_((target("avx2")))**

指定编译优化指令集

```c++
__attribute__((target("avx2"))) void testavx2(bool * __restrict a, int * __restrict b) {
    for (int i = 0; i < 4096; i++) {
        a[i] = b[i] > 0;
    }
}
__attribute__((target("avx512f"))) void testavx512f(bool * __restrict a, int * __restrict b) {
    for (int i = 0; i < 4096; i++) {
        a[i] = b[i] > 0;
    }
}
```

在上面的程序中指定编译器选择 avx2 指令集和 avx512f 指令集优化函数执行，从汇编代码中我们可以看到两者的差异。avx2 编译的汇编代码中使用了 ymm 寄存器，并且一次执行 256bit 的数据大小，而 avx512f 编译的汇编代码中使用了 zmm 寄存器，一次执行512bit 的数据大小。

```
testavx2(bool*, int*) [clone .avx2]:                   # @testavx2(bool*, int*) [clone .avx2]
        xor     eax, eax
        vpxor   xmm0, xmm0, xmm0
        vpbroadcastb    ymm1, byte ptr [rip + .LCPI0_1] # ymm1 = [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
.LBB0_1:                                # =>This Inner Loop Header: Depth=1
        vmovdqu ymm2, ymmword ptr [rsi + 4*rax]
        vmovdqu ymm3, ymmword ptr [rsi + 4*rax + 32]
        vmovdqu ymm4, ymmword ptr [rsi + 4*rax + 64]
        vmovdqu ymm5, ymmword ptr [rsi + 4*rax + 96]
        vpcmpgtd        ymm2, ymm2, ymm0
        vpcmpgtd        ymm3, ymm3, ymm0
        vpackssdw       ymm2, ymm2, ymm3
        vpcmpgtd        ymm3, ymm4, ymm0
        vpcmpgtd        ymm4, ymm5, ymm0
        vpackssdw       ymm3, ymm3, ymm4
        vpermq  ymm3, ymm3, 216                 # ymm3 = ymm3[0,2,1,3]
        vpacksswb       ymm3, ymm3, ymm0
        vpand   ymm3, ymm3, ymm1
        vpermq  ymm2, ymm2, 216                 # ymm2 = ymm2[0,2,1,3]
        vpacksswb       ymm2, ymm2, ymm0
        vpand   ymm2, ymm2, ymm1
        vpunpcklqdq     ymm2, ymm2, ymm3        # ymm2 = ymm2[0],ymm3[0],ymm2[2],ymm3[2]
        vpermq  ymm2, ymm2, 216                 # ymm2 = ymm2[0,2,1,3]
        vmovdqu ymmword ptr [rdi + rax], ymm2
        add     rax, 32
        cmp     rax, 4096
        jne     .LBB0_1
        vzeroupper
        ret

testavx512f(bool*, int*) [clone .avx512f]:                # @testavx512f(bool*, int*) [clone .avx512f]
        xor     eax, eax
        vpxor   xmm0, xmm0, xmm0
        vpbroadcastd    zmm1, dword ptr [rip + .LCPI1_0] # zmm1 = [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
.LBB1_1:                                # =>This Inner Loop Header: Depth=1
        vpcmpltd        k1, zmm0, zmmword ptr [rsi + 4*rax]
        vpcmpltd        k2, zmm0, zmmword ptr [rsi + 4*rax + 64]
        vpcmpltd        k3, zmm0, zmmword ptr [rsi + 4*rax + 128]
        vpcmpltd        k4, zmm0, zmmword ptr [rsi + 4*rax + 192]
        vmovdqa32       zmm2 {k1} {z}, zmm1
        vmovdqa32       zmm3 {k2} {z}, zmm1
        vmovdqa32       zmm4 {k3} {z}, zmm1
        vmovdqa32       zmm5 {k4} {z}, zmm1
        vpmovdb xmmword ptr [rdi + rax], zmm2
        vpmovdb xmmword ptr [rdi + rax + 16], zmm3
        vpmovdb xmmword ptr [rdi + rax + 32], zmm4
        vpmovdb xmmword ptr [rdi + rax + 48], zmm5
        add     rax, 64
        cmp     rax, 4096
        jne     .LBB1_1
        vzeroupper
        ret
```

需要注意的是混用传统 SSE 和 AVX 指令集会导致所谓的 SSE-AVX Transition Penalty。AVX2，AVX512 混用也有相应的 Frequency Scaling 问题。

### **SIMD intrinsic**

本章节中的 SIMD 指令集主要以 Intel 服务器为主，在 ARM 服务器中也有对应的指令集，比如 ASIMD， SVE，SVE2等。在最新的 Arm V9 架构中， SVE，SVE2 已经是标配的扩展指令集。

在 Intel 服务器中，SIMD 指令集对应着 XMM, YMM, ZMM 等寄存器。

在 SSE1-SSE4.2 指令集中，主要使用的是 128bits 的 XMM 寄存器，寄存器包括 XMM0-XMM15 共16个寄存器。

在AVX/AVX2中使用的是256bits的 YMM 寄存器，寄存器也包括 YMM0-YMM15 共 16 个寄存器。YMM 寄存器的前 128bits 就是 XMM 寄存器。

AVX512F，AVX512L，AVX512DQ 等使用的是 512bits 的 ZMM 寄存器。ZMM 寄存器的前 256bits 是 YMM 寄存器。

SSE 为了兼容 AVX，所以实际上 YMM0 的前 128bits 就是 XMM0，YMM1 的前 128bits 就是XMM1，以此类推。

在 x86-64 平台上，可以使用 GCC 内置的 `__builtin_cpu_supports`  函数在编译期确定 CPU 支持哪种指令集。比如`__builtin_cpu_supports("avx512f")` 如果是 `true` 表示 CPU 支持 avx512f 指令集。

我们下面主要以 AVX/AVX2 的 SIMD 指令集来讲解。AVX/AVX2 指令的命名规则一般如下 ：`_mm{128/256}\_{operator}\_{ps/pd/epi{xx}/si128/si256}`。

SIMD 支持多种数据类型，包括单精度浮点数 float ，双精度浮点数 double ，8/16/32/64/128/256 位整型。

ps 表示计算类型是单精度浮点数

pd 表示计算类型是多精度浮点数

epi8/16/32/64 表示计算类型是有符号 8/16/32/64 位整型

epu8/16/32/64 表示计算类型是无符号 8/16/32/64 位整型

si128/256 表示计算类型是128/256 位整型

在代码中\_\_m256 表示的是浮点数，\_\_mm256i 表示的是整型类型。\_\_mm256d 表示的是双精度浮点数。所以比如_\_mm256_set1_ps(float) ,表示的是用 8 个相同的单精度浮点数填充整个 YMM 寄存器。其效果等同于 \_mm256_set_ps(float, float, float, float, float, float, float, float) 。

**operator** 包括多种操作类型，比如add表示两个寄存器相加。load 表示读取，store 表示存储。extract 表示提取寄存器中某一个数值。通常我们可以通过 \_mm\_extract\_epiXXX 函数来提取某一个整型值。除此之外，还有 gather，shuffle，permutation，mask等多种指令类型可供选择。

```c++
_mm256_set_ps : 设置8个float填充YMM，返回__m256 
_mm256_setr_ps : 以reverse的形式填充YMM，返回__m256
_mm256_add_ps : 计算两个__m256相加的结果
_mm256_movemask_ps: 计算单精度浮点数向量的所有符号位，返回值为int，比如0b00001110 表示第2，3，4为positive
```

让我们从下面的 SIMD 程序来揭开手写 SIMD 的序幕吧。

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

上面的程序展示了如何使用 AVX2 指令集实现两个向量的相加。首先通过 \_mm256_setr_ps 定义两个 \_\_m256 的向量，通过_mm256_add_ps 将向量相加，最后通过 \_mm256_store_ps 将相加后的结果拷贝到float数组中。从程序中可以看到我们只用了1个 SIMD 指令就完成了8个 float 数据的相加，这也是 SIMD 高效的原因。

上面的程序中主要用到 \_mm256\_setr\_ps，\_mm256\_add\_ps，\_mm256\_store\_ps 三个指令。

\_mm256\_setr\_ps 表示使用8个 float 类型以 reverse 的方式填充 YMM 寄存器。在寄存器中高位字节存在高位地址，低位字节存在低位地址。但Intel的内存字节序为小端序。高位字节在高位地址，低位字节在低位地址。所以对于数组中的元素，前面的元素存在低位地址。比如`int a[] = {1,2,3,4}`在内存中1的地址是最小的，所以当存入寄存器时，应该存在 YMM 寄存器的低位地址上。也就是_mm256_set_ps 的最后一个元素。这与我们的直觉相反，所以我们可以使用 \_mm256_setr_ps 来代替，寄存器中的顺序跟数组的顺序就是一致的。

\_mm256\_add\_ps 表示将两个寄存器中的\_\_m256相加，并将结果保存在寄存器中，这一过程只需要一个 Instructions 即可完成，对应的汇编指令是`vaddps ymm0, ymm0, ymm1`。

\_mm256\_store\_ps 表示将寄存器中的结果保存到内存中，同样的也可以使用 \_mm256\_stream\_ps 表示绕过 cache 直接写入内存中，如果当前数据没有立即被使用时其性能会更好。

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

下面的程序中首先通过 \_mm256_cmpgt_epi32 来获取大于0的 mask 标记位，mask 实际上是一个0和1组成的 __m256i 变量，通过\_mm256_and_si256 得到满足 mask 条件的所有 result 结果，并将其与 res 累加起来，最后通过 hsum 函数将 result 中的 8 个 int 累加起来，实现的方式是想将高128位和低128位提取出来相加，然后执行一次128位的 hadd，此时只需要将前 64 位的两个 int 相加，即可得到整个向量的和。通过下面的向量化实现，我们可以获得与编译器自动向量化版本相等的性能结果。比不开启自动向量化优化性能有6倍的提升。

![image-20220817164313346](/Users/jewisliu/Public/hpc/hpc/cpu/pic/2.png)

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

filter 是数据库中常见的算子，filter 实现的功能是将满足一定条件的 input 数据输出到 output 中。非向量化的代码实现如下：

```c++
int filter(int * input, int bond, int * output) {
    int k = 0;

    for (int i = 0; i < N; i++)
        if (input[i] < bond)
            output[k++] = input[i];

    return k;
}
```



我们可以使用 `_mm256_permutevar8x32_epi32` 来实现 filter 的向量化。向量化实现时需要建立一个 lookup table。lookup table 中保存的是0-255的二进制对应的偏移量数组，比如对于13，二进制是00001101，对应的数组是{0,2,3,0,0,0,0,0}。这里只有前三个值是有效的。

可以按照以下步骤对上述的filter函数进行向量化实现。

1. `_mm256_cmpgt_epi32`对8个整型值与P进行比较，得到比较后的mask向量
2. `_mm256_movemask_ps`讲mask向量转换成8-bit的整型值。
3. 根据8-bit整型值查找lookup table得到得到需要排序的permutation向量

4. `_mm256_permutevar8x32_epi32`函数通过permutation向量来重新排列input向量。比如对于一个数组{1,2,3,4,5,6,7,8}通过{0,2,3,0,0,0,0,0}数组重排得到新的值为{1,3,4,1,1,1,1,1}。只有前三位是有效的。
5. 将结果保存在output向量中，其中有部分结果是不需要的。
6. `__builtin_popcount`计算出新的偏移量k,下一次保存到output时以第k地址所在位置开始保存。

向量化版本的实现中无论选择率的大小，性能都比较稳定。因为向量化实现中去除了条件选择判断，并且实现了比非向量化版本更好的性能。

```c++
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

```c++
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

```c++
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

### 

不同类型的 CPU 指令的耗时：https://norvig.com/21-days.html#answers

| execute typical instruction         | 1/1,000,000,000 sec = 1 nanosec        |
| ----------------------------------- | -------------------------------------- |
| fetch from L1 cache memory          | 0.5 nanosec                            |
| branch misprediction                | 5 nanosec                              |
| fetch from L2 cache memory          | 7 nanosec                              |
| Mutex lock/unlock                   | 25 nanosec                             |
| fetch from main memory              | 100 nanosec                            |
| send 2K bytes over 1Gbps network    | 20,000 nanosec                         |
| read 1MB sequentially from memory   | 250,000 nanosec                        |
| fetch from new disk location (seek) | 8,000,000 nanosec                      |
| read 1MB sequentially from disk     | 20,000,000 nanosec                     |
| send packet US to Europe and back   | 150 milliseconds = 150,000,000 nanosec |



**reference** ：

https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html （build_in 内置指令）

https://www.ibm.com/docs/en/zos/2.4.0?topic=pragmas-individual-pragma-descriptions (pragma优化)

https://gcc.gnu.org/onlinedocs/gcc-4.5.2/gcc/Optimize-Options.html（optimize options）

http://svmoore.pbworks.com/w/file/fetch/70583970/VectorOps.pdf

https://sites.cs.ucsb.edu/~tyang/class/240a17/slides/SIMD.pdf

http://web.eecs.utk.edu/~jplank/plank/classes/cs494/494/notes/SIMD/index.html

https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX

http://ftp.cvut.cz/kernel/people/geoff/cell/ps3-linux-docs/CellProgrammingTutorial/BasicsOfSIMDProgramming.html