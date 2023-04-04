# 什么是多线程同步？
同一进程内的多个线程会共享数据，对共享数据的并发访问会出现`race condition`，这个词的官方翻译是竞争条件，但`condition`翻译成条件令人困惑，特别是对初学者而言，不够清晰明了，翻译软件显示`condition`有状况、状态的含义，可能翻译成竞争状况更直白。

多线程同步是指：
- 协调多个线程对共享数据的访问，避免出现数据不一致的情况。
- 协调各个事件的发生顺序，使多线程在某个点交汇并按预期步骤往前推进，比如某线程需要等另一个线程完成某项工作才能开展该线程的下一步工作。

要掌握多线程同步，需先理解为什么需要多线程同步、哪些情况需要同步。

# 为什么需要同步?
理解为什么要同步（Why）是多线程编程的关键，它甚至比掌握多线程同步机制（How）本身更加重要。

识别什么地方需要同步是编写多线程程序的难点，只有准确识别需要保护的数据、需要同步的点，再配合系统或语言提供的合适的同步机制，才能编写安全高效的多线程程序。

下面通过几个例子解释为什么需要同步。

## 示例1
假设有1个长度为256的字符数组`msg`用于保存消息，函数`read_msg()`和`write_msg()`分别用于`msg`的读和写：
```c++
// example 1
char msg[256] = "this is old msg";

char* read_msg() {
    return msg;
}

void write_msg(char new_msg[], size_t len) {
    memcpy(msg, new_msg, std::min(len, sizeof(msg)));
}

void thread1() {
    char new_msg[256] = "this is new msg, it's too looooooong";
    write_msg(new_msg, sizeof(new_msg));
}

void thread2() {
    printf("msg=%s\n", read_msg());
}
```
如果线程1调用`write_msg()`，线程2调用`read_msg()`，不加保护。

因为`msg`的长度是256字节，完成长达256字节的内存写入需要多个内存周期，在线程1写入新消息期间，线程2可能读到不一致的数据。即可能读到"this is new msg"，而后半段"it's very..."线程1还没来得及写入，它不是完整的新消息。

在这个例子中，不一致表现为数据不完整。

## 示例2
比如对于二叉搜索树的节点，一个结构体有3个成分：
- 一个指向父节点的指针
- 一个指向左子树的指针
- 一个指向右子树的指针
```c++
// example 2
struct Node {
    struct Node *parent;
    struct Node *left_child, *right_child;
};
```
这3个成分是有关联的，将节点加入BST，要设置这3个指针域，从BST删除该节点，要修改该节点的父、左孩子节点、右孩子节点的指针域。

对多个指针域的修改，不能在一个指令周期完成，如果完成了一个成分的写入，还没来得其他成分修改，就有可能被其他线程读到了，但此时节点的有些指针域还没有设置好，通过指针域去取数则可能出错。

## 示例3
考虑两个线程对同一个整型变量做自增，变量的初始值是0，我们预期2个线程完成自增后变量的值为2。
```c++
// example 3
int x = 0; // 初始值为0
void thread1() { ++x; }
void thread2() { ++x; }
```
简单的自增操作，包括三步：
- 加载: 从内存中读取变量x的值存放到寄存器。
- 更新: 在寄存器里完成自增。
- 保存: 把位于寄存器中的新值写入内存。

两个线程并发执行`++x`，让我们看看真实情况是什么样的？
1. 如果2个线程，先后执行自增，在时间上完成错开。无论是1先2后，或是2先1后，那么`x`的最终值是2，符合预期。但多线程并发并不能确保对一个变量的访问在时间上完全错开。
2. 如果时间上没有完全错开，假设线程1在core1上执行，线程2在core2上执行，那么，一个可能的执行过程：
    - 首先，线程1把`x`读到core1的寄存器，线程2也把`x`的值加载到core2的寄存器，此时，存放在两个core的寄存器中`x`的副本都是0。
    - 然后，线程1完成自增，更新寄存器里`x`的值的副本（0变1），线程2也完成自增，更新寄存器里`x`的值的副本（0变1）。
    - 然后，线程1将更新后的新值1写入变量`x`的内存位置。
    - 最后，线程2将更新后的新值1写入同一内存位置，变量`x`的最终值是1，不符合预期。

