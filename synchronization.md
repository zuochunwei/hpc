# 线程同步
进程内的多个线程共享同一虚拟地址空间，每个线程都是一个独立的执行流，所以，多个线程对应多个执行流，多个执行流会出现竞争同一资源的情况，资源包括内存数据、打开的文件句柄、套接字等，如果不加以控制和协调，则有可能出现数据不一致，而这种数据不一致可能导致结果错误，甚至程序奔溃。

并发编程的错误非常诡谲且难以定位，它总是隐藏在某个角落，大多数时候，程序运转良好，等代码交付上线后，会莫名其妙的出错，就像墨菲定律描述的那样：凡可能出错，就一定出错。

## 数据不一致源于什么？
1. CPU与内存访问性能差距很大，CPU一纳秒能执行10条指令，而从内存取数据需要耗费几十纳秒。因此，Cache被作为内存的缓存插入到CPU与内存之间，数据会在Cache里缓存内存数据的副本，内存数据与缓存数据不总是一致。
    - 比如修改变量（写入数据），如果采取写穿透（Write Through）的方式，则会在更新缓存中对应的Cache Line的同时把数据写入内存，如果数据不在缓存，则直接写入内存，但每次写操作都写入内存，而内存的访问时延通常高达几十个指令周期，这种写的方式性能太低。
    - 而采用写回（Write Back）的方式，如数据在Local Cache里，则更新缓存后就直接返回，这样就减少访问内存的频率，也就提升了性能，但这样的话，内存数据和Cache里的数据是不一致的。
2. 现代处理器朝着多CPU多核架构发展，每个核有自己的L1/L2 Cache ，核之间共享L3 Cache，然后再通过总线连接内存，内存被所有CPU/Core所共享，所以，一个内存数据会被多个CPU/Core Cache，不仅内存与Cache中的数据可能不一致，Cache里的多份拷贝也会不一致，Cache一致性协议用于处理Cache的数据一致性问题。

## CPU如何使用内存数据？
- CPU通常不会直接操作内存
    - 这是因为有些指令对操作数有限制，比如X86-64限制mov指令的源和目的操作数不能都是内存地址，所以把一个字节从一个内存地址复制到另一个内存地址，需要两条mov汇编指令，先从源地址move到寄存器，再从寄存器move到目标地址
    - 即使mov的一个操作数是内存地址，实际上，CPU处理的时候，也会先将内存地址的数据加载到Cache Line，再作用于Cache Line，而非直接修改内存
- 多CPU多核系统上，如果Core的local Cache没有对应变量的数据，它并不是只有从内存里加载数据到Cache这一条路，而是会通过CPU/Core间消息，从别的CPU/Core的Cache里拿数据的拷贝，这个核间消息不是通过共享的总线传递，而是基于Interconnect的message passing
- 当某个Core更新Local Cache里的数据时，它需要通过CPU/Core间消息把这个写入操作传播到其他Core的Cache，如果其他Core也Cache了这个数据，要让对应Cache Line失效，这个叫写传播（Write Propagation），总线嗅探通过感知到核间消息来实现写传播
- 另外，某个CPU核心里对数据的操作顺序，必须在其他核心看起来顺序是一样的，这个称为事务的串形化（Transaction Serialization），而做到这一点，则需要CPU的缓存更新需要引入‘锁’的概念，多个核心有相同数据的Cache，那么对于数据的更新，只有拿到锁进行，而基于总线嗅探实现的MESI协议就是为了实现事务串行化，如果一个数据在某个Core的Cache Line是独占（Exclusive）状态，则它相当于拿到了自由修改权，如果一个数据被加载到多个Core的Cache，则是Shared的状态，这时候，需要通过向其他核广播请求，Invalidate其他核里的Cache Line才能修改。

## 为什么需要多线程同步？
我们先用2个例子来描述，如果不做线程同步，程序会出现什么问题。

### 例子1
有一个货物售卖程序，变量int item_num记录某商品的数量，它被初始值为100（代表可售卖数量为100）。售卖函数检查剩余商品数，如果剩余商品数大于等于售卖数量，则扣除商品件数，并返回成功；否则，返回失败。代码如下：

```c++
bool sell(int num) {
    if (item_num >= num) {
        item_num -= num;
        return true;
    }
    return false;
}
```

单线程下，sell函数被多次调用，程序运转良好，结果符合预期，但在多线程环境下，会出错。

