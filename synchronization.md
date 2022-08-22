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
原子，意味着不可切分的最小单元，程序中的原子操作指任务不可切分到更小的步骤。

原子性（atomic）是一个可见性的概念：

当我们称一个操作是atomic的，实际上隐含了一个对什么atomic的上下文。

注意：我们说的是从线程视角观察不到完成一半的状态，而并非不存在物理上的进度状态，它取决于你的观察视角。

比如说一个线程中被互斥锁保护的区域，对另一个线程是atomic的，因为从另一个线程视角来看，它没法进入临界区读到数据中间状态，但是对kernel而言却不是atomic的。

从线程视角只能观察到未做和已做两种状态，观察不到完成一半的状态，任务执行不会被中断，也不会穿插进其他操作。

原子性对多线程操作是一个非常重要的属性，因为它不可切分，所以，一个线程没法在另一个线程执行原子操作的时候穿插进去。

比如一个线程原子的写入共享数据，那么其他线程没有办法读到“半修改的数据”；同样，如果一个线程原子读取共享数据，那么它读取的是共享变量在那个瞬间的值，因此原子的读和写没有数据竞争（Data Race）。

原子操作常用于与顺序无关的场景。

#### 原子指令
原子指令指单一的不可再分的不可中断的被硬件直接执行的机器指令，原子指令是无锁编程的基石。

原子指令常被分成两类：
1. store/load
2. read-modify-write(RMW)

Store/Load指令
1. store：存储数据到内存，对应变量写（赋值）
2. load：从内存加载数据，对应变量读

通常，一条简单的store/load机器指令是原子的，比如数据复制指令（mov）可以把内存位置的数据读取到CPU寄存器，相当于Load数据。

x86架构读/写“按数据类型对齐要求对齐的长度不大于机器字长的数据”是原子的。

那什么是数据类型对齐要求呢？

比如在x86_64架构LLP64系统上（LLP64指long、long long和pointer类型是64位的），只要int32类型数据满足放置在起始地址除4为0，int64/long类型数据满足起始地址除8为0，则该数据就是满足类型对齐要求，那么对它的读和写，都是原子的。

一字节的数据读写一定是原子的。

其实，Intel新CPU架构确保读写放置在一个Cache Line的数据（不大于机器字长），跨Cache Line的数据访问无法保证原子性。

C/C++编程中，变量和结构体会自动满足对齐要求，比如：
```c++
int i;

void f() {
    long l;
}

struct Foo {
    int x;
    short s;
    void* ptr;
} foo;
```

全局变量i会被放置在起始地址可以被4整除的内存位置，局部变量y会被放置在起始地址可以被8整除的内存位置，而结构体内的x成员会被放置在起始地址可以被4整除的内存位置。

为了把ptr安置在起始地址可以被8整除的内存位置，编译器会在s后加入填充，从而使得ptr也满足对齐要求。

通过C malloc()接口动态分配的内存，其返回值一般也会对齐到8/16字节，如果有更高的内存对齐要求，可以通过aligned_alloc(alignment, size)接口。C++中的alignas关键字用于设置结构或变量的对齐要求。

对一个满足对齐要求的不大于机器字长的类型变量赋值是原子的，不会出现半完成（即只完成一半字节的赋值），读的情况亦如此。

注意：对长度大于机器字长的数据读写，是不符合原子操作特征的，比如在x86_64系统上，对下面结构体变量的读写都是非原子的:

```c++
struct Foo {
    int a;
    int b;
    int c;
} foo1;

void set_foo1(const Foo& f) {
    foo1 = f;
}
```
foo1包含3个int成员共12字节，大于机器字长8字节，所以对`foo1 = f`不是原子的。

基于以上知识，我们便知道，一些getter/setter接口，即使在多线程环境下，也可以不用加锁，比如：
```c++
struct Foo {

size_t get_x() const { // OK
    return x;
}

void set_y(float y) { // OK
    this->y = y;
}

size_t x;
float y;
};

int main() {
    char buf[8];
    Foo* f = (Foo*)buf;
    f->set(3.14); // dangerous
}
```
但是，如果你把一块buf，强转成Foo，然后调用它的getter/setter，则是危险的，有可能破坏前述的对齐要求。

如果你把一个int变量编码进一个buf，则最好使用memcpy，而不是强转+赋值。

**Read-Modify-Write指令**
但有时候，我们需要更复杂的操作指令，而不仅仅是单独的读或写，它需要把几个动作组合在一起完成某项任务。

比如语句`++count`对应到“读+修改+写”三个操作，但这3个操作不是一个原子操作。所以，多线程程序中使用`++count`，多个执行流会交错执行，会导致计数错误（通常结果比预期数值小）。

考虑另一个情况：读+判断，来我们看一下经典单件实现：
```c++
class Singleton {
    static Singleton* instance;
public:
    static Singleton* get_instance() {
        if (instance == nullptr) {
            instance = new Singleton;
        }
        return instance;
    }
};
```