线程1和线程2在同一个core上交错执行，也有可能出现同样的问题，这个问题跟硬件结构无关。

之所以会出现不符合预期的情况，主要是因为“加载+更新+保存”这3个步骤不能在一个内存周期内完成。

多个线程对同一变量并发读写，不加同步的话会出现数据不一致，对应到该示例，不一致表现为终值既可能为1也可能为2。

## 示例4
用C++类模板实现一个队列:
```c++
// example 4
template <typename T>
class Queue {
    static const unsigned int Q_CAPACITY = 100;
    T elements[Q_CAPACITY];
    int element_num = 0, head = 0, tail = -1;
public:
    // 入队
    bool push(const T& element) {
        if (element_num == Q_CAPACITY) return false;
        ++element_num;
        tail = (++tail) % Q_CAPACITY;
        elements[tail] = element;
        return true;
    }
    // 出队
    void pop() {
        assert(!empty());
        head = (++head) % Q_CAPACITY;
        --element_num;
    }
    // 判空
    bool empty() const { 
        return element_num == 0; 
    }
    // 访队首
    const T& front() const {
        assert(!empty());
        return elements[head];
    }
};
```
代码解释：
- `T elements[]`保存数据；2个游标，分别用于记录队首`head`和队尾`tail`的位置（下标）
- `push()`接口，先移动`tail`游标，再把元素添加到队尾
- `pop()`接口，移动`head`游标，弹出队首元素（逻辑上弹出）
- `front()`接口，访问队首
- `front()、pop()`先做断言，调用`pop()/front()`的客户代码需确保队列非空

假设现在有一个`Queue<int>`实例q，因为不能直接调用`pop`，我们封装一个`try_pop()`，代码大概写这样：
```c++
Queue<int> q;

void try_pop() {
    if (!q.empty()) {
        // printf("%d\n", q.front());
        q.pop();
    }
}
```
如果多个线程调用`try_pop()`，会有问题，为什么？

原因：`判空+出队`这2个操作，不能在一个指令周期内完成。如果线程1在判空为伪之后，线程2穿插进来，判空也伪，这样就有可能2个线程竞争弹出唯一的元素。

多线程环境下，读变量然后基于变量值做下一步操作，这样的逻辑如果不加保护就会出错，这是由数据使用方式引入的问题。

## 示例5
再看一个简单的，简单的对int32_t多线程读写。
```c++
// example 5
int32_t data[8] = {1,2,3,4,5,6,7,8}; 

struct Foo {
    int32_t get() const { return x; }
    void set(int32_t x) { this->x = x; }
    int32_t x;
} foo;

void print_int(int32_t value) {
    // 正确的打印value的值
}

void thread_write1() {
    for (;;) { for (auto v : data) { foo.set(v); } }
}

void thread_write2() {
    for (;;) { for (auto v : data) { foo.set(v); } }
}

void thread_read() {
    for (;;) { print_int(foo.get()); }
}
```
2个写线程1个读线程，写线程在循环里用data里的元素值设置foo对象的x成分，读线程简单的打印foo对象的x值。

程序一直跑下去，最后打印出来的数据，会出现除data初始化值外的数据吗？

`int32_t Foo::get() const`的实现有问题吗？如果有问题？是什么问题？