为了理解上述代码行为，需要先了解一个基本事实：程序变量存放在内存中，对变量做加减，会先将变量加载到通用寄存器，再执行算术运算（更新寄存器中的值），最后把寄存器的新值存入内存位置。

寄存器里会保存内存变量值的副本，对变量的加减乘除等运算直接作用于副本，而非变量内存位置。不过，如果对变量赋值，则指令会接受一个内存位置作为操作数，指令会直接操作内存位置。

#### 多核并行
- 假设线程t1和线程t2，分别在core1和core2同时执行，t1执行sell(50), t2执行sell(100)。
- 2个线程代表2个执行（指令）流，如果2个执行流同时执行到if (item_num >= num)这一行判断语句。
- 内存位置保存的item_num的值会被分别加载到2个核心的寄存器，因为item的值为100，所以加载到寄存器后也都为100，2个线程的条件检查都顺利通过。
- 线程t1，执行减法运算item_num -= num，参数num为50，结果为100-50=50，更新寄存器。
- 线程t2，执行减法运算item_num -= num，参数num为100，结果为100-100=0，更新寄存器。
- 然后，t1和t2线程先后把各自寄存器里的新值store到内存，后一个线程的store操作会覆盖前值。
- 如果t1线程先store，内存中的item_num被修改为50，t2线程再执行store，内存中的item_num被覆盖，item_num值被替换为0。
- 如果t2线程先store，t1线程后store，则item_num的最终值为50。

无论哪种情况，结果都是错误的，我们只有100件商品，却超卖出150件。

#### 单核并发
- 如果程序在单CPU单Core的机器上运行，t1线程和t2线程并发（非并行）交错执行，因为只有一个CPU，所以同一时刻，只有一个线程在执行。
- 假设t1在CPU上执行，它把item_num(100) load到寄存器，判断通过(100 >= 100)。
- 这时候，发生线程调度（比如t1的时间配额耗尽），t2被调度到CPU上执行，然后t2依次完成load、check、compute和store操作。
- 然后t1又被调度到CPU上恢复执行，t1会直接用寄存器中的item_num副本（100），执行计算，item -= 50的结果为50，更新寄存器，所以t1线程执行后，新值50被store入item_num所在内存。
- 我们期望t1或者t2的sell只有一个操作成功，但结果并非如此。

CPU核相当于工人，而程序线程相当于任务，核的数量决定了并行度，多个核代表多个任务可以被同时执行，但多线程竞争导致数据的不一致，在单核环境也有可能出现。

让我们再看一个计数的例子：

### 例子2
全局变量int count用来计数，我们启动100个线程，每个线程的处理逻辑：在100次循环里累加count；主函数启动线程并等待所有线程执行完成，程序退出前打印count数值，代码如下:

```c++
#include <thread>
#include <iostream>

int count = 0;

void thread_proc() {
    for (int i = 0; i < 100; ++i) {
        ++count;
    }
}

int main() {
    std::thread threads[100];
    for (auto &x : threads) {
        x = std::move(std::thread(thread_proc));
    }
    for (auto &x : threads) {
        x.join();
    }
    std::cout << "count:" << count << std::endl;
    return 0;
}
```
我的机器上打印count:9742，而我们期望的结果是100*100=10000，结果与预期不符，为什么会这样？

简单的一行++count，实际上也包括：从内存加载变量的值到寄存器（load），寄存器中更新值（update），将寄存器中的新值存入内存（store）。

因为多个线程并发/并行执行，而load/update/store的三步不是原子的，不能在一个cpu cycle内完成，所以多个线程可能加载了相同的旧值，在寄存器中分别更新，再写入变量count的内存位置，导致最终值比期望的1000小，而且运行多次，会出现不同的结果。

## 问题出在哪里？
上述2个例子都表明：不加控制的多线程并发/并行访问同一数据，会导致数据不一致，从而产生错误的结果，所以，需要某种同步机制，协调多线程的运行，确保结果的正确性。

上述问题的根因都出在多线程对数据的竞争访问，数据竞争不仅因为对数据的访问不能在一个指令周期内完成，也因为我们经常要先读一个变量值，再基于它的值做决策，而这2个步骤的组合并非原子的。

只有同时满足以下情况，才需要做多线程同步：
- 数据竞争是前提条件，多线程对同一数据的并发访问
- 所有线程对数据只读的话，没有问题，不需要同步，必须有线程对数据进行写（修改），多线程混合读写才需要同步
- 对数据的访问在时间上要同时或者交错（多线程对数据读写在开始结束时间上有重叠），如果时间上能错开，也没有问题
- 需要出现数据不一致的情况，是不能忍受的错误（比如的超卖，计数错误），否则，也不必同步（比如只是记一条粗略统计的日志，计数不准确也没有关系），这一点往往被忽略

