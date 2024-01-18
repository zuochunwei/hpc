# 数据结构

## hashtable

Hashtable 是这个非常复杂的数据结构，通常由以下四部分组成。

* 哈希函数
* 碰撞解决策略
* 扩容策略
* 内存布局

每个组件对 hashtable 的性能都是有影响的，并且组件之中也是互相影响的，比如内存布局也会影响碰撞解决策略和哈希函数的选择。

### 哈希函数

哈希函数又称为散列函数。哈希算法广泛应用于很多场景，例如加密，索引，认证等，一般来说，加密场景的hash 函数安全性更高，但性能差，而用于查找的哈希函数则性能更好。

常见的加密哈希算法比如MD5,SHA1,SHA-256, CHACHA20等

常见的查找哈希算法：crc32c,Murmur hash,xxhash, cityhash等

**murmurhash**

murmurhash 是 Austin Appleby 于 2008 年创立的一种非加密哈希算法，适用于基于哈希进行查找的场景。MurMur 经常用在分布式环境中，比如 Hadoop，其特点是高效快速，但是缺点是分布不是很均匀。

**CityHash**

CityHash是由google在2011年 发布的, 其性能好于 MurmurHash。但后来 CityHash 的哈希算法被发现容易受到针对算法漏洞的攻击，该漏洞允许多个哈希冲突发生。

**xxhash**

xxhash 由 Yann Collet 发表，http://cyan4973.github.io/xxHash/ 这是它的官网，据说性能很好，似乎被很多开源项目使用，Bloom Filter 的首选。

### 碰撞解决策略

hash 函数计算时，不可避免的会产生 hash 冲突，hash 冲突指的是两个不同的数据落入了同一个 hash 桶内。解决 hash 冲突常见的有三种方法：开放定址法，链地址法，再哈希法。`unordered_map` 使用的就是链地址法。

链式连接具有以下优点：

- 指向键、值的指针的稳定性。
- 能够存储大型物体、不可移动物体。
- 即使哈希函数不好或负载系数非常高，也能很好地工作。

但由于以下缺点，它们在实践中并不经常用于高性能场景：

- 内存开销高。
- 加载分配器（即使只是一个函数调用在哈希表查找路径上也是昂贵的）。
- 缓存局部性差，此类哈希表在查找过程中会进行大量随机内存访问。

开放定址法分为线性探测法，平方探测法，再哈希法，Robin Hood 哈希法。

开放式寻址具有以下优点：

- 内存开销低。通常我们只需要存储键和值。
- 良好的缓存局部性。例如，我们只需从内存中获取一次即可查找或插入密钥。对于不适合 CPU 缓存的大型哈希表来说，这些变得更加重要，其中内存访问延迟是哈希表性能的最重要因素。



### 扩容方法

最流行的方法是使用哈希表大小作为 2 的幂。因此，在每次调整大小期间，我们都会将哈希表大小加倍。这种方法很好，因为正如我们已经讨论过的，我们希望在哈希表查找过程中花费尽可能少的时间。如果表适合 CPU 缓存，则它应该在纳秒内发生，并且如果哈希表大小是 2 的幂，则存储桶计算会很快：

```
size_t place = hash & (size - 1)
```

* linear hashing

  线性哈希是Witold Litwin于1980年首次描述的动态哈希算法。它是一种开放寻址方法，用于数据库系统为新数据记录分配空间。它是可扩展哈希算法的变体，该算法使用一种称为桶分割的技术，在需要更多空间时增加哈希表的大小。线性哈希通过从哈希表中添加或删除桶来支持数据库系统的动态增长和重新组织。

### 内存布局

* 特殊键值

* Null 键值

* 元数据

### consistent hashing

一致性哈希算法（Consistent Hashing algorithm）是一种用来解决分布式系统中存储节点管理的数据一致性算法。它可以让分布式系统中的节点有效地添加、删除和维护，而无需重新映射所有数据。一致性哈希算法通过将数据分割成一系列的桶，将数据映射到存储节点上，以此来确保在节点添加或删除时只需要对附近的桶进行重新映射，而不需要对所有的桶进行重新映射。

在分布式系统中，我们可以通过hash分布的方式将数据均匀分布在所有的分布式节点上。但当分布式系统的拓扑发生改变时，也就是增删节点时，往往需要进行数据的重新分布。一致性哈希就是为了解决这一问题而诞生的。