## 示例6
看一个用数组实现FIFO队列的程序，一个线程写`put()`，一个线程读`get()`。
```c++
// example 6
#include <iostream>
#include <algorithm>

// 用数组实现的环型队列
class FIFO {
    static const unsigned int CAPACITY = 1024;  // 容量：需要满足是2^N

    unsigned char buffer[CAPACITY];             // 保存数据的缓冲区
    unsigned int in = 0;                        // 写入位置
    unsigned int out = 0;                       // 读取位置

    unsigned int free_space() const { return CAPACITY - in + out; }
public:
    // 返回实际写入的数据长度（<= len），返回小于len时对应空闲空间不足
    unsigned int put(unsigned char* src, unsigned int len) {
        // 计算实际可写入数据长度（<=len）
        len = std::min(len, free_space());

        // 计算从in位置到buffer结尾有多少空闲空间
        unsigned int l = std::min(len, CAPACITY - (in & (CAPACITY - 1)));
        // 1. 把数据放入buffer的in开始的缓冲区，最多到buffer结尾
        memcpy(buffer + (in & (CAPACITY - 1)), src, l);   
        // 2. 把数据放入buffer开头（如果上一步还没有放完），len - l为0代表上一步完成数据写入
        memcpy(buffer, src + l, len - l);
        
        in += len; // 修改in位置，累加，到达uint32_max后溢出回绕
        return len;
    }

    // 返回实际读取的数据长度（<= len），返回小于len时对应buffer数据不够
    unsigned int get(unsigned char *dst, unsigned int len) {
        // 计算实际可读取的数据长度
        len = std::min(len, in - out);

        unsigned int l = std::min(len, CAPACITY - (out & (CAPACITY - 1)));
        // 1. 从out位置开始拷贝数据到dst，最多拷贝到buffer结尾
        memcpy(dst, buffer + (out & (CAPACITY - 1)), l);
        // 2. 从buffer开头继续拷贝数据（如果上一步还没拷贝完），len - l为0代表上一步完成数据获取
        memcpy(dst + l, buffer, len - l);

        out += len; // 修改out，累加，到达uint32_max后溢出回绕
        return len;
    }
};
```
环型队列只是逻辑上的概念，因为采用了数组作为数据结构，所以实际物理存储上并非环型。
- `put()`用于往队列里放数据，参数`src+len`描述了待放入的数据信息。
- `get()`用于从队列取数据，参数`dst+len`描述了要把数据读到哪里、以及读多少字节。
- `capacity`精心选择为2的n次方，可以得到2个好处：
    - 非常技巧性的利用了无符号整型溢出回绕，便于处理对`in`和`out`移动。
    - 便于计算长度，通过按位与操作`&`而不必除余。
    - 搜`kfifo`获得更详细的解释。
- `in`和`out`是2个游标。
    - `in`用来指向新写入数据的存放位置，写入的时候，只需要简单增加in。
    - `out`用来指示从buffer的什么位置读取数据的，读取的时候，也只需简单增加out。
    - `in`和`out`在操作上之所以能单调增加，得益于上述capacity的精心选择。
- 为了简化，队列容量被限制为1024字节，不支持扩容，这不影响多线程的讨论。

写的时候，先写入数据再移动out游标；读的时候，先拷贝数据，再移动in游标；out游标移动后，消费者才获得`get`到新放入数据的机会。

直觉告诉我们2个线程不加同步的并发读写，会有问题，但真有问题吗？如果有，到底有什么问题？

# 要保护什么？
多线程程序里，我们要保护的是数据而非代码，虽然Java等语言里有临界代码、sync方法，但最终要保护的还是代码访问的数据。

# 串行化
如果有一个线程正在访问某共享（临界）资源，那么在它结束访问之前，其他线程不能执行访问同一资源的代码（访问临界资源的代码叫临界代码），其他线程想要访问同一资源，则它必须等待，直到那个线程访问完成，它才能获得访问的机会，现实中有很多这样的例子。

比如高速公路上的汽车过检查站，假设检查站只有一个车道，则无论高速路上有多少车道，过检查站的时候只能一辆车接着一辆车，从单一车道鱼贯而入。

对多线程访问共享资源施加这种约束就叫串行化。

# 原子操作和原子变量
针对前面的两个线程对同一整型变量自增的问题，如果“load、update、store”这3个步骤是不可分割的整体，即自增操作`++x`满足原子性，上面的程序便不会有问题。

因为这样的话，2个线程并发执行`++x`，只会有2个结果：
- 线程1`++x`，然后，线程2`++x`，结果是2
- 线程2`++x`，然后，线程1`++x`，结果是2

除此之外，不会出现第三种情况，线程1、2孰先孰后，取决于线程调度，但不影响最终结果。