## 保护的到底是什么？
- 多线程同步，保护的是数据，而非代码，代码保存在进程的文本段，程序运行过程中，只会从保存文本段的内存位置读取指令序列，而不会修改文本（代码）段。
- 细化一下，我们保护的是全局资源/数据、static局部数据、或者通过指针/引用指向的堆内存上的数据；而普通局部变量则通常不需要保护，因为局部变量位于栈上，每个线程有独立的栈，线程栈是相互隔离的，不通过特殊手段不可互访。

## 多线程同步机制有哪些？
线程间同步的机制有很多，Posix线程同步机制包括互斥锁（Mutex）、读写锁（Reader-Writer Lock）、自旋锁（Spin Lock）、条件变量（Condition Variable）、屏障（Barrier）等。
C/C++、Java等编程语言都有类似的线程同步机制和编程接口，细节上不尽相同，但概念和原理上都是相通的，我们主要考查Posix线程同步机制。

### 互斥锁
互斥锁，某个线程获得锁之后，直到该线程释放锁之前，其他的线程便没法获得锁，表现出排他的特征。

互斥锁，本质上是一个内存变量，无他。互斥锁目的是做串行化，即在同一时间点，访问受保护数据的临界代码，只会有一个执行流正在运行，直到这段临界代码执行完，其他执行流才有机会进入这段临界代码，这就从根本上杜绝了某个数据被多个执行流交错访问。

互斥锁的用法，通常是在访问共享资源前加锁，完成资源访问后再解锁，当申请加锁的时候，锁已经被其他线程持有，那么申请加锁的线程便会阻塞住，当锁被释放（解锁），则阻塞在该锁上的所有线程，都会被唤醒，只有一个线程会加锁成功，其他线程意识到锁处于加锁状态，则又转入阻塞等待解锁，因此，一次只有一个线程会前进。用mutex对sell做同步的代码如下：

```c++
bool sell(int num) {
    mutex.lock();
    if (item_num >= num) {
        item_num -= num;
        mutex.unlock();
        return true;
    }
    mutex.unlock();
    return false;
}
```

加锁保护后，对item_num的判断和减值这2个行代码，将变成密不可分的原子操作，多线程同时调用sell()都不会出现前述的超卖错误。

**RAII**
上面的程序，有一个容易出错的点，那就是在两次返回的时候都需要解锁，如果函数有多个返回点，则需要在每个返回点都小心的解锁，如果逻辑更复杂一点，或者调用的其他函数抛出异常，则很难确保unlock得到正确的调用。

所以，C++提供一个叫RAII的技术，常用来应对这种情况，RAII利用C++临时对象的析构在对象出作用域时一定会得到调用的特性，来提升安全性，通过构建一个临时对象，在对象的构造函数里加锁，在析构函数里解锁，来完成配对的操作，常用于加/解锁，打开/关闭文件句柄，申请/释放资源等操作。

```c++
class LockGuard {
    std::mutex& mutex;
    LockGuard(const LockGuard&) = delete;
    LockGuard(LockGuard&&) = delete;
    LockGuard& operator=(const LockGuard&) = delete;
    LockGuard& operator=(LockGuard&&) = delete;
public:
    LockGuard(std::mutex& mutex) : mutex(mutex) { mutex.lock(); }
    ~LockGuard() { mutex.unlock(); }
};

int item_num = 100;
std::mutex item_num_mutex;

bool sell(int num) {
    LockGuard lg(item_num_mutex);
    if (item_num >= num) {
        item_num -= num;
        return true;
    }
    return false;
}
```

**注意**：
- 通过互斥锁对数据进行保护，需要开发者遵从约束，按某种规定的方式编写数据访问代码，这种约束是对程序员的隐式约束，它并非某种强制的限制。
- 比如使用互斥量对数据x进行并发访问控制，假设有2处代码对该数据竞争访问，其中一处用互斥量做了保护，而另外一处，没有使用互斥锁加以保护，则代码依然能通过编译，程序依然能运行，只是结果上可能是错误，这个跟现实中没有钥匙开锁就不能得到（访问）权限是不一样的。

