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

*Bloom Filter*是由Bloom在1970年提出的一种空间效率高的概率型数据结构。它可以用来查找某一个元素是否存在于集合中。BloomFilter有多种变种，比如ribbon filters， cuckoo filter， SIMD blocked Bloom filter，Xor Filter等。

## ConcurrentHashMap

线程安全的HashMap实现。