Linux操作系统和C/C++编程语言都提供了整型原子变量，原子变量的自增、自减等操作都是原子的，操作是原子性的，意味着它是一个不可细分的操作整体，原子变量的用户观察它，只能看到未完成和已完成2种状态，看不到半完成状态。

如何保证原子性是实现层面的问题，应用程序员只需要从逻辑上理解原子性，并能恰当的使用它就行了。

原子变量非常适用于计数、产生序列号这样的应用场景。

# 锁
前面举了很多例子，阐述多线程不加同步并发访问数据会引起什么问题，下面讲解用锁如何做同步。

## 互斥锁
针对线程1`write_msg()` + 线程2`read_msg()`的问题，如果能让线程1`write_msg()`的过程中，线程2不能`read_msg()`，那就不会有问题。

这个要求，其实就是要让多个线程互斥访问共享资源。

互斥锁就是能满足上述要求的同步机制，互斥是排他的意思，它可以确保在同一时间，只能有一个线程对那个共享资源进行访问。

互斥锁有且只有2种状态：
- 已加锁（locked）状态
- 未加锁（unlocked）状态

互斥锁提供加锁和解锁两个接口：
- 加锁（acquire）：当互斥锁处于未加锁状态时，则加锁成功（把锁设置为已加锁状态），并返回；当互斥锁处于已加锁状态时，那么试图对它加锁的线程会被阻塞，直到该互斥量被解锁。
- 解锁（release）：通过把锁设置为未加锁状态释放锁，其他因为申请加锁而陷入等待的线程，将获得执行机会。如果有多个等待线程，只有一个会获得锁而继续执行。

我们为某个共享资源配置一个互斥锁，使用互斥锁做线程同步，那么所有线程对该资源的访问，都需要遵从“加锁、访问、解锁”的三步：
```c++
DataType shared_resource;
Mutex shared_resource_mutex;

void shared_resource_visitor1() {
    // step1: 加锁
    shared_resource_mutex.lock();
    // step2: operate shared_resouce
    // operation1
    // step3: 解锁
    shared_resource_mutex.unlock();
}

void shared_resource_visitor2() {
    // step1: 加锁
    shared_resource_mutex.lock();
    // step2: operate shared_resouce
    // operation2
    // step3: 解锁
    shared_resource_mutex.unlock();
}
```
`shared_resource_visitor1()`和`shared_resource_visitor2()`代表对共享资源的不同操作，多个线程可能调用同一个操作函数，也可能调用不同的操作函数。

假设线程1执行`shared_resource_visitor1()`，该函数在访问数据之前，申请加锁，如果互斥锁已经被其他线程加锁，则调用该函数的线程会阻塞在加锁操作上，直到其他线程访问完数据，释放（解）锁，阻塞在加锁操作的线程1才会被唤醒，并尝试加锁：
- 如果没有其他线程申请该锁，那么线程1加锁成功，获得了对资源的访问权，完成操作后，释放锁。
- 如果其他线程也在申请该锁，那么：
    - 如果其他线程抢到了锁，那么线程1继续阻塞。
    - 如果线程1抢到了该锁，那么线程1将访问资源，再释放锁，其他竞争该锁的线程得以有机会继续执行。

如果不能承受加锁失败而陷入阻塞的代价，可以调用互斥量的`try_lock()`接口，它在加锁失败后会立即返回。

注意：在访问资源前申请锁，是一个编程契约，通过遵守契约而获得数据一致性的保障，它并非一种硬性的限制，即如果别的线程遵从三步曲，而另一个线程不遵从这种约定，代码能通过编译且程序能运行，但结果可能是错的。

## 读写锁
读写锁跟互斥锁类似，也是申请锁的时候，如果不能得到满足则阻塞，但读写锁跟互斥锁也有不同，读写锁有3个状态：
- 已加读锁状态
- 已加写锁状态
- 未加锁状态

对应3个状态，读写锁有3个接口：加读锁，加写锁，解锁：
- 加读锁：如果读写锁处于**已加写锁**状态，则申请锁的线程阻塞，否则把锁至于**已加读锁**成功返回。
- 加写锁：如果读写锁处于**未加锁**状态，则把锁设置为**已加读锁**状态，并成功返回；否则阻塞。
- 解锁：把锁设置为**未加锁**状态后返回。

