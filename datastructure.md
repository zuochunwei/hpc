# 数据结构

## hash

crc32c,Murmur hash, cityhash是目前最快的hash函数。由Google提出。

### consistent hashing

一致性哈希是指将「存储节点」和「数据」都映射到一个首尾相连的哈希环上，如果增加或者移除一个节点，仅影响该节点在哈希环上顺时针相邻的后继节点，其它数据也不会受到影响。 但是一致性哈希算法不能够均匀的分布节点，会出现大量请求都集中在一个节点的情况，在这种情况下进行容灾与扩容时，容易出现雪崩的连锁反应。

### linear hashing

动态调整hashtable

## HLL

HyperLogLog简称HLL，是一种不精确的去重计数算法，用于解决计算机科学中的count distinct问题和应用数学中的基数估算问题。通常提到去重计算的数据结构，我们可以想到set，bitmap，hashmap等。但在数据量较小的场景下，以上算法尚且可用，但当数据量达到上亿级别时，以上数据结构的内存消耗则迅速变大，性能也不尽如人意。以占用内存较小的bitmap为例，每个元素在bitmap占据一个bit位，对于一百亿条数据的集合，需要10^11/8/1024/1024=1.2GB内存。此时，HyperLogLog就派上了用场了。

首先我们思考一下如果我们要统计一个网站中的用户访问数应该怎么做？

首先我们可以将每个用户的ID通过hash计算后得到一个Int64的整型值，接下来我们做一系列实验来统计从低位到高位来计算第一个1出现的位置。

假设一共有n个用户，那么实验对象的个数就是n，第一个1出现的位置用k_max表示。

当我们的实验次数足够多时我们可以推导出
$$
n \approx 2^{k\_max}
$$
来估算n的个数。

但上面的估算在仍然比较粗糙，特别是当统计数量比较少的时候。

所以LogLog算法就出现了

LogLog算法做了一个优化就是分桶。假设对于上述所有用户的ID经散列函数计算后得到的hash值，取低6位作为分桶的标记位，然后统计接下来的24位中第一个1出现的位置。最后讲所有桶的k_max最大值取平均值，来估算n
$$
n \approx 2^{(k\_max1+k\_max2+...+k\_maxn)/n}
$$
LogLog算法采用的是平均值来计算k_max,但缺点是受某些极值的影响比较大，比如k_max1=100, k_max2=1,平均下来就是50.5，这样的统计信息仍然不准确。

HyperLogLog算法在LogLog算法的基础上，将几何平均值改为了调和平均值。从而将估算的准确率提高到了最高的水平。
$$
n \approx 2^{n/(1/k_max1 + 1/k_max2+1/k_maxn)}
$$


## RoaringBitmap

RoaringBitmap表示压缩位图索引。在这之前我们先得了解什么是Bitmap。Bitmap索引经常被用在数据库和搜索引擎中。通过利用位并行运算的优势，它能够显著地加速查询。但是，它也有一个缺点，那就是会耗费更多的内存，因此就有了压缩的BitMap索引。在Bitmap压缩方案中，Roaring bitmaps基于RLE的压缩性能更好。

## SkipList

跳表，Btree+的简单实现。是一个有序的链表。跳表的查询的时间复杂度为 O(logn）插入和删除的时间复杂度也为 O(logn)。

## BloomFilter

*Bloom Filter*是由Bloom在1970年提出的一种空间效率高的概率型数据结构。它可以用来查找某一个元素是否存在于集合中。

BloomFilter有几个关键的参数。

`m`代表由m个bit组成的位数组，初始化全部为0。

`k`代表有k个哈希函数，每个哈希函数的作用是将集合中的某个元素哈希映射到位数组的某一个位置，并将对应的bit位置为1。

BloomFilter有两个主要的方法：插入和查找。

插入：将要插入的一个元素经过哈希函数映射到位数组的某一个bit位，将对应的bit位置为1。所以k个哈希函数可以将位数组的k个bit位置为1。

查找：将要查找的元素经过k次哈希函数计算后找到位数组的k个bit位，如果对应的bit位都为1，则表示该元素可能存在于集合中。而如果有一个bit位不为1，则表示该元素一定不在集合中。

从以上的介绍中可以发现，BloomFilter可能存在误判，也就是说某一个元素可能不存在集合中，但其对应的bit位都为1.

衡量BloomFilter的误判率称为false positives。 false positives可以通过以下公式计算得到
$$
\epsilon = (1-(1-1/m)^{kn})^{k} \approx(1-e^{-kn/m})^{k}
$$
n表示插入的元素的个数。

由此我们可以得出最优的散列函数的个数k可以通过下面的计算得到
$$
k = m/n ln2
$$
位数组中位的大小m可以计算通过下面的计算得到
$$
m = -nlnP/(ln2)^2
$$
一般我们可以根据误判率来计算得到位数组的大小m，同时得到散列函数的个数k。

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



BloomFilter有多种变种，比如ribbon filters， cuckoo filter， SIMD blocked Bloom filter，Xor Filter等。相比于经典的BloomFilter算法，新的变种在性能上，空间使用上已经有了很大的提升。

## ConcurrentHashMap

线程安全的HashMap实现。
