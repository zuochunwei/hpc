## CPU

* CPU架构演进（SMP、NUMA）

  CPU的频率达到瓶颈后转向多核的发展方向，而CPU多核之间主要存在两种架构体系分别是SMP和NUMA架构，通过lscpu即可查看CPU的numa信息。

  **SMP(Symmetric Multi-Processor)**

  对称多处理器结构的主要特征是共享，即所有CPU共享所有的资源（总线，内存，I/O）等。操作系统管理着一个队列，每个处理器依次处理队列中的进程。如果两个处理器同时请求访问一个资源（例如同一段内存地址），由硬件、软件的锁机制去解决资源争用问题。这同时也预示着SMP最大的问题就是扩展性，CPU利用率很快便达到了瓶颈。

  **NUMA(Non-Uniform Memory Access)**

  NUMA解决了SMP扩展性的问题。NUMA的主要特征是将CPU进行分组，每组CPU共享内部的内存，I/O资源。不同组别之间的CPU通过CPU内互联网络连接。从而解决了CPU共享的问题。不过这也造成了CPU访问远端内存的速度要远小于访问本地的内存。



* CPU pipeline

  **标量处理器**CPU内部一般采用标量流水线技术，即一个CPU指令一般分为五个阶段：取址(IF)，译码(ID)，执行(EX)，访存(MEM)，写回(WB)。所以一般一个CPU指令需要4～5个时钟周期(取决于是否需要访存)。如果指令间没有依赖关系，则多个指令可以并行执行，而当指令间存在依赖时，则后一个指令必须等待前一个指令执行完才可以执行。

  ![img](/Users/jewisliu/Public/hpc/hpc/pic/1.png)**超标量处理器**是在单个处理器内一种称为指令级并行的并行形式的CPU。与每个时钟周期最多可以执行一条指令的标量处理器相比，超标量处理器可以通过同时将多条指令分派到处理器上的不同执行单元来在一个时钟周期内执行多条指令。

  

* SIMD

  CPU除了多核化的优化外，另一个比较重要的优化便是SIMD。SIMD在很多基础软件中有很多重要的应用，如BloomFilter，加密算法等。

  SIMD(**Single instruction, multiple data**),即单指令，多数据，是费林分类法(Flynn's taxonomy)下的一种并行处理技术。从1997年面向x86架构下的MMX指令扩展（80-bits 寄存器），到后来的SSE1-SSE4.2(128bits XMM寄存器)，AVX/AVX2（256bits YMM寄存器）， AVX512（512bits ZMM寄存器），SIMD寄存器的长度每增大一倍，一般相应的性能也得到显著提升。除了Intel平台，Arm平台在ARMv7也有NEON sets，一种128-bits的固定长度的指令集; 在ARMv8开始支持的SVEand SVE2 instruction sets(最大到2048 bits)可变长度的指令集。

  **自动向量化技术**

  在GCC编译器中,指定优化选项 `-O3` or `-ftree-vectorize` 即可开启自动向量化。

  __restrict__ 关键字告诉编译器表示变量之间的内存不存在重叠，可以进行向量化优化。

  ```c++
  void add(int * __restrict__ a, const int * __restrict__ b, int n) {
      for (int i = 0; i < n; i++)
          a[i] += b[i];
  }
  ```

  Pragma帮助编译器自动向量化，pragma GCC ivdep表示下面的循环没有依赖关系，编译器可以进行自动向量化编译。pragma GCC unroll n,表示帮助编译器循环展开。

  ```c++
  void add (int * a, const int * b, int n) {
    #pragma GCC ivdep
  	for (int i = 0; i < n; i++)
      a[i] += b[i]; 
  }
  ```

  ```c++
  void add (int * a, const int * b, int n) {
    int i = 0;
  	for (i = 0; i < n - 3; i+=4) {
  		a[i] += b[i];
      a[i+1] += b[i+1];
      a[i+2] += b[i+2];
      a[i+3] += b[i+3];
  	}
  	
  	while (i < n) { a[i] += b[i]; }
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

  常用SIMD指令：SIMD的命名规则：_mm{128/256}\_{operator}\_{ps/pd/epi{xx}/si/128/si256}

  ```c++
  _mm256_set1_ps
  _mm256_setr_ps
  _mm256_add_ps
  _mm256_store_ps
  _mm256_movemask_ps
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
      float a[8] = { 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };
      float b[8] = { 1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0 };
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

  

* Out-of-Order Execution

  为了最大化流水线的执行效率，CPU通过乱序执行来优化并发执行的效率。乱序执行分为两种情况：

  1. 在编译期，编译器进行指令重排。
  2. 在运行期，CPU 进行指令乱序执行。

* branch prediction

  Intel的CPU的乱序执行通常会让一些指令提前执行，而分支预测则有可能打破这种秩序。如果分支预测成功，则乱序执行会提升程序的执行效率，而当分支预测失败，反而会导致CPU返回计算错误的分支。如果保持较高的分支预测效率，分支预测便可以提升流水线的性能。

  

* memory barrier/memory fence/CPU fence
* （乱序执行）CPU一致性storebuffer，InvalidQueue（ARM VS INTEL）

* context switch

* lock（shared rw lock & unqiue lock） vs atomic

  一般来说锁的开销要大于原子操作，常见的锁分为自旋锁，互斥锁，读写锁。自旋锁通过用于较短时间的锁，因为它会长时间占用CPU，互斥锁的原理是将CPU置于等待队列中，等待唤醒，所以 常用于较长时间的上锁。读写锁更多用于读多写少的场景。

* Perf

  perf top

  perf record -a -e cycles(cpu cycles event)



* cpu clock/ timer

  同样，我们要对一段程序进行性能测试时，需要记录程序运行的时间，在Linux平台上有多种计时工具，常见的如clock, gettimeofday, clock_gettime, std::chrono::system_clock, std:;chrono::steady_clock, std::chrono::high_resolution_clock, rdtsc。在所有的计时工具中，std::chrono的稳定性和精度均为良好并且跨平台性最好(C++11标准)，rdtsc精度最高，速度最快，稳定性最好 。我们所有的性能测试均采用std::chronologically::high_resolution_clock的计时方式来测试性能。













reference：

http://svmoore.pbworks.com/w/file/fetch/70583970/VectorOps.pdf

https://sites.cs.ucsb.edu/~tyang/class/240a17/slides/SIMD.pdf

http://web.eecs.utk.edu/~jplank/plank/classes/cs494/494/notes/SIMD/index.html

https://www.codeproject.com/Articles/874396/Crunching-Numbers-with-AVX-and-AVX

http://ftp.cvut.cz/kernel/people/geoff/cell/ps3-linux-docs/CellProgrammingTutorial/BasicsOfSIMDProgramming.html