可见，读写锁提升了线程的并行度，可以提升吞吐。它可以让多个读线程同时读共享资源，而写线程访问共享资源的时候，其他线程不能执行，所以，读写锁适合对共享资源访问“读大于写”的场合。

读写锁也叫“共享互斥锁”，多个读线程可以并发访问同一资源，这对应共享的概念，而写线程是互斥的，写线程访问资源的时候，其他线程无论读写，都不可以进入临界代码区。

考虑一个场景，如果有线程1、2、3共享资源x，读写锁rwlock保护资源，线程1读访问某资源，然后线程2以写的形式访问同一资源x，因为rwlock已经被加了读锁，所以线程2被阻塞，然后过了一段时间，线程3也读访问资源x，这时候线程3可以继续执行，因为读是共享的，然后线程1读访问完成，线程3继续访问，过了一段时间，在线程3访问完全前，线程1又申请访问资源，那么它还是会获得访问权，但是读资源的线程2会一直被阻塞。

为了避免共享的读线程饿死写线程，通常读写锁的实现，会给写线程优先权，当然这处决于读写锁的实现，作为读写锁的使用方，理解它的语义和使用场景就够了。

## 自旋锁
自旋锁（Spinlock）的接口跟互斥量差不多，但实现原理不同。

线程在acquire自旋锁失败的时候，它不会主动让出CPU从而进入睡眠状态，而是会忙等，它会紧凑的执行**测试和设置(Test-And-Set)**指令，直到TAS成功，否则就一直占着CPU做TAS。

自旋锁对使用场景有一些期待，它期待acquire自旋锁成功后很快会release锁，线程运行临界区代码的时间很短，访问共享资源的逻辑简单，这样的话，别的acquire自旋锁的线程只需要忙等很短的时间就能获得自旋锁，从而避免被调度走陷入睡眠，它假设自旋的成本比调度的低，它不愿耗费时间在线程调度上（线程调度需要保存和恢复上下文需要耗费CPU）。

内核态线程很容易满足这些条件，因为运行在内核态的中断处理函数里可以通过关闭调度，从而避免CPU被抢占，而且有些内核态线程调用的处理函数不能睡眠，只能使用自旋锁。

而运行在用户态的应用程序，则推荐使用互斥锁等睡眠锁。因为运行在用户态应用程序，虽然很容易满足临界区代码简短，但持有锁时间依然可能很长。在分时共享的多任务系统上、当用户态线程的时间配额耗尽，或者在支持抢占式的系统上、有更高优先级的任务就绪，那么持有自旋锁的线程就会被系统调度走，这样持有锁的过程就有可能很长，而忙等自旋锁的其他线程就会白白消耗CPU资源，这样的话，就跟自旋锁的理念相背。

Linux系统优化过后的mutex实现，在加锁的时候会先做有限次数的自旋，只有有限次自旋失败后，才会进入睡眠让出CPU，所以，实际使用中，它的性能也足够好。

此外，自旋锁必须在多CPU或者多Core架构下，试想如果只有一个核，那么它执行自旋逻辑的时候，别的线程没有办法运行，也就没有机会释放锁。

## 锁的粒度和范围
合理设置锁的粒度
锁的范围要尽量小，最小化持有锁的时间

## 死锁
程序出现死锁有两种典型原因：

### ABBA锁
假设程序中有2个资源X和Y，分别被锁A和B保护，线程1持有锁A后，想要访问资源Y，而访问资源Y之前需要申请锁B，而如果线程2正持有锁B，并想要访问资源X，为了访问资源X，所以线程2需要申请锁A。

线程1和线程2分别持有锁A和B，并都希望申请对方持有的锁，因为线程申请对方持有的锁，得不到满足，所以便会陷入等待，也就没有机会释放自己持有的锁，对方执行流也就没有办法继续前进，导致相持不下，无限互等，进而死锁。

