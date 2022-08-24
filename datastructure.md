# 数据结构

## hash

Murmur hash, cityhash

### consistent hashing

一致性哈希是指将「存储节点」和「数据」都映射到一个首尾相连的哈希环上，如果增加或者移除一个节点，仅影响该节点在哈希环上顺时针相邻的后继节点，其它数据也不会受到影响。 但是一致性哈希算法不能够均匀的分布节点，会出现大量请求都集中在一个节点的情况，在这种情况下进行容灾与扩容时，容易出现雪崩的连锁反应。

### linear hashing

动态调整hashtable

## HLL

不精确的去重计数

## roaring bit map

压缩位图索引

## skip list

跳表，Btree+的简单实现

## BloomFilter

BloomFilter有多种变种，比如ribbon filters， cuckoo filter， SIMD blocked Bloom filter，Xor Filter等。

## ConcurrentHashMap

线程安全的HashMap
