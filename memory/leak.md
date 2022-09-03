## 内存泄漏
何谓内存泄漏？动态申请的内存丢失引用，造成没有办法回收它（进程退出的时候会统一回收进程资源），这便是内存泄漏。

### 怎么查内存泄漏？

我们可以review代码，但从海量代码里找到隐藏的问题，这如同大海捞针。

所以，我们需要借助工具，比如valgrind，但这些找内存泄漏的工具，往往对你使用动态内存的方式有某种期待，或者说约束，比如常驻内存的对象会被误报出来，然后真正有用的信息会掩盖在误报的汪洋大海里。很多时候，甚至valgrind根本解决不了日常项目中的问题。

所以很多著名的开源项目，为了能用valgrind跑，都费大力气，大幅修改源代码，从而使得项目符合valgrind的要求，满足这些要求，用vargrind跑完没有任何报警的项目叫valgrind干净。

下面介绍一种通过wrap malloc/free定位C/C++内存泄漏的的方法：

### 怎么去定位内存泄漏呢？
malloc各种不同size的chunk，也就是每种不同size的chunk会有不同数量，如果我们能够跟踪每种size的chunk数量，那就可以知道哪种size的chunk在泄漏。
很简单，如果该size的chunk数量一直在增长，那它很可能泄漏了。

光知道某种size的chunk泄漏了还不够，我们得知道是哪个调用路径上导致该size的chunk被分配，从而去检查是不是正确释放了。

### 怎么跟踪到每种size的chunk数量？

我们可以维护一个全局 unsigned int malloc_map[1024 * 1024]数组，该数组的下标就是chunk的size，malloc_map[size]的值就对应到该size的chunk分配量。

这等于维护了一个chunk size到chunk count的映射表，它足够快，而且它可以覆盖到0 ~ 1M大小的chunk的范围，它已经足够大了，试想一次分配一兆的块已经很恐怖了，可以覆盖到大部分场景。

那大于1M的块怎么办呢？我们可以通过log记录下来。

在__wrap_malloc里，++malloc_map[size]

在__wrap_free里，--malloc_map[size]

很简单，我们通过malloc_map记录了各size的chunk的分配量。

### 如何知道释放的chunk的size？

不对，free(void *p)只有一个参数，我如何知道释放的chunk的size呢？怎么办？

我们通过在__wrap_malloc(size_t)的时候，分配8+size的chunk，也就是多分配8字节，开始的8字节存储该chunk的size，然后返回的是(char*)chunk + 8，也就是偏移8个字节返回给调用malloc的应用程序。

这样在free的时候，传入参数void* p，我们把p往前移动8个字节，解引用就能得到该chunk的大小，而该大小值就是前一步，在__wrap_malloc的时候设置的size。

好了，我们真正做到记录各size的chunk数量了，它就存在于malloc_map[1M]的数组中，假设64个字节的chunk一直在被分配，数量一直在增长，我们觉得该size的chunk很有可能泄漏，那怎么定位到是哪里调用过来的呢？

### 如何记录调用链？

我们可以维护一个toplist数组，该数组假设有10个元素，它保存的是chunk数最大的10种size，这个很容易做到，通过对malloc_map取top 10就行。

然后我们在__wrap_malloc(size_t)里，测试该size是不是toplist之一，如果是的话，那我们通过glibc的backtrace把调用堆栈dump到log文件里去。

注意：这里不能再分配内存，所以你只能使用backtrace，而不能使用backtrace_symbols，这样你只能得到调用堆栈的符号地址，而不是符号名。

### 如何把符号地址转换成符号名，也就是对应到代码行呢？
addr2line
addr2line工具可以做到，你可以追查到调用链，进而定位到内存泄漏的问题。
至此，你已经get到了整个核心思想。

当然，实际项目中，我们做的更多，我们不仅仅记录了toplist size，还记录了各size chunk的增量toplist，会记录大块的malloc/free，会wrap更多的API。

总结一下：通过wrap malloc/free + backtrace + addr2line，你就可以定位到内存泄漏了，当然，上面的方法，你还需要处理多线程的问题，不过它不是一个大问题。