因为对instance的判断和`instance = new Singleton`不是原子的，所以，我们需要加锁：
```c++
class Singleton {
   static Singleton* instance;
   static std::mutex mutex;
public:
   static Singleton* get_instance() {
       mutex.lock();
       if (instance == nullptr)
           instance = new Singleton;
       mutex.unlock();
       return instance;
   }
};
```

但为了性能，更好的方案是加双检，代码变成下面这样：
```c++

static Singleton* get_instance() {
   if (instance == nullptr) {
       mutex.lock();
       if (instance == nullptr) { // 双检
           instance = new Singleton;
       }
       mutex.unlock();
   }
   return instance;
}
```

第一个检查，如果instance不为空，那么直接返回instance，大多数时候命中这个情况，因为instance一旦被创建，就不再为空。

如果instance为空，那么再加锁、然后第二次检查instance是否为空，为什么要双检呢？因为前面的检查通过后，有可能其他线程创建了实例，导致instance不再为空。

看起来一切都挺好的，高效又缜密。

但双检真的安全吗？这其实是一个非常经典的问题。它有2个风险：

1. 首先，编写者没有告诉编译器，必须假设instance可能被其他线程修改，所以，编译器可能认为2个if保留一个就可以了，当然，它也可能不做这个优化，取决于编译器的策略，因此，instance必须改为volatile，告诉编译器，两次读都必须从内存里加载，避免双检被优化掉。
2. 就是前面讲的原子性，instance指针不能保证一定在8字节对齐的地方保存，所以，需要用std::atomic<Singleton*>代替。

---

逻辑上，需要几个操作是一个密不可分的整体，现代CPU通常都直接提供这类原子指令的支持，这类RMW原子指令通常包括：
- test-and-set(TAS)，把1写入某个内存位置并返回旧值；如果原来内存位置是1，则返回1，否则原子的写入1并返回0；只能标识0和1两种情况
- fetch_and_add，增加某个内存位置的值，并返回旧值；可用来做atomic的后自增
- compare-and-swap(CAS)，比较内存位置的值和指定的值，如果相等，则将新值写入内存位置，如果不等，什么也不做；比tas更强

以上所有操作都是在一个内存位置执行多个动作，但这些操作都是原子单步的，它不会被中断，也不会穿插进其他操作，这个重要属性使得RMW指令非常适合用来实现无锁编程。

虽然CPU在执行机器指令的时候，会把它分成更小粒度的微指令（micro-operations），但程序员应把关注点放在微指令上层的原子指令上。

#### 原子操作的不同层次
前面讲的原子指令是硬件层面，不同架构甚至不同型号CPU有不同的原子指令，它是CPU层面的东西，跨平台特性差，用它编写的代码不可移植，所以应该尽量避免直接使用原子指令。

回到软件层面，软件层面的原子操作包括三个层次：
- 操作系统层面，linux操作系统提供atomic这种原子类型，配合相关的编程接口使用，大多数它只是对原子指令的简单封装，但它屏蔽了硬件差异，比原子指令更易用
    ```c
    atomic_read(atomic_t *v)
    atomic_set(atomic_t *v, int i)
    atomic_inc(atomic_t *v)
    atomic_dec(atomic_t *v)
    atomic_add(int i, atomic_t *v)
    atomic_sub(int i, atomic_t *v)
    atomic_inc_and_test(atomic_t *v)
    atomic_dec_and_test(atomic_t *v);
    atomic_sub_and_test(int i, atomic_t *v)
    ```
- 编译器层面，gcc提供原子操作build-in函数，使用gcc编译c/c++代码，可以直接使用它们
    ```c
    //其中type对应8/16/32/64位整数
    type __sync_fetch_and_add (type *ptr, type value, ...)
    type __sync_fetch_and_sub (type *ptr, type value, ...)
    type __sync_fetch_and_or (type *ptr, type value, ...)
    type __sync_fetch_and_and (type *ptr, type value, ...)
    type __sync_fetch_and_xor (type *ptr, type value, ...)
    type __sync_fetch_and_nand (type *ptr, type value, ...)
    type __sync_add_and_fetch (type *ptr, type value, ...)
    type __sync_sub_and_fetch (type *ptr, type value, ...)
    type __sync_or_and_fetch (type *ptr, type value, ...)
    type __sync_and_and_fetch (type *ptr, type value, ...)
    type __sync_xor_and_fetch (type *ptr, type value, ...)
    type __sync_nand_and_fetch (type *ptr, type value, ...)
    ```
    
    gcc在实现C++11之后，新的原子接口，以__atomic为前缀，推荐使用下面这些接口：
    ```c
    type __atomic_add_fetch(type *ptr, type val, int memorder)
    type __atomic_sub_fetch(type *ptr, type val, int memorder)
    type __atomic_and_fetch(type *ptr, type val, int memorder)
    type __atomic_xor_fetch(type *ptr, type val, int memorder)
    type __atomic_or_fetch(type *ptr, type val, int memorder)
    type __atomic_nand_fetch(type *ptr, type val, int memorder)
    type __atomic_fetch_add(type *ptr, type val, int memorder)
    type __atomic_fetch_sub(type *ptr, type val, int memorder)
    type __atomic_fetch_and(type *ptr, type val, int memorder)
    type __atomic_fetch_xor(type *ptr, type val, int memorder)
    type __atomic_fetch_or(type *ptr, type val, int memorder)
    type __atomic_fetch_nand(type *ptr, type val, int memorder)
    ```