### 自旋锁
在cpu核上自旋，粘着cpu不停的测试，直到其他cpu解锁，这是一种消耗cpu的加锁方式，适合持有锁的时间非常短的情况，它基于一种假设，睡眠导致的调度开销大于在cpu上自旋测试的开销。

### 读写锁
写排他，读并行，适合多读少写的场景，可以提升吞吐。

### 条件变量（生产者-消费者模式）
需要配合互斥量使用，常用于生产者消费者模式

假设你要编写一个网络处理程序，I/O线程从套接字接收字节流，反序列化后产生一个个消息（自定义协议），然后投递到一个消息队列，一组工作线程负责从消息队列取出并处理消息。

这是典型的生产者-消费者模式，I/O线程生产消息（往队列put），Work线程消费消息（从队列get），I/O线程和Work线程并发访问消息队列，显然，消息队列是竞争资源，需要同步。

我们可以给队列配置互斥锁，put和get操作都先加锁，操作完成再解锁。代码差不多是这样的：

```c++
void io_thread() {
    while (1) {
        select(max_fd, read_fds, write_fds, nullptr, ...);
        Msg* msg = read_msg_from_socket();
        msg_queue_mutex.lock();
        msg_queue.put(msg);
        msg_queue_mutex.unlock();
    }
}

void work_thread() {
    while (1) {
        msg_queue_mutex.lock();
        Msg* msg = msg_queue.get();
        msg_queue_mutex.unlock();
        if (msg != nullptr) {
            process(msg);
        }
    }
}
```

work线程组的每个线程都忙于检查消息队列是否有消息，如果有消息就取一个出来，然后处理消息，如果没有消息就在循环里不停检查，这样的话，即使负载很轻，但work_thread还是会消耗大量的CPU时间，我们当然可以在两次查询之间加入短暂的sleep，从而让出cpu，但是这个睡眠的时间设置为多少合适呢？设置长了的话，会出现消息到来得不到及时处理，设置太短了，还是无辜消耗了CPU资源，这种不断问询的方式在编程上叫轮询。

轮询行为逻辑上，相当于你在等一个投递到楼下小邮局的包裹，你下楼问过没到之后，就上楼去，然后马上又下楼问，你不停的上楼下楼+询问，其实，你大可不必如此，何不等包裹到达以后，让门卫打电话通知你呢？

条件变量提供了一种类似通知的机制，它让两类线程能够在一个点交汇。条件变量能够让线程等待某个条件发生，条件本身受互斥锁保护，因此条件变量必须搭配互斥锁使用，线程在改变条件前先获得锁，然后改变条件状态，再解锁，然后发出通知，等待条件的睡眠中的线程在被唤醒前，必须先获得锁，再判断条件状态，如果条件不成立，则继续转入睡眠并释放锁。

对应到上面的例子，工作线程等待的条件是消息队列非空，用条件变量改写上面的代码：

```c++
void io_thread() {
    while (1) {
        select(max_fd, read_fds, write_fds, nullptr, ...);
        Msg* msg = read_msg_from_socket();
        {
            std::lock_guard<std::mutex> lock(msg_queue_mutex);
            msg_queue.push_back(msg);
        }
        msg_queue_not_empty.notify_all();
    }
}

void work_thread() {
    while (1) {
        Msg* msg = nullptr;
        {
            std::unique_lock<std::mutex> lock(msg_queue_mutex);
            msg_queue_not_empty.wait(lock, []{ return !msg_queue.empty(); });
            msg = msg_queue.get();
        }
        process(msg);
    }

}
```
lock_guard是互斥量的一个RAII包装类，unique_lock除了会在析构函数自动解锁外，还支持主动unlock调用。

生产者在往msg_queue投递消息的时候，需要对msg_queue加锁，通知work线程的代码可以放在解锁之后，等待msg_queue_not_empty条件必须受msg_queue_mutex保护，wait的第二个参数是一个lambda表达式，因为会有多个work线程被唤醒，线程被唤醒后，会重新获得锁，检查条件，如果不成立，则再次睡眠，条件变量的使用必须非常谨慎，否则容易出现不能唤醒的情况。

Posix条件变量的编程接口跟C++的类似，概念上是一致的。

### 原子操作
原子操作，从语义上理解，既原子性的执行一系列操作，这一系列操作是密不可分的整体，要么都做，要么都不做，中间也不会被穿插进其他操作。

比如对一个整型变量自增（++count），对长度不大于机器字长，且满足数据类型自身对齐要求的变量的Load和Store操作，仅需要一条指令即可完成，都是原子操作（一条指令完成是原子操作的必要非充分条件），其他core无法观察到中间状态，但因为++count需要执行Load/Update/Store三步骤，多核环境下，这三步是可以相互穿插的，不是原子操作。