上述的情况似乎很明显，但如果代码量很大，有时候，这种死锁的逻辑不会这么浅显，它被复杂的调用逻辑所掩盖，但抽茧剥丝，最根本的逻辑就是上面描述的那样。

这种情况叫ABBA锁，既某个线程持有A锁申请B锁，而另一个线程持有B锁申请A锁。

这种情况可以通过try lock实现，尝试获取锁，如果不成功，则释放自己持有的锁，而不一根筋下去。

另一种解法就是锁排序，对A/B两把锁的加锁操作，都遵从同样的顺序（比如先A后B），也能避免死锁。

### 自死锁
对于不支持重复加锁的锁，如果线程持有某个锁，而后又再次申请锁，因为该锁已经被自己持有，再次申请锁必然得不到满足，从而导致死锁。

# 条件变量：生产者/消费者模式
条件变量需要配合互斥量使用，常用于生产者消费者模式。

假设你要编写一个网络处理程序，I/O线程从套接字接收字节流，反序列化后产生一个个消息（自定义协议），然后投递到一个消息队列，一组工作线程负责从消息队列取出并处理消息。

这是典型的生产者-消费者模式，I/O线程生产消息（往队列put），Work线程消费消息（从队列get），I/O线程和Work线程并发访问消息队列，显然，消息队列是竞争资源，需要同步。

我们可以给队列配置互斥锁，put和get操作都先加锁，操作完成再解锁。代码差不多是这样的：