- 编程语言层面，也通常提供原子操作类型和接口，这也是使用原子操作的推荐方式，它有良好的跨平台性和可移植性，程序员应优先使用它们：
    - C11新增了原子操作库，通过stdatomic.h头文件提供atomic_fetch_add/atomic_compare_exchange_weak等接口。
    - C++11也新增了原子操作库，提供一种类型为std::atomic<T>的类模板，它提供++/--/+=/-=/fetch_sub/fetch_add等原子操作接口。

原子操作常用于与顺序无关的场景，比如计数错误的场景，用原子变量改写后，则会输出符合预期的值。

原子操作是编写Lock-free多线程程序的基础，原子操作只保证原子性，不保证操作顺序。

在Lock-free多线程程序中，光有原子操作是不够的，需要将原子操作和Memory Barrier结合起来，才能实现免锁。

### 锁同步的问题
锁是操作系统提供的一种同步原语，通过在访问共享资源前加锁，结束访问共享资源后解锁，让任何时刻只有一个线程访问共享，本质是做串行化。

程序对共享资源的访问任务，一般包括三步骤，读原值，修改值，将新值写回，用锁同步的话，就是在确保这3个步骤，不会被打断，访问共享资源的临近代码区只有一个线程在同时运行，第一个获得锁的线程能往前推进，其他线程只能等着直到持有锁的线程释放锁。

线程同步分为阻塞型同步和非阻塞型同步。

互斥量、信号、条件变量这些系统提供的机制都属于阻塞型同步，在争用资源的时候，会导致调用线程阻塞。

而非阻塞型同步是在无锁的情况下，通过某种算法和技术手段实现不用阻塞而同步。

锁是阻塞（Blocking）同步机制，阻塞同步机制的缺陷是可能挂起你的程序，如果持有锁的线程crash，则锁永远得不到释放，而其他线程则将陷入无限等待，另外，它也可能导致优先级倒转等问题。

所以，我们需要非阻塞（Non-Blocking）的同步机制。

### 什么是lock-free？
lock-free没有锁同步的问题，所有线程无阻碍的执行原子指令，而不是等待。

比如一个线程读atomic类型变量，一个线程写atomic变量，它们没有任何等待，硬件原子指令确保不会出现数据不一致，写入数据不会出现半完成，读取数据也不会读一半。

那到底什么是lock-free？

有人说lock-free就是不使用mutex / semaphores之类的无锁（lock-Less）编程，这句话严格来说并不对。

我们先看一下wiki对Lock-free的描述:
> Lock-freedom allows individual threads to starve but guarantees system-wide throughput. An algorithm is lock-free if, when the program threads are run for a sufficiently long time, at least one of the threads makes progress (for some sensible definition of progress). All wait-free algorithms are lock-free.

> In particular, if one thread is suspended, then a lock-free algorithm guarantees that the remaining threads can still make progress. Hence, if two threads can contend for the same mutex lock or spinlock, then the algorithm is not lock-free. (If we suspend one thread that holds the lock, then the second thread will block.)

翻译一下：
- 第一段：lock-free允许单个线程饥饿但保证系统级吞吐。如果一个程序的线程执行足够长的时间，那么至少一个线程会往前推进，那么这个算法就是lock-free的。​​​
- 第二段：尤其是，如果一个线程被暂停，lock-free算法保证其他线程依然能够往前推进。

第一段是给lock-free下定义，第二段是对lock-free做解释。

因此，如果2个线程竞争同一个互斥锁或者自旋锁，那它就不是lock-free的。

因为如果我们暂停持有锁的线程，那么另一个线程会被阻塞。

wiki的这段描述很抽象，它不够直观，稍微再解释一下：

lock-free描述的是代码逻辑的属性，不使用锁的代码，大部分具有这种属性。所以，我们经常会混淆这lock-free和无锁这2个概念。

其实，lock-free是对代码（算法）性质的描述，是属性，而无锁是说代码如何实现，是手段。

lock-free的关键描述是：如果一个线程被暂停，那么其他线程应能继续前进，它需要有系统级（system-wide）的吞吐。