C++提供一种类型为std::atomic<T>的类模板，它提供++/--/+=/-=/fetch_sub/fetch_add等原子操作。

原子操作是编写Lock-free多线程程序的基础，原子操作只保证原子性，不保证操作顺序。在Lock-free多线程程序中，光有原子操作是不够的，需要将原子操作和Memory Barrier结合起来，才能实现免锁。

原子操作常用于与顺序无关的场景，比如前面例子中的计数，用原子变量改写后，则会输出符合预期的count=10000。

```c++
std::atomic<int> count = 0;

void thread_proc() {
    for (int i = 0; i < 100; ++i) {
        ++count; // 原子的自增
    }
}
```

### 程序序（Program Order）
对单线程程序而言，代码会一行行顺序执行，就像我们编写的程序的顺序那样。比如
```c++
a = 1;
b = 2;
```
会先执行`a = 1`，再执行`b = 2`，从程序角度看到的代码行依次执行叫程序序，我们在此基础上构建软件，以此作为讨论的基础。

### 内存序（Memory Order）
与程序序相对应的内存序，是指从某个角度观察到的对于内存的读和写所真正发生的顺序。

内存操作顺序并不唯一，在一个包含core0和core1的CPU中，core0和core1有着各自的内存操作顺序，这两个内存操作顺序不一定相同。

从包含多个Core的CPU的视角看到的全局内存操作顺序（Global Memory Order）跟单core视角看到的内存操作顺序亦不同，而这种不同，对于有些程序逻辑而言，是不可接受的，例如：

程序序要求`a = 1`在`b = 2`之前执行，但内存操作顺序可能并非如此，对a赋值1并不确保发生在对b赋值2之前。
- 如果编译器认为对b赋值没有依赖对a赋值，那它完全可以在编译期为了性能调整编译后的汇编指令顺序
- 即使编译器不做调整，到了执行期，也有可能对b的赋值先于对a赋值执行

虽然对一个Core而言，如上所述，这个Core观察到的内存操作顺序不一定符合程序序，但内存操作序和程序序必定产生相同的结果，无论在单Core上对a、b的赋值哪个先发生，结果上都是a被赋值为1、b被赋值为2，如果单核上，乱序执行会影响结果，那编译器的指令重排和CPU乱序执行便不会发生，硬件会提供这项保证。

但多核系统，硬件不提供这样的保证，多线程程序中，每个线程所工作的Core观察到的不同内存操作序，以及这些顺序与全局内存序的差异，常常导致多线程同步失败，所以，需要有同步机制确保内存序与程序序的一致，内存屏障（Memory Barrier）的引入，就是为了解决这个问题，它让不同的Core之间，以及Core与全局内存序达成一致。

### 乱序执行（Out-of-order Execution）
乱序执行会引起内存顺序跟程序顺序不同，乱序执行的原因是多方面的，比如编译器指令重排、超标量指令流水线、预测执行、Cache-Miss等。内存操作顺序无非精确匹配程序顺序，这有可能带来混乱，既然有副作用，那为什么还需要乱序执行呢？

答案是为了性能。

我们先看看没有乱序执行之前，早期的有序处理器（In-order Processors）是怎么处理指令的？
- 指令获取，从代码节内存区域加载指令到I-Cache
- 译码
- 如果指令操作数可用（例如操作数位于寄存器中），则分发指令到对应功能模块中；如果操作数不可用，通常是需要从内存加载，则处理器会stall，一直等到它们就绪，知道数据被加载到cache或拷贝进寄存器
- 指令被功能单元执行
- 功能单元将结果写回寄存器或内存位置

乱序处理器（Out-of-order Processors）又是怎么处理指令的呢？
- 指令获取，从代码节内存区域加载指令到I-Cache
- 译码
- 分发指令到指令队列
- 指令在指令队列中等待，一旦操作数就绪，指令就离开指令队列，那怕它之前的指令未被执行（乱序）
- 指令被派往功能单元并被执行
- 执行结果放入队列（Store Buffer），而不是直接写入Cache
- 只有更早请求执行的指令结果写入cache后，指令执行结果才写入cache，通过对指令结果排序写入cache，使得执行看起来是有序的。