一致性哈希是指将「存储节点」和「数据」都映射到一个首尾相连的哈希环上，如果增加或者移除一个节点，仅影响该节点在哈希环上顺时针相邻的后继节点，其它数据也不会受到影响。 但是一致性哈希算法不能够均匀的分布节点，会出现大量请求都集中在一个节点的情况，在这种情况下进行容灾与扩容时，容易出现雪崩的连锁反应。

Karger在1997年时第一次提出了一致性hash的算法。在2014年Google在论文A Fast, Minimal Memory, Consistent Hash Algorithm提出了最新的一致性hash算法，相比于原来的算法有了很大的提升。下面就是在论文中提出的JumpConsistentHash的算法实现。

```
int32_t JumpConsistentHash(uint64_t key, int32_t num_buckets) {
    int64_t b = -1, j = 0;
    while (j < num_buckets) {
        b = j;
        key = key * 2862933555777941757ULL + 1;
        j = (b + 1) * (double(1LL << 31) / double((key >> 33) + 1));
    }
    return b;
}
```



## HLL

HyperLogLog简称HLL，是一种不精确的去重计数算法，用于解决计算机科学中的count distinct问题和应用数学中的基数估算问题。通常提到去重计算的数据结构，我们可以想到set，bitmap，hashmap等。但在数据量较小的场景下，以上算法尚且可用，但当数据量达到上亿级别时，以上数据结构的内存消耗则迅速变大，性能也不尽如人意。以占用内存较小的bitmap为例，每个元素在bitmap占据一个bit位，对于一百亿条数据的集合，需要10^11/8/1024/1024=1.2GB内存。此时，HyperLogLog就派上了用场了。

首先我们思考一下如果我们要统计一个网站中的用户访问数应该怎么做？

首先我们可以将每个用户的ID通过hash计算后得到一个Int64的整型值，接下来我们做一系列实验来统计从低位到高位来计算第一个1出现的位置。

假设一共有n个用户，那么实验要估算的数据就是n，第一个1出现的位置前导零的个数用k_max表示。比如下面的hash value的k_max为3.

![hll01](pic/hll01.png)