我们从反面举例来看，假设我们要借助锁实现一个无锁队列，我们可以直接使用线程不安全的std::queue + std::mutex来做：
```c++
template <typename T>
class Queue {
public:
    void push(const T& t) {
        q_mutex.lock();
        q.push(t);
        q_mutex.unlock();
    }
private:
    std::queue<T> q;
    std::mutex q_mutex;
};
```
如果有线程A/B/C同时执行push方法，最先进入的线程A获得互斥锁。

线程B和C因为获取不到互斥锁而陷入等待。

这个时候，线程A如果因为某个原因（如出现异常，或者等待某个资源）而被永久挂起，那么同样执行push的线程B/C将被永久挂起，系统整体（system-wide）没法推进。这显然不符合lock-free的要求。

因此：所有基于锁（包括spinlock）的并发实现，都不是lock-free的。

因为它们都会遇到同样的问题：即如果永久暂停当前占有锁的线程/进程的执行，将会阻塞其他线程/进程的执行。

而对照lock-free的描述，它允许部分process（理解为执行流）饿死但必须保证整体逻辑的持续前进，基于锁的并发显然是违背lock-free要求的。

#### CAS loop实现lock-free
Lock-Free同步主要依靠CPU提供的read-modify-write原语，著名的“比较和交换”CAS（Compare And Swap）在X86机器上是通过cmpxchg系列指令实现的原子操作，CAS逻辑上用代码表达是这样的：
```c++
bool CAS(T* ptr, T expect_value, T new_value) {
   if (*ptr != expect_value) {
      return false;
   }
   *ptr = new_value;
   return true;
}
```
CAS接受3个参数：
- 内存地址
- 期望值，通常传第一个参数所指内存地址中的旧值
- 新值
逻辑描述：CAS比较内存地址中的值和期望值，如果不相同就返回失败，如果相同就将新值写入内存并返回成功。

当然这个C函数描述的只是CAS的逻辑，这个函数操作不是原子的，因为它可以划分成几个步骤：读取内存值、判断、写入新值，各步骤之间是可以插入其他操作的。

不过前面讲了，原子指令相当于把这些步骤打包，它可能是通过`lock; cmpxchg`实现的，但那是实现细节，程序员更应该注重在逻辑上理解它的行为。

通过CAS实现Lock-free的代码通常借助循环，代码如下：
```c
do {
    T expect_value = *ptr;
} while (!CAS(ptr, expect_value, new_value));
```
1. 创建共享数据的本地副本，expect_value
2. 根据需要修改本地副本，从ptr指向的共享数据里load后赋值给expect_value
3. 检查共享的数据跟本地副本是否相等，如果相等，则把新值复制到共享数据

第三步是关键，虽然CAS是原子的，但加载expect_value跟CAS这2个步骤，并不是原子的。

所以，我们需要借助循环，如果ptr内存位置的值没有变（*ptr == expect_value），那就存入新值返回成功；否则说明加载expect_value后，ptr指向的内存位置被其他线程修改了，这时候就返回失败，重新加载expect_value，重试，直到成功为止。

CAS loop支持多线程并发写，这个特点太有用了，因为多线程同步，很多时候都面临多写的问题，我们可以基于cas实现Fetch-and-add(FAA)算法，它看起来像这样：
```c++
T faa(T& t) {
    T temp = t;
    while (!compare_and_swap(x, temp, temp + 1));
}
```
第一步加载共享数据的值到temp，第二步比较+存入新值，直到成功。

**lock-free栈**
下面是C++ atomic compare_exchange_weak()实现的一个lock-free堆栈（来自CppReference）
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
代码解析：
- 节点（node）保存T类型的数据data，并且持有指向下一个节点的指针
- `std::atomic<node<T>*>`类型表明atomic里放置的是Node的指针，而非Node本身，因为指针在64位系统上是8字节，等于机器字长，再长没法保证原子性
- stack类包含head成员，head是一个指向头结点的指针，头结点指针相当于堆顶指针，刚开始没有节点，head为NULL
- push函数里，先根据data值创建新节点，然后要把它放到堆顶
- 因为是用链表实现的栈，所以，如果新节点要成为新的堆顶（相当于新节点作为新的头结点插入），那么新节点的next域要指向原来的头结点，并让head指向新节点
- `new_node->next = head.load`把新节点的next域指向原头结点，然后`head.compare_exchange_weak(new_node->next, new_node)`，让head指向新节点
- C++ atomic的compare_exchange_weak()跟上述的CAS稍有不同，head.load()不等于new_node->next的时候，它会把head.load()的值重新加载到new_node->next
- 所以，在加载head值和cas之间，如果其他线程调用push操作，改变了head的值，那没有关系，该线程的本次cas失败，下次重试便可以了
- 多个线程同时push时，任一线程在任意步骤阻塞/挂起，其他线程都会继续执行并最终返回，无非就是多执行几次while循环

这样的行为逻辑显然符合lock-free的定义，注意用cas+loop实现自旋锁不符合lock-free的定义，注意区分。