指令乱序执行是结果，但原因并非只有CPU的乱序执行，而是由两种因素导致：
- 编译期：指令重排（编译器），编译器会为了性能而对指令重排，源码上先后的两行，被编译器编译后，可能调换指令顺序，但编译器会基于一套规则做指令重排，有明显依赖的指令不会被随意重排，指令重排不能破坏程序逻辑。
- 运行期：乱序执行（CPU），CPU的超标量流水线、以及预测执行、Cache-Miss等都有可能导致指令乱序执行，也就是说，后面的指令有可能先于前面的指令执行。

### Store Buffer

**为什么需要Store Buffer？**
前面已经提到了Store Buffer，考虑下面的代码：

```c
void set_a()
{
    a = 1;
}
```

- 假设运行在core0上的set_a()对整型变量a赋值1，计算机通常不会直接写穿通到内存，而是会在Cache中修改对应Cache Line
- 如果Core0的Cache里没有a，赋值操作（store）会造成Cache Miss
- Core0会stall在等待Cache就绪（比如从内存加载变量a到对应的Cache Line），但Stall会损害CPU性能，相当于CPU在这里停顿，白白浪费着宝贵的CPU时间
- 有了Store Buffer，当变量在Cache中没有就位的时候，就先Buffer住这个Store操作，而Store操作一旦进入Store Buffer，core便认为自己Store完成，当随后Cache就位，store会自动写入对应cache。

所以，我们需要Store Buffer，每个Core都有独立的Store Buffer，每个Core都访问私有的Store Buffer, Store Buffer帮助CPU遮掩了Store操作带来的延迟。

**Store Buffer会带来什么问题？**
```c++
a = 1;
b = 2;
assert(a == 1);
```
上面的代码，断言a==1的时候，需要读（load）变量a的值，而如果a在被赋值前就在Cache中，就会从Cache中读到a的旧值（可能是1之外的其他值），所以断言就可能失败。

但这样的结果显然是不能接受的，它违背了最直观的程序顺序性。

问题出在变量a除保存在内存外，还有2份拷贝，一份在Store Buffer里，一份在Cache里，如果不考虑这2份拷贝的关系，就会出现数据不一致。那怎么修复这个问题呢？

可以通过在Core Load数据的时候，先检查Store Buffer中是否有悬而未决的a的新值，如果有，则取新值；否则从cache取a的副本。这种技术在多级流水线CPU设计的时候就经常使用，叫Store Forwarding。有了Store Buffer Forwarding，就能确保单核程序的执行遵从程序顺序性，但多核还是有问题，让我们考查下面的程序：

```c++
int a = 0; // 被CPU1 CACHE
int b = 0; // 被CPU0 CACHE

// CPU0执行
void x() {
    a = 1;
    b = 2;
}

// CPU1执行
void y() {
    while (b);
    assert(a == 1);
}
```

假设a和b都被初始化为0；CPU0执行x()函数，CPU1执行y()函数；变量a在CPU1的local Cache里，变量b在CPU0的local Cache里。
    - CPU0执行`a = 1;`的时候，因为a不在CPU0的local cache，CPU0会把a的新值1写入Store Buffer里，并发送Read Invalidate消息给其他CPU
    - CPU1执行`while (b);`,因为b不在CPU1的local cache里，CPU1会发送Read Invalidate消息去其他CPU获取b的值
    - CPU0执行`b = 2;`，因为b在CPU0的local Cache，所以直接更新local cache中b的副本
    - CPU0收到CPU1发来的读b请求，把b的新值（2）发送给CPU1；同时存放b的Cache Line的状态被设置为Shared，以反应b同时被CPU0和CPU1 cache住的事实
    - CPU1收到b的新值（2）后结束循环，继续执行`assert(a == 1);`，因为此时local Cache中的a值为0，所以断言失败
    - CPU1收到CPU0发来的Read Invalidate后，更新a的值为1，但为时已晚，程序在上一步已经崩了

怎么办？答案留到内存屏障一节揭晓。

### Invalidate Queue
**为什么需要Invalidate Queue**
当一个变量加载到多个core的Cache，则这个CacheLine处于Shared状态，如果Core1要修改这个变量，则需要通过发送核间消息Invalidate来通知其他Core把对应的Cache Line置为Invalid，当其他Core都Invalid这个CacheLine后，则本Core获得该变量的独占权，这个时候就可以修改它了。

收到Invalidate消息的core需要回Invalidate ACK，一个个core都这样ACK，等所有core都回复完，Core1才能修改它，这样CPU就白白浪费。

