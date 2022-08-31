# 数据结构

## hash

crc32c,Murmur hash, cityhash是目前最快的hash函数。由Google提出。

### consistent hashing

一致性哈希是指将「存储节点」和「数据」都映射到一个首尾相连的哈希环上，如果增加或者移除一个节点，仅影响该节点在哈希环上顺时针相邻的后继节点，其它数据也不会受到影响。 但是一致性哈希算法不能够均匀的分布节点，会出现大量请求都集中在一个节点的情况，在这种情况下进行容灾与扩容时，容易出现雪崩的连锁反应。

### linear hashing

动态调整hashtable

## HLL

对于统计去重数据的场景下可以使用hashmap等数据结构，但他们占用内存较大，HyperLogLog是一个不精确的去重计数算法，可以占用很小内存的情况下估算出统计信息。

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
