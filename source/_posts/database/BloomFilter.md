---
title: LevelDB 源码分析(二) - BloomFilter
data: 2024-05-1
cover: /img/blog/blog7.jpg
categories:
- database
tags:
- database
- leveldb
---

## 背景

Bloom Filter 是一种空间效率很高的随机数据结构，它利用位数组表示一个集合，并能判断一个元素是否属于这个集合。

Bloom Filter 的这种高效是有一定代价的：在判断一个元素是否属于某个集合时，有可能会把不属于这个集合的元素误认为属于这个集合（false positive）。

leveldb 中利用布隆过滤器判断指定的 key 值是否存在于 sstable 中，若过滤器表示不存在，则该 key 一定不存在，由此加快了查找的效率。

<!--more-->

## 优劣

相对于其他表示数据集的数据结构，如平衡二叉搜索树、Trie 树、哈希表，甚至更简单的数组或者链表。大都需要对数据项本身存储。但是 Bloom Filter 并不存储数据项本身，而是用高效紧凑的位数组来存。

如此设计，使得 Bloom Filter 的大小与数据项本身大小（如字符串的长短）无关。可以理解为在哈希表基础上，忽略了冲突处理，从而省下了额外开销。

对于 Bloom Filter ，其误判率应该和以下参数有关：

- 哈希函数的个数 k
- 底层位数组的长度 m
- 数据集大小 n

当 k = ln2 * (m/n) 时，Bloom Filter 获取最优的准确率。m/n 即 bits per key（集合中每个 key 平均分到的 bit 数）。

## 模块分析

首先是 LevelDB 中的过滤器策略 FilterPolicy。这是一个基类，可以自定义不同的过滤策略。

```cpp
#include <string>

#include "leveldb/export.h"

namespace leveldb {

class Slice;

class LEVELDB_EXPORT FilterPolicy {
 public:
  virtual ~FilterPolicy();

  // 过滤策略的名字，用来唯一标识该 Filter 持久化、载入内存时的编码方法。
  virtual const char* Name() const = 0;

  // 给长度为 n 的 keys 集合创建一个过滤策略, 位数组位于 dst 中
  virtual void CreateFilter(const Slice* keys, int n,
                            std::string* dst) const = 0;

  // 过滤函数，位数组位于 filter 中
  virtual bool KeyMayMatch(const Slice& key, const Slice& filter) const = 0;
};

LEVELDB_EXPORT const FilterPolicy* NewBloomFilterPolicy(int bits_per_key);

}  // namespace leveldb
```

对于布隆过滤器，如果 Key 在 filter 里，那么一定会 Match 上；反之如果不在，那么有小概率也会 Match 上，进而会多做一些磁盘访问，但是概率小的情况下影响不是很大。

这里的声明了一个 NewBloomFilterPolicy 函数，这样的设计可以让使用者自行定义策略类。这种设计也称之为策略模式，将策略单独设计为一个类或接口，不同的子类对应不同的策略方法。

而 LevelDB 提供了实现了该接口的一个内置的过滤策略：BloomFilterPolicy

```cpp
class BloomFilterPolicy : public FilterPolicy {
 public:
  explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
    // 根据上面提到的结论：k = ln2 * (m/n)，获取哈希函数的个数 k。
    // 并对该个数做一定的限制
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }

  const char* Name() const override { return "leveldb.BuiltinBloomFilter2"; }

  void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
    // n 为 key 的个数， bits_per_key_ 是前面提到的集合中每个 key 平均分到的 bit 数
    size_t bits = n * bits_per_key_;
    if (bits < 64) bits = 64; // 如果数组太短，会有很高的误判率。
    size_t bytes = (bits + 7) / 8;  // 向上取整，之后在调整为 8 的倍数 (因为字符数组用 char 来表示)
    bits = bytes * 8;

    const size_t init_size = dst->size();
    dst->resize(init_size + bytes, 0);  // 在原有的基础上扩增计算出的 bits 大小
    dst->push_back(static_cast<char>(k_));  // 记下哈希函数的个数
    char* array = &(*dst)[init_size];   // 指针指向扩增的起始点
    for (int i = 0; i < n; i++) {
      // 使用 double-hashing 方法，仅使用一个 hash 函数来生成 k 个 hash 值，近似等价于使用 k 个哈希函数的效果
      uint32_t h = BloomHash(keys[i]);
      const uint32_t delta = (h >> 17) | (h << 15);  // 循环右移 17 bits 作为步长
      for (size_t j = 0; j < k_; j++) {
        const uint32_t bitpos = h % bits;
        array[bitpos / 8] |= (1 << (bitpos % 8));
        h += delta;
      }
    }
  }

  bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const override {
    const size_t len = bloom_filter.size();
    if (len < 2) return false;

    const char* array = bloom_filter.data();
    const size_t bits = (len - 1) * 8; // -1 是去掉 k 所占空间

    // 取出创建 Filter 时保存的哈希函数个数 k
    const size_t k = array[len - 1];
    if (k > 30) {
      // 超过我们设定 k 个数，直接返回 true，不滤掉该 SSTable.
      return true;
    }

    uint32_t h = BloomHash(key);
    const uint32_t delta = (h >> 17) | (h << 15);  // 循环右移 17 bits 作为步长
    for (size_t j = 0; j < k; j++) {
      const uint32_t bitpos = h % bits;
      if ((array[bitpos / 8] & (1 << (bitpos % 8))) == 0) return false;
      h += delta;
    }
    return true;
  }

 private:
  size_t bits_per_key_;
  size_t k_;
};
```

在这里 BloomFilter 采用了 Double Hash 做开放寻址哈希的优化，详见 [哈希冲突的解决方式](../skills/Hash.md)

而 KeyMayMatch 是 CreateFilter 的逆过程。

## 小结

Bloom Filter 通常用于快速判断某个元素是否在集合中。其本质上实通过容忍一定的错误率，来换取时空的高效性。从实现角度来理解，是在哈希表的基础上省下了冲突处理部分，并通过 k 个独立哈希函数来减少误判，LevelDB 在实现时使用了某种优化：利用一个哈希函数来达到近似 k 个哈希函数的效果。