```c++
void io_thread() {
    while (1) {
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

work线程组的每个线程都忙于检查消息队列是否有消息，如果有消息就取一个出来，然后处理消息，如果没有消息就在循环里不停检查，这样的话，即使负载很轻，但work_thread还是会消耗大量的CPU时间，我们当然可以在两次查询之间加入短暂的`sleep`，从而让出cpu，但是这个睡眠的时间设置为多少合适呢？设置长了的话，会出现消息到来得不到及时处理（延迟上升），设置太短了，还是无辜消耗了CPU资源，这种不断问询的方式在编程上叫轮询。

轮询行为逻辑上，相当于你在等一个投递到楼下小邮局的包裹，你下楼查验没有之后就上楼回房间，然后又下楼查验，你不停的上下楼查验，其实，大可不必如此，何不等包裹到达以后，让门卫打电话通知你去取呢？

条件变量提供了一种类似通知`notify`的机制，它让两类线程能够在一个点交汇。条件变量能够让线程等待某个条件发生，条件本身受互斥锁保护，因此条件变量必须搭配互斥锁使用，锁保护条件，线程在改变条件前先获得锁，然后改变条件状态，再解锁，最后发出通知，等待条件的睡眠中的线程在被唤醒前，必须先获得锁，再判断条件状态，如果条件不成立，则继续转入睡眠并释放锁。

对应到上面的例子，工作线程等待的条件是消息队列有消息（非空），用条件变量改写上面的代码：

```c++
void io_thread() {
    while (1) {
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
`std::lock_guard`是互斥量的一个`RAII`包装类，`std::unique_lock`除了会在析构函数自动解锁外，还支持主动`unlock()`。

生产者在往msg_queue投递消息的时候，需要对msg_queue加锁，通知work线程的代码可以放在解锁之后，等待msg_queue_not_empty条件必须受msg_queue_mutex保护，wait的第二个参数是一个lambda表达式，因为会有多个work线程被唤醒，线程被唤醒后，会重新获得锁，检查条件，如果不成立，则再次睡眠，条件变量的使用必须非常谨慎，否则容易出现不能唤醒的情况。

C语言的条件变量、Posix条件变量的编程接口跟C++的类似，概念上是一致的，故在此不展开介绍。

# lock-free和无锁数据结构
## 锁同步的问题
线程同步分为阻塞型同步和非阻塞型同步。

互斥量、信号、条件变量这些系统提供的机制都属于阻塞型同步，在争用资源的时候，会导致调用线程阻塞。

而非阻塞型同步是指在无锁的情况下，通过某种算法和技术手段实现不用阻塞而同步。

锁是阻塞同步机制，阻塞同步机制的缺陷是可能挂起你的程序，如果持有锁的线程崩溃或者hang住，则锁永远得不到释放，而其他线程则将陷入无限等待；另外，它也可能导致优先级倒转等问题。

所以，我们需要lock-free这类非阻塞的同步机制。

## 什么是lock-free
lock-free没有锁同步的问题，所有线程无阻碍的执行原子指令，而不是等待。

比如一个线程读atomic类型变量，一个线程写atomic变量，它们没有任何等待，硬件原子指令确保不会出现数据不一致，写入数据不会出现半完成，读取数据也不会读一半。

那到底什么是lock-free？

有人说lock-free就是不使用mutex / semaphores之类的无锁（lock-Less）编程，这句话严格来说并不对。

我们先看一下wiki对Lock-free的描述:
> Lock-freedom allows individual threads to starve but guarantees system-wide throughput. An algorithm is lock-free if, when the program threads are run for a sufficiently long time, at least one of the threads makes progress (for some sensible definition of progress). All wait-free algorithms are lock-free.

> In particular, if one thread is suspended, then a lock-free algorithm guarantees that the remaining threads can still make progress. Hence, if two threads can contend for the same mutex lock or spinlock, then the algorithm is not lock-free. (If we suspend one thread that holds the lock, then the second thread will block.)

翻译一下：
- 第一段：lock-free允许单个线程饥饿但保证系统级吞吐。如果一个程序线程执行足够长的时间，那么至少一个线程会往前推进，那么这个算法就是lock-free的。​​​
- 第二段：尤其是，如果一个线程被暂停，lock-free算法保证其他线程依然能够往前推进。

第一段给lock-free下定义，第二段是对lock-free做解释。

如果2个线程竞争同一个互斥锁或者自旋锁，那它就不是lock-free的。因为如果暂停（Hang）持有锁的线程，那么另一个线程会被阻塞。

wiki的这段描述很抽象，它不够直观，稍微再解释一下：

lock-free描述的是代码逻辑的属性，不使用锁的代码，大部分具有这种属性。

大家经常会混淆这lock-free和无锁这2个概念。其实，lock-free是对代码（算法）性质的描述，是属性；而无锁是说代码如何实现，是手段。

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

这个时候，线程A如果因为某个原因（如出现异常，或者等待某个资源）而被永久挂起，那么同样执行push的线程B/C将被永久挂起，系统整体（system-wide）没法推进，而这显然不符合lock-free的要求。

因此：所有基于锁（包括spinlock）的并发实现，都不是lock-free的。

因为它们都会遇到同样的问题：即如果永久暂停当前占有锁的线程/进程的执行，将会阻塞其他线程/进程的执行。

而对照lock-free的描述，它允许部分process（理解为执行流）饿死但必须保证整体逻辑的持续前进，基于锁的并发显然是违背lock-free要求的。

## CAS loop实现lock-free
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

# 内存屏障
TODO

# 伪共享
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

## 改进版f_fast
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

## f和f_fast的行为差异
shm数组总共有2M个long元素，因为16M / sizeof(long) => 2M
1. f()函数行为逻辑
    - 线程1和线程2的work_thread里会交错地对shm元素赋值，shm的2M个long元素，会顺序的一个接一个的派给2个线程去赋值。
    - 可能元素1有线程1赋值，元素2由线程2赋值，然后元素3和元素4由线程1赋值，然后元素5又由线程2赋值... 
    - 每次派元素的时候，shm_offset都会atomic的增加8字节，所以不会出现2个线程给1个元素赋值的情况

2. f_fast()函数行为逻辑
    - 每次派元素的时候，shm_offset原子性的增加128字节（16个元素）
    - 这16个字节作为一个整体，派给线程1或者线程2；虽然线程1和线程2还是会交错的操作shm元素，但是以16个元素（128字节）为单元，这16个连续的元素不会被分开派发
    - 一次派发的16个元素，会在内部循环里被一个接着一个的赋值，在一个线程里被执行

## 为什么f_fast更快？
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

## 答案
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

## 另一个伪共享的例子
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

## 伪共享的疑问
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

# 附录
