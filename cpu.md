## CPU

* CPU架构演进（SMP、NUMA）

* SIMD指令集/标量指令集

SIMD(**Single instruction, multiple data**),即单指令，多数据，是Flynn's taxonomy下的一种并行处理技术。从1997年面向x86架构下的MMX指令扩展（80-bits 寄存器），到后来的SSE1-SSE4.2(128bits XMM寄存器)，AVX/AVX2（256bits YMM寄存器）， AVX512（512bits ZMM寄存器），SIMD寄存器的长度每增大一倍，一般相应的性能也得到显著提升。出了Intel平台，Arm平台在ARMv7也有NEON sets，一种128-bits的固定长度的指令集; 在ARMv8开始支持的SVEand SVE2 instruction sets(最大到2048 bits)可变长度的指令集。



自动向量化技术

在GCC编译器中,指定优化选项 `-O3` or `-ftree-vectorize` 即可开启自动向量化。



常用SIMD指令：

```
_mm256_setr_ps
_mm256_add_ps
```

第一个SIMD程序

```
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

__m256 __attribute__((aligned(32))) vec[2];
int main()
{
    float a[8] = { 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };
    float b[8] = { 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };

    const __m256 __attribute__((aligned(32))) va = _mm256_setr_ps(1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0);
    const __m256 __attribute__((aligned(32))) vb = _mm256_setr_ps(1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0);

    __m256 __attribute__((aligned(32))) vc = _mm256_add_ps(va, vb);

    std::cout << vc << std::endl;

}
```



* memory barrier/memory fence
* （乱序执行）CPU一致性storebuffer（ARM VS INTEL）
* CPU pipeline
* branchless分支预测
* thread context switch
* lock（shared rw lock & unqiue lock） vs atomic

* Perf

* cpu clock/ timer

  











include <immintrin.h>

http://svmoore.pbworks.com/w/file/fetch/70583970/VectorOps.pdf

https://sites.cs.ucsb.edu/~tyang/class/240a17/slides/SIMD.pdf

http://web.eecs.utk.edu/~jplank/plank/classes/cs494/494/notes/SIMD/index.html

https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX

http://ftp.cvut.cz/kernel/people/geoff/cell/ps3-linux-docs/CellProgrammingTutorial/BasicsOfSIMDProgramming.html