***ABA问题**
CAS经常伴随ABA问题，考虑前面的CAS逻辑，CAS里会做判等，判等的2个操作数，一个是前步加载的内存地址的值，另一个是再次从内存地址读取的值，如果2个操作数相等，它便认为内存位置的值没变。

但实际上，如果“两次读取一个内存位置的值相同，则说明该内存位置没变”，这个假设不一定成立，为什么呢？

假设线程1在前面一步从内存位置加载数据后。

线程2切入，将该内存位置的数据从A修改为B，然后又修改为A。

等到线程1继续执行CAS的时候，它观测到的内存位置值未变（依然是A），因为线程2将数据从A修改到B，再修改成A。

线程1两次读到了相同的值，它便断定内存位置没有被修改，实际并非如此，线程2改了内存位置。

基于上述逻辑行事，就有可能出现未定义的结果。

这就是经典的ABA问题，会不会出问题，完全取决于你的程序逻辑，有些逻辑可以容忍ABA问题，而有些不可以。

ABA问题很多时候由内存复用引起，比如一块内存被回收后又被分配，地址值不变，但这个对象已经不是之前的那个对象了，有很多解决ABA问题的技术手段，比如增加版本号等等，目的无非是去识别这种变化。

### wait-free
wiki关于wait-free的词条解释：
> Wait-freedom is the strongest non-blocking guarantee of progress, combining guaranteed system-wide throughput with starvation-freedom. An algorithm is wait-free if every operation has a bound on the number of steps the algorithm will take before the operation completes.[15] This property is critical for real-time systems and is always nice to have as long as the performance cost is not too high.

翻译过来就是：wait-free有着最强的非阻塞进度保证，wait-free有系统级吞吐兼具无饥饿特征。

如果一个算法的每个操作完成都只有有限操作步数，那么这个算法就是wait-free的。

这个特征对于实时系统非常关键，且它的性能成本不会太高。

wait-free比lock-free更严格，所有wait-free算法都是lock-free的，反之不成立，wait-free是lock-free的子集。

wait-free的关键特征是所有步骤都在有限步骤完成，所以前面的`cas + loop`实现的lock-free栈就不是wait-free的。

因为理论上，如果调用push的线程是个倒霉蛋，一直有其他线程push且恰好又都成功了，则这个倒霉蛋线程的cas会一直失败，一直循环下去，这与wait-free每个操作都必须在有限步骤完成的要求相冲突。wait-free的尝试次数通常跟线程数有关，线程数越多，则这个有限的上限就越大。

但前面讲的atomic fetch_add则是wait-free的，因为它不会失败。

简单点说，lock-free可以有循环，而wait-free连循环都不应该有。