当我们的实验次数足够多时我们可以从下面的公式来估算n的个数
$$
n \approx 2^{k\_max}
$$
但上面的估算在仍然比较粗糙，特别是当统计数量比较少的时候。所以需要一个修正系数来修正结果的正确性。最终的计算公式为
$$
n \approx \frac{2^{k\_max}}{\phi} ；{\phi} ≈ 0.77351
$$
这就是 [Flajolet-Martin](https://en.wikipedia.org/wiki/Flajolet–Martin_algorithm)算法。但上面的算法仍然有不少问题，比如如果样本中有一个hashvalue的k_max非常大，则结果将非常不准确，为了解决某一样本对整体估算结果的影响，LogLog算法就出现了。

LogLog算法做了一个优化就是分桶。假设对于上述所有用户的ID经散列函数计算后得到的hash值，取低4位作为分桶的标记位，分桶的个数为m，然后统计接下来的16位中第一个1出现的位置前导零的个数位k。

![hll02](pic/hll02.png)

最后将所有桶的k最大值取平均值。那么公式就变为了
$$
n \approx const \times m\times 2^{\displaystyle \frac{k1+k2+...+km}{m}}
$$
LogLog算法采用的是几何平均值来计算k,但缺点是仍然受某些极值的影响比较大，比如`k_max1=100, k_max2=1`,平均下来就是`50.5`，这样的统计信息仍然不准确。

HyperLogLog算法在LogLog算法的基础上，将几何平均值改为了调和平均值。调和平均值将极值对整个统计结果的影响降到了最低，从而将估算的准确率提高到了最高的水平。
$$
n \approx const \times m\times {\displaystyle \frac{m}{2^ {1/k1+1/k2+...+1/km}}}
$$
以上就是HyperLogLog算法的原理介绍。从[HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf)论文可以看到详细的推导证明。

HyperLogLog有非常广泛的应用，在数据库系统中，基数估算是一类影响优化器决策的重要参数。在Presto等数据库中，HLL常用来估算某一个数据表的行数。



## RoaringBitmap

在了解RoaringBitmap之前我们先得了解什么是Bitmap。Bitmap索引经常被用在数据库和搜索引擎中。通过利用位并行运算的优势，它能够显著地加速查询。但是，它也有一个缺点，那就是会耗费更多的内存。比如对于uint32的整数类型，整数范围最大为2^32-1，为构建该整数的位图结构，需要占用2^32/8/1024/1024=512M大小的内存空间。但通常情况下，位图是稀疏的，所以可以通过压缩位图来减少内存的使用。所以便有了RoaringBitmap，RoaringBitmap表示压缩位图索引。在Bitmap压缩方案中，Roaring bitmaps比基于RLE(Run-length encoding)的压缩性能更好。

RoaringBitmap简称为RBM，于2016年由S. Chambi、D. Lemire、O. Kaser等人在论文《Better bitmap performance with Roaring bitmaps》与《Consistently faster and smaller compressed bitmaps with Roaring》中提出。RBM的原理是：将32位无符号整数分为高16位和低16位。按照高16位分桶,也叫做container，即最多可能有`2^ 16=65536`个container。插入数据时，先按照高16位找到container，如果未找到则新建一个，再将低16位放入container中。

低16位的存储分为三种数据结构，ArrayContainer, BitmapContainer,RunContainer。

ArrayContainer
当桶内数据的基数不大于4096时，会采用ArrayContainer来存储。其本质上是一个unsigned short类型的有序数组。数组初始长度为4，随着数据的增多会自动扩容，最大长度就是4096。当数据达到4096时占用内存大小为8KB。另外还包括有一个计数器，用来实时记录Container内数据的总数。

BitmapContainer
当桶内数据的基数大于4096时，会采用BitmapContainer来存储。其本质就是普通Bitmap位图，用长度固定为1024的unsigned long型数组表示，每个unsigned long是8个字节，64位，即位图的大小固定为2^16（8KB）。它同样有一个计数器。

RunContainer
RunContainer使用可变长度的unsigned short数组存储用行程长度编码（RLE）压缩后的数据。

例如，连续的整数序列11, 12, 13, 14, 15, 27, 28, 29会被RLE压缩为两个二元组11, 4, 27, 2，表示11后面紧跟着4个连续递增的值，27后面跟着2个连续递增的值。

由此可见，RunContainer的压缩效果可好可坏。考虑极端情况：如果所有数据都是连续的，那么最终只需要4字节；如果所有数据都不连续（比如全是奇数或全是偶数），那么不仅不会压缩，还会膨胀成原来的两倍大。所以，RBM引入RunContainer是作为其他两种container的折衷方案。

性能分析：对于以上三种Container，BitmapContainer可根据下标直接寻址，复杂度为`O(1)`，ArrayContainer和RunContainer都需要二分查找，复杂度`O(log n)`。


内存分析，BitmapContainer是恒定为8KB的。ArrayContainer的空间占用与基数（c）有关，为(2 + 2c)B；RunContainer的则与它存储的连续序列数（r）有关，为(2 + 4r)B

## BloomFilter

在计算机科学中，查找问题是一类重要的问题。查找问题是查找某一个元素是否存在于集合中，比如我们熟知的二分查找等等。为了解决这类查找问题时，我们设计了很多高效的数据结构，除了最常见的set，hashmap，bitmap外，BloomFilter就是一种空间效率非常高的概率型数据结构。它最早是由Bloom在1970年提出的。BloomFilter之所以称之为概率型数据结构，是因为它可以判断一个元素一定不在这个集合中，但不能判断一个元素是否存在集合中，也就是存在一定的误判率。在较小的误判率下，BloomFilter可以显著提升系统的查询性能，并且减少对系统的内存消耗。

BloomFilter是由bit位组成的位数组组成，初始化为0，每一个将要插入的元素都可以映射到BloomFilter位数组中的某一个bit位，映射的方法就是我们最常使用的散列函数。BloomFilter的基本原理就是将集合中的所有元素通过散列函数映射到位数组中，映射的bit位置为1。待查找的元素同样同样的散列函数找到对应的bit位，如果bit位为1，则该元素可能存在，如果bit位不为1，则该元素不可能存在。

BloomFilter有几个关键的参数。

`m`代表由m个bit组成的位数组，初始化全部为0。

`k`代表有k个散列函数，每个散列函数的作用是将集合中的某个元素哈希映射到位数组的某一个位置，并将对应的bit位置为1。

BloomFilter像其它数据结构一样需要两个最重要的方法：插入和查找。

插入：将要插入的一个元素经过哈希函数映射到位数组的某一个bit位，将对应的bit位置为1。所以k个哈希函数可以将位数组的k个bit位置为1。

查找：将要查找的元素经过k次哈希函数计算后找到位数组的k个bit位，如果对应的bit位都为1，则表示该元素可能存在于集合中。而如果有一个bit位不为1，则表示该元素一定不在集合中。

从以上的介绍中可以发现，BloomFilter可能存在误判，也就是说某一个元素可能不存在集合中，但其对应的bit位都为1.

衡量BloomFilter的误判率称为false positives。 false positives可以通过以下公式计算得到
$$
\epsilon = (1-(1-1/m)^{kn})^{k} \approx(1-e^{-kn/m})^{k}
$$
n表示插入的元素的个数。

我们的目标就是使得误判率最小，对上式两侧求导，我们可以得出最优的散列函数的个数k与m，n之间的关系如下。
$$
k = \frac{m}{n} ln2
$$
位数组中位的大小m可以计算通过下面的计算得到
$$
m = \frac {-nln\epsilon}{(ln2)^2}
$$
一般我们可以根据误判率来计算得到位数组的大小m，同时得到散列函数的个数k。

通过以上的推导我们可以知道，通过调整散列函数的个数k，以及位数组的大小m可以寻求一定误判率下的时间效率和空间效率的平衡。k越大表示需要计算的散列函数次数越多，则性能相应的会降低，m越大表示需要占用的内存空间越大，空间效率更差。在给定集合中元素个数和误判率的情况下，我们可以得到最优的位数组大小m和散列函数个数k。

由上面的分析我们可以写出一个bloomfilter的实现

```
class BloomFilter
  {
    public:
        BloomFilter(size_t size_, size_t hashes_);

        bool find(const char * data, size_t len);
        void add(const char * data, size_t len);

    private:
        size_t size;
        size_t hashes;
        std::vector<char> filter;
  };

  BloomFilter::BloomFilter(size_t size_, size_t hashes_)
    : size(size_), hashes(hashes_), filter(size, 0)
  {
  }

  bool BloomFilter::find(const char * data, size_t len)
  {
      for (size_t i = 0; i < hashes; ++i)
      {
          size_t pos = std::hash<std::string_view>()(std::string_view{data, len}) % (8 * size);
          if (!(filter[pos / 8] & (1ULL << (pos % 8 ))))
                  return false;
      }
      return true;
  }

  void BloomFilter::add(const char * data, size_t len)
  {
      for (size_t i = 0; i < hashes; ++i)
      {
          size_t pos = std::hash<std::string_view>()(std::string_view{data, len}) % (8 * size);
          filter[pos / 8] |= (1ULL << (pos % 8));
      }
  }
```

BloomFilter有多种变种，比如ribbon filters， cuckoo filter， block bloom filter，Xor Filter等。相比于经典的BloomFilter算法，新的变种在性能和空间使用上已经有了很大的提升。

BloomFilter有非常广泛的应用。假如我们做一个kv-store，为了加速会在cache里做kv粒度的缓存。所以 kv-store.lookup(key)的逻辑是，先在cache里查找，找不到再去磁盘文件里搜索。这时就可以用bloom filter，先去bloom filter里查一下，如果返回真，那就去cache里找，如果找不到（可能存在误判），再去磁盘文件里找。

## SkipList

跳表，Btree+的不稳定实现。是一个有序的链表。跳表的查询的平均时间复杂度为 O(logn）插入和删除的平均时间复杂度也为 O(logn)。



## ConcurrentHashmap

线程安全的HashMap实现。







Reference:

1. HyperLogLog 

   https://towardsdatascience.com/hyperloglog-a-simple-but-powerful-algorithm-for-data-scientists-aed50fe47869

   https://engineering.fb.com/2018/12/13/data-infrastructure/hyperloglog/

   http://content.research.neustar.biz/blog/hll.html

2. BloomFilter

   http://oserror.com/backend/bloomfilter/

3. Hash

   http://cyan4973.github.io/xxHash/

   http://www.alloyteam.com/2017/05/hash-functions-introduction/

4. Linear Hash

   https://ruby-china.org/topics/39466

   https://hackthology.com/linear-hashing.html

   https://dsf.berkeley.edu/jmh/cs186/f02/lecs/lec18_2up.pdf

5. RoaringBitmap

   https://blog.csdn.net/yizishou/article/details/78342499?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-78342499-blog-119736050.pc_relevant_multi_platform_featuressortv2dupreplace&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-78342499-blog-119736050.pc_relevant_multi_platform_featuressortv2dupreplace&utm_relevant_index=1

   https://www.jianshu.com/p/818ac4e90daf