事实上，其他核在收到Invalidate消息后，会把Invalidate消息缓存到Invalidate Queue，并立即回复ACK，真正Invalidate动作可以延后再做，这样一方面因为Core可以快速返回别的Core发出的Invalidate请求，不会导致发生Invalidate请求的Core不必要的Stall，另一方面也提供了进一步优化可能，比如在一个CacheLine里的多个变量的Invalidate可以攒一次做了。

但写Store Buffer的方式其实是Write Invalidate，它并非立即写入内存，如果其他核此时从内存读数，则有可能不一致。

### 内存屏障
那有没有方法确保对b的赋值一定先于对a的赋值呢？有，内存屏障被用来提供这个保障。

内存屏障（Memory Barrier），也称内存栅栏、屏障指令等，是一类同步屏障指令，是CPU或编译器在对内存随机访问的操作中的一个同步点，同步点之前的所有读写操作都执行后，才可以开始执行此点之后的操作。

语义上，内存屏障之前的所有写操作都要写入内存；内存屏障之后的读操作都可以获得同步屏障之前的写操作的结果。

内存屏障，其实就是提供一种机制，确保代码里写顺序写下的多行，会按照书写的顺序，被存入内存，主要是解决StoreBuffer引入导致的写入内存间隙的问题。

```c++
void x() {
    a = 1;
    wmb();
    b = 2;
}
```
像上面那样在a=1后、b=2前插入一条内存屏障语句，就能确保a=1先于b=2生效，从而解决了内存乱序访问问题，那插入的这句smp_mb()，到底会干什么呢？

回忆前面的流程，CPU0在执行完`a = 1`之后，执行smp_mb()操作，这时候，它会给Store Buffer里的所有数据项做一个标记（marked），然后继续执行`b = 2`，但这时候虽然b在自己的cache里，但由于store buffer里有marked条目，所以，CPU0不会修改cache中的b，而是把它写入Store Buffer；所以CPU0收到Read消息后，会把b的0值发给CPU1，所以继续在`while (b);`自旋。

简而言之，Core执行到write memory barrier（wmb）的时候，如果Store Buffer还有悬而未决的store操作，则都会被mark上，直到被标注的Store操作进入内存后，后续的Store操作才能被执行，因此wmb保障了barrier前后操作的顺序，它不关心barrier前的多个操作的内存序，以及barrier后的多个操作的内存序，是否与Global Memory Order一致。

```
a = 1;
b = 2;
wmb();
c = 3;
d = 4;
```

wmb()保证`a = 1; b = 2;`发生在`c = 3; d = 4;`之前，不保证`a = 1`和`b = 2`的内存序，也不保证`c = 3`和`d = 4`的内部序。

**Invalidate Queue的引入的问题**
就像引入Store Buffer会影响Store的内存一致性，Invalidate Queue的引入会影响Load的内存一致性：

因为Invalidate queue会缓存其他核发过来的消息，比如Invalidate某个数据的消息被delay处置，导致core在Cache Line中命中这个数据，而这个Cache Line本应该被Invalidate消息标记无效。

如何解决这个问题呢？
- 一种思路是硬件确保每次load数据的时候，需要确保Invalidate Queue被清空，这样可以保证load操作的强顺序。
- 软件的思路，就是仿照wmb()的定义，加入rmb()约束。rmb()给我们的invalidate queue加上标记。当一个load操作发生的时候，之前的rmb()所有标记的invalidate命令必须全部执行完成，然后才可以让随后的load发生。这样，我们就在rmb()前后保证了load观察到的顺序等同于global memory order。

所以，我们可以像下面这样修改代码：

```c++
a = 1;
wmb();
b = 2;
```

```c++
while(b != 2) {};
rmb();
assert(a == 1);
```

**系统对内存屏障的支持**
gcc编译器在遇到内嵌汇编语句`asm volatile("" ::: "memory");`将以此作为一条内存屏障，重排序内存操作，即此语句之前的各种编译优化将不会持续到此语句之后。

Linux 内核提供函数 barrier()用于让编译器保证其之前的内存访问先于其之后的完成。
```c
#define barrier() __asm__ __volatile__("" ::: "memory")
```

CPU内存屏障
- 通用barrier，保证读写操作有序， mb()和smp_mb()
- 写操作barrier，仅保证写操作有序，wmb()和smp_wmb()
- 读操作barrier，仅保证读操作有序，rmb()和smp_rmb()

**小结**
为了提高处理器的性能，SMP中引入了store buffer(以及对应实现store buffer forwarding)和invalidate queue。