wait-free非常难做，以前一直看不到wait-free的数据结构和算法实现，直到2011才有人搞出了一个wait-free队列，虽然这个队列也用到了cas，但是它为每一步发送的操作提供了一个state array，为每个操作赋予一个number，从而保证有限步完成入队出队操作，论文链接：(wait-free queue)[http://www.cs.technion.ac.il/~erez/Papers/wfquque-ppopp.pdf]

## Obstruction-free
Obstruction-free提供比lock-free更弱的一个非阻塞进度保证，所有lock-free都属于Obstruction-free。

Obstruction-free翻译过来叫无障碍，是指在任何时间点，一个孤立运行线程的每一个操作可以在有限步之内结束。只要没有竞争，线程就可以持续运行，一旦共享数据被修改，Obstruction-free要求中止已经完成的部分操作，并进行回滚，obstruction-free是并发级别更低的非阻塞并发，该算法在不出现冲突性操作的情况下提供单线程式的执行进度保证。

obstruction-freedom要求可以中止任何部分完成的操作并且能够回滚已做的更改，为了内容完备性，把obstruction-free列在这里。

### 无锁数据结构
一些无锁（阻塞）数据结构不需要特殊原子操作原语即可实现，例如：
- 支持单线程读和单线程写的Ring Buffer先进先出队列，通过把容量设置为2^n，利用整型回绕特点+内存屏障就可以实现，参考linux内核的kfifo
- 带单写线程和任意数读线程的读拷贝更新（RCU），通常，读无等待（wait-free），写无锁（lock-free）
- 带多写线程和任意数读线程的读拷贝更新（RCU），通常，读无等待（wait-free），写用锁做串行化

无锁数据结构实现起来比较困难，非必要不要自己去实现。

### 自旋锁
在cpu核上自旋，粘着cpu不停的测试，直到其他cpu解锁，这是一种消耗cpu的加锁方式，适合持有锁的时间非常短的情况，它基于一种假设：睡眠导致的调度开销大于在cpu上自旋测试的开销。
- 自旋锁也叫优雅粒度的锁，跟操作系统提供的阻塞机制的粗粒度锁（mutex和信号量）对应。
- 自旋锁一般不应该被长时间持有，如果持有锁的时间可能比较长，那就用操作系统为你提供的粗粒度的锁就好了。
- 线程持有自旋锁的时候，线程一般不应该被调度走，内核层可以禁中断禁调度保障这一点，但应用层程序一般无法避免线程被调度走，所以应用层使用自旋锁其实得不到保障。
- 自旋锁不可递归。
- 自旋锁常用来追求低延时，而不是为了提升系统吞吐，因为自旋锁，不会把执行线程调度走，不会阻塞（睡眠）。

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
乱序执行会引起内存顺序跟程序顺序不同，乱序执行的原因是多方面的，比如编译器指令重排、超标量指令流水线、预测执行、Cache-Miss等。内存操作顺序无法精确匹配程序顺序，这有可能带来混乱，既然有副作用，那为什么还需要乱序执行呢？

答案是为了性能。

我们先看看没有乱序执行之前，早期的有序处理器（In-order Processors）是怎么处理指令的？
- 指令获取，从代码节内存区域加载指令到I-Cache
- 译码
- 如果指令操作数可用（例如操作数位于寄存器中），则分发指令到对应功能模块中；如果操作数不可用，通常是需要从内存加载，则处理器会stall，一直等到它们就绪，直到数据被加载到cache或拷贝进寄存器
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

内存屏障，其实就是提供一种机制，确保代码里顺序写下的多行，会按照书写的顺序，被存入内存，主要是解决StoreBuffer引入导致的写入内存间隙的问题。

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

## 伪共享（false sharing）
```c++
const size_t shm_size = 16*1024*1024; //16M
static char shm[shm_size];
std::atomic<size_t> shm_offset{0};

void f() {
    for (;;) {
        auto off = shm_offset.fetch_add(sizeof(long));
        if (off >= shm_size) break;
        *(long*)(shm + off) = off;
    }
}
```
考察上面的程序，shm是一块16M字节的内存，我测试的机器的L3 Cache是32M，所以挑选16M这个值确保shm数组在Cache里能存放得下。

f()函数在循环里，把shm视为long类型的数组，依次给每个元素赋值，shm_offset用于记录偏移位置，shm_offset.fetch_add(sizeof(long))原子性的增加shm_offset的值（因为x86_64系统上long的长度为8，所以shm_offset每次增加8字节），并返回增加前的值，对shm上long数组的每个元素赋值后，结束循环从函数返回。

因为shm_offset是atomic类型变量，所以多线程调用f()依然能正常工作，虽然多个线程会竞争shm_offset，但每个线程会排他性的对各long元素赋值，多线程并行会加快对shm的赋值操作。

我们加上多线程调用代码，代码如下：
```c++
std::atomic<size_t> step{0};

const int THREAD_NUM = 2;

void work_thread() {
    const int N = 10;
    for (int n = 1; n <= N; ++n) {
        f();
        ++step;
        while (step.load() < n * THREAD_NUM) {}
        shm_offset = 0;
    }
}

int main() {
    std::thread threads[THREAD_NUM];
    for (int i = 0; i < THREAD_NUM; ++i) {
        threads[i] = std::move(std::thread(work_thread));
    }
    for (int i = 0; i < THREAD_NUM; ++i) {
        threads[i].join();
    }
    return 0;
}
```
- main函数里启动2个工作线程work_thread
- 工作线程对shm共计赋值N（10）轮，后面的每一轮会访问Cache里的shm数据，step用于work_thread之间每一轮的同步
- 工作线程调用完f()后会增加step，等2个工作线程都调用完之后，step的值增加到n * THREAD_NUM后，while()会结束循环，重置shm_offset，重新开始新一轮对shm的赋值

编译后执行上面的程序，产生如下的结果：
```
time ./a.out

real 0m3.406s
user 0m6.740s
sys 0m0.040s
```
time命令用于时间测量，在a.out程序运行完成，会打印耗时，real列显式耗时3.4秒。

### 改进版f_fast
我们稍微修改一下f函数，改进版f函数取名f_fast：
```c++
void f_fast() {
    for (;;) {
        const long inner_loop = 16;
        auto off = shm_offset.fetch_add(sizeof(long) * inner_loop);
        for (long j = 0; j < inner_loop; ++j) {
            if (off >= shm_size) return;
            *(long*)(shm + off) = j;
            off += sizeof(long);
        }
    }
}
```
for循环里，shm_offset不再是每次增加8字节（sizeof(long)），而是8*16=128字节，然后在内层的循环里，依次对16个long连续元素赋值，然后下一轮循环又再次增加128字节，直到完成对shm的赋值。

编译后重新执行程序，结果显示耗时降低到0.06秒，对比前一种耗时3.4秒，f_fast性能提升。
```
time ./a.out

real 0m0.062s
user 0m0.110s
sys 0m0.012s
```

### f和f_fast的行为差异
shm数组总共有2M个long元素，因为16M / sizeof(long) => 2M
1. f()函数行为逻辑
    - 线程1和线程2的work_thread里会交错地对shm元素赋值，shm的2M个long元素，会顺序的一个接一个的派给2个线程去赋值。
    - 可能元素1有线程1赋值，元素2由线程2赋值，然后元素3和元素4由线程1赋值，然后元素5又由线程2赋值... 
    - 每次派元素的时候，shm_offset都会atomic的增加8字节，所以不会出现2个线程给1个元素赋值的情况

2. f_fast()函数行为逻辑
    - 每次派元素的时候，shm_offset原子性的增加128字节（16个元素）
    - 这16个字节作为一个整体，派给线程1或者线程2；虽然线程1和线程2还是会交错的操作shm元素，但是以16个元素（128字节）为单元，这16个连续的元素不会被分开派发
    - 一次派发的16个元素，会在内部循环里被一个接着一个的赋值，在一个线程里被执行

### 为什么f_fast更快？
第一眼感觉是f_fast()里shm_offset.fetch_add()调用频次降低到了原来的1/16，我们有理由怀疑是原子变量的竞争减少导致程序执行速度加快。

为了验证，让我们在内层的循环里加一个原子变量test的fetch_add，test原子变量的竞争会像f()函数里shm_offset.fetch_add()一样被激烈，修改后的f_fast代码变成下面这样：
```c++
void f_fast() {
    for (;;) {
        const long inner_loop = 16;
        auto off = shm_offset.fetch_add(sizeof(long) * inner_loop);
        for (long j = 0; j < inner_loop; ++j) {
            test.fetch_add(1);
            if (off >= shm_size) return;
            *(long*)(shm + off) = j;
            off += sizeof(long);
        }
    }
}
```
为了避免test.fetch_add(1)的调用被编译器优化掉，我们在main函数的最后把test的值打印出来。

编译后测试一下，结果显示：执行时间只是稍微增加到`real 0m0.326s`。所以，很显然，并不是atomic的调用频次减少导致性能飙升。

我们重新审视f()循环里的逻辑：f()循环里的操作很简单：原子增加、判断、赋值。

我们把f()的里赋值注释掉，再测试一下，发现它的速度得到了很大提升，看来是`*(long*)(shm + off) = off;`这一行代码执行慢，但这明明只是一行赋值。

我们把它反汇编来看，它只是一个mov指令，源操作数是寄存器，目标操作数是内存地址，从寄存器拷贝数据到一个内存地址，为什么会这么慢呢？

### 答案
现在揭晓原因，导致f()性能底下的元凶是伪共享（false sharing），那什么是伪共享？

要说清这个问题，还得联系CPU的架构，以及CPU怎么访问数据，我们回顾一下关于多核Cache结构：

**背景知识**
我们知道现代CPU可以有多个核，每个核有自己的L1-L2缓存，L1又区分数据缓存（L1-DCache）和指令缓存（L1-ICache），L2不区分数据和指令Cache，而L3是跨核共享的，L3通过内存总线连接到内存，内存被所有CPU所有Core共享。

CPU访问L1 Cache的速度大约是访问内存的100倍，Cache作为CPU与内存之间的缓存，减少对内存的访问频率。

从内存加载数据到Cache的时候，是以Cache Line为长度单位的，Cache Line的长度通常是64字节，所以，那怕你只读一个字节，但是包含该字节的整个Cache Line都会被加载到缓存，同样，如果你修改一个字节，那么最终也会导致整个Cache Line被冲刷到内存。

如果一块内存数据被多个线程访问，假设多个线程在多个Core上并行执行，那么它便会被加载到多个Core的的Local Cache中；这些线程在哪个Core上运行，就会被加载到哪个Core的Local Cache中，所以，内存中的一个数据，在不同Core的Cache里会同时存在多份拷贝。

如果我们修改了Core1缓存里的某个数据，则该数据所在的Cache Line的状态需要同步给其他Core的缓存，Core之间可以通过核间消息同步状态，比如通过发送Invalidate消息给其他核，接收到该消息的核会把对应Cache Line置为无效，然后重新从内存里加载最新数据。

当然，被加载到多个Core缓存中的同一Cache Line，会被标记为共享（Shared）状态，对共享状态的缓存行进行修改，需要先获取缓存行的修改权（独占），MESI协议用来保证多核缓存的一致性，更多的细节可以参考MESI的文章。

**示例分析**
假设线程1运行在Core1，线程2运行在Core2。

- 因为shm被线程1和线程2这两个线程并发访问，所以shm的内存数据会以Cache Line粒度，被同时加载到2个Core的Cache，因为被多核共享，所以该Cache Line被标注为Shared状态。
- 假设线程1在offset为64的位置写入了一个8字节的数据（sizeof(long)），要修改一个状态为Shared的Cache Line，Core1会发送核间通信消息到Core2，去拿到该Cache Line的独占权，在这之后，Core1才能修改Local Cache。
- 线程1执行完`shm_offset.fetch_add(sizeof(long))`后，shm_offset会增加到72。
- 这时候Core2上运行的线程2也会执行`shm_offset.fetch_add(sizeof(long))`，它返回72并将shm_offset增加到80。
- 线程2接下来要修改shm[72]的内存位置，因为shm[64]和shm[72]在一个Cache Line，而这个Cache Line又被置为Invalidate，所以，它需要从内存里重新加载这一个Cache Line，而在这之前，Core1上的线程1需要把Cache Line冲刷到内存，这样线程2才能加载最新的数据。

这种交替执行模式，相当于Core1和Core2之间需要频繁的发送核间消息，收到消息的Core的Cache Line被置为无效，并重新从内存里加载数据到Cache，每次修改后都需要把Cache中的数据刷入内存，这相当于废弃掉了Cache，因为每次读写都直接跟内存打交道，Cache的作用不复存在，这就是性能低下的原因。

这种多核多线程程序，因为并发读写同一个Cache Line的数据（临近位置的内存数据），导致Cache Line的频繁失效，内存的频繁Load/Store，从而导致性能急剧下降的现象叫伪共享，伪共享是性能杀手。

### 另一个伪共享的例子
假设线程x和y，分别修改Data的a和b变量，如果被频繁调用，也会出现性能低下的情况，怎么规避呢？
```c++
struct Data {
    int a;
    int b;
};

Data data; // global

void thread1() {
    data.a = 1;
}

void thread2() {
    data.b = 2;
}
```

**空间换时间**
避免Cache伪共享导致性能下降的思路是用空间换时间，通过增加填充，让a和b两个变量分布到不同的Cache Line，这样对a和b的修改就会作用于不同Cache Line，就能避免Cache失效的问题。
```c++
struct Data {
    int a;
    int padding[60];
    int b;
};
```

在Linux kernel中存在__cacheline_aligned_in_smp宏定义用于解决false sharing问题。

```c
#ifdef CONFIG_SMP
#define __cacheline_aligned_in_smp __cacheline_aligned
#else
#define __cacheline_aligned_in_smp
#endif

struct Data {
    int a;
    int b __cacheline_aligned_in_smp;
};
```
从上面的宏定义，我们可以看到：
- 在多核（MP）系统里，该宏定义是 __cacheline_aligned，也就是Cache Line的大小
- 在单核系统里，该宏定义是空的

### 伪共享的疑问
既然多CPU多核并发读写一个Cache Line里的内存数据，会出现伪共享，那么，我们对`atomic<size_t> shm_offset`的fetch_add()操作也满足这个条件，多个线程并发读写一个原子变量，为什么性能不会很差呢？

我们反汇编发现`atomic.fetch_add`会被翻译成`lock; xadd %rax (%rdx)`，lock是一个指令前缀，配合其他指令使用。

执行lock指令，Intel CPU会根据情况自行决定到底是assert LOCK# signal（锁总线），还是锁缓存。

对于早期的CPU（P6前），lock就是锁总线（bus lock），某个核心遇到lock指令，就触发总线的“LOCK#”那跟线，仲裁器干活，选择一个核心独占总线，其他核心不能再通过总线与内存通讯（不能访问任何内存数据），从而达到原子性的访问内存的目的。

锁总线的操作比较重，相当于全局的内存总线锁，lock前缀之后的指令操作直接作用于内存，bypass掉缓存，lock也相当于内存屏障。

Intel P6 CPU开始，优化了lock指令，通过ringbus + MESI协议做了缓存锁，如果访问的内存区域已经缓存在处理器的缓存行中，则不会assert LOCK#信号，它会对CPU的缓存中的缓存行进行锁定，在锁定期间，其它CPU不能同时缓存此数据，在修改之后，通过缓存一致性协议来保证修改的原子性，这个操作被称为“缓存锁”。

false sharing对应的是多线程同时读写一个Cache Line的多个数据，Core-A修改数据x后，会置Cache Line为Invalid，Core-B读该缓存行的另一个数据y，需要Core-A把Cache Line Store到内存，Core-B再从内存里Load对应Cache Line，数据要过内存。

而atomic，多个线程修改的是同一个变量，对应汇编会带lock指令前缀，而这个lock前缀，会提醒CPU优先采用缓存锁方式处理。

所有核心通过RingBus连接成一个环，如果一份内存数据被多个Core加载到Cache，则状态为Shared（S）；一旦一个核心修改了某个Cache Line数据，则状态变成Modify（M），其他核心就能通过RingBus迅速感知到这个修改，从而把自己的Cache Line置为Invalid（I），并且从标记为M的Cache中把数据读过来，完成数据不过内存的核间传播。

因为atomic在Cache Line里的最新值通过RingBus传递给其他核心，不需要频繁的内存Store/Load，所以性能不会那么糟。

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