store buffer的引入导致core上的store顺序可能不匹配于global memory的顺序，对此，我们需要使用wmb()来解决。
invalidate queue的存在导致core上观察到的load顺序可能与global memory order不一致，对此，我们需要使用rmb()来解决。

由于wmb()和rmb()分别只单独作用于store buffer和invalidate queue，因此这两个memory barrier共同保证了store/load的顺序。

#### lock-free
线程同步分为阻塞型同步和非阻塞型同步，互斥量、信号、条件变量这些系统提供的机制都属于阻塞型同步，在争用资源的时候，会导致调用线程阻塞，而非阻塞型同步是在无锁的情况下，通过某种算法和技术手段实现不用阻塞而同步。

Lock-Free主要依靠CPU提供的read-modify-write原语，著名的“比较和交换”CAS（Compare And Swap）在X86机器上是通过cmpxchg系列指令实现的原子操作，通过CAS实现Lock-free的代码通常这样写：

```c
do {
    ...
} while (!CAS(ptr, old_value, new_value));
```

下面是用atomic实现的一个免锁堆栈:

```c++
template <typename T>
struct node {
    T data;
    node* next;
    node(const T& data) : data(data), next(nullptr) {}
};
 
template <typename T>
class stack {
    std::atomic<node<T>*> head;
public:
    void push(const T& data) {
      node<T>* new_node = new node<T>(data);
      new_node->next = head.load(std::memory_order_relaxed);
      while (!head.compare_exchange_weak(new_node->next, new_node,
                                        std::memory_order_release,
                                        std::memory_order_relaxed));
    }
};
```

## 相关概念
### 线程安全与可重入
代码所在的进程有多个线程在同时运行，这些线程可能在同时运行这些代码，如果多线程下运行的结果和单线程运行的结果是一样的，那么线程就是安全的。反之，线程就是不安全的。
不访问共享变量（包括全局变量、static local变量、成员变量），只操作参数，无副作用的函数才是线程安全函数，线程安全函数可多线程重入。

### 锁的粒度
锁的粒度太大，会影响并发，导致性能下降，但锁的粒度太大，会增加代码复杂度，需要平衡。

### 锁的范围
尽量减少锁的范围，不必锁覆盖的代码范围越小越好。

### 线程私有数据
因为全局变量是进程内的所有线程共享的，但有时应用程序设计中必要提供线程私有的全局变量，这个变量仅在线程中有效，但却可以跨过多个函数访问。

比如在程序里可能需要每个线程维护一个链表，而会使用相同的函数来操作这个链表，最简单的方法就是使用同名而不同变量地址的线程相关数据结构。这样的数据结构可以由Posix线程库维护，成为线程私有数据 (Thread-specific Data，或称为 TSD)。

Posix线程私有数据相关接口：
- int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));
- int pthread_key_delete(pthread_key_t key);
- int pthread_setspecific(pthread_key_t key, const void *value);
- void *pthread_getspecific(pthread_key_t key);

而C++等语言，提供threadlocal关键字，在语言层面支持。

### 死锁
死锁常见于两种情况。

#### ABBA锁
假设程序中有2个资源X和Y，分别被锁A和B保护，线程1持有锁A后，想要访问资源Y，而访问资源Y之前需要申请锁B，而如果线程2正持有锁B，并想要访问资源X，为了访问资源X，所以线程2需要申请锁A。

线程1和线程2分别持有锁A和B，并都希望申请对方持有的锁，因为线程申请对方持有的锁，得不到满足，所以便会陷入等待，也就没有机会释放自己持有的锁，对方执行流也就没有办法继续前进，导致相持不下，无限互等，导致死锁。

上述的情况似乎很明显，但如果代码量很大，有时候，这种死锁的逻辑不会这么浅显，它被复杂的调用逻辑所掩盖，但抽茧剥丝，最根本的原因就是上面描述的那样。

我们描述了死锁发生的典型场景，这种情况叫ABBA锁，既某个线程持有A锁申请B锁，而另一个线程持有B锁申请A锁。这种情况可以通过try lock实现，尝试获取锁，如果不成功，则释放自己持有的锁，而不一根筋下去。

#### 自死锁
对于不支持重复加锁的锁，如果线程持有某个锁，而后又再次申请锁，因为该锁已经被自己持有，再次申请锁必然得不到满足，从而导致死锁。

## 这种情况需要加锁吗？

## 扩展知识
### 分布式锁
