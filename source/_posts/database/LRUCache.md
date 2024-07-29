---
title: LevelDB 源码分析(一) - LRUCache
data: 2024-04-29
cover: /img/blog/blog8.jpg
categories:
- database
tags:
- database
- leveldb
---

## 背景

LRU （Last Recent Used）是一个重要的缓存替换算法，常用于缓存场景,一般的实现是使用一个哈希表（unordered_map）和一个双向链表，哈希表解决索引问题，双向链表维护访问顺序。

<!--more-->

## 内部实现

LevelDB的LRUCache设计有4个数据结构，是依次递进的关系，分别是：
- LRUHandle
- HandleTable
- LRUCache
- ShardedLRUCache

LRUHandle 是 LRUCache 中的数据节点，HandleTable 是 LRUCache 为了实现可扩展哈希性而自定义的哈希桶，LRUCache 便是核心类，内部维护了一条冷数据双向链表，一条热数据双向链表，以及 HandleTable来索引节点，此时LRU的缓存管理数据结构已经实现。而 ShardedLRUCache 是为了在多线程访问时提升效率，内部有16个LRUCache，查找key的时候，先计算属于哪一个LRUCache，然后在相应的LRUCache中上锁查找。

## 模块分析

首先是 LevelDB 的导出接口 Cache

```cpp
/**
 * Handle 仅作为指针类型使用，可以理解为 void *
 * 优势在于这是一个强类型，同时增强语意。
 */
struct Handle {};

/**
 * 插入一个键值对（key，value）到缓存（cache）中
 * 并从缓存总容量中减去该键值对所占额度（charge） 
 * 
 * 返回指向该键值对的句柄（handle）
 * 
 * Slice 是自定义的字符串类型，相当于 go 里面的切片
 * 本质是封装的 const char *
 *
 * 在键值对不再被使用时，键值对会被传入的 deleter 参数释放
 */
virtual Handle* Insert(const Slice& key, void* value, size_t charge,
                       void (*deleter)(const Slice& key, void* value)) = 0;

/**
 * 如果缓存中没有相应键（key），则返回 nullptr
 */
virtual Handle* Lookup(const Slice& key) = 0;

/**
 * 释放 Insert/Lookup 函数返回的句柄
 */
virtual void Release(Handle* handle) = 0;

/**
 * 获取句柄中的值，类型为 void*（表示任意用户自定义类型）
 */
virtual void* Value(Handle* handle) = 0;

/**
 * 如果缓存中包含给定键所指向的条目，则删除之。
 * 只有在所有持有该条目句柄都释放时，该条目所占空间才会真正被释放
 */
virtual void Erase(const Slice& key) = 0;

/**
 * 返回一个自增的数值 id。当一个缓存实例由多个客户端共享时，
 */
virtual uint64_t NewId() = 0;

/**
 * 驱逐全部没有被使用的数据条目
 * 利用此接口定期释放内存。
 */
virtual void Prune() {}

/**
 * 返回当前缓存中所有数据条目所占容量总和的一个预估
 */
virtual size_t TotalCharge() const = 0;
```

LRUCahce 的实现依靠双向环形链表和哈希表。其中双向环形链表维护 Recently 属性，哈希表维护 Used 属性。双向环形链表和哈希表的节点信息都存储于 LRUHandle 结构中：

```cpp
struct LRUHandle {
  void* value;
  void (*deleter)(const Slice&, void* value);
  LRUHandle* next_hash;
  LRUHandle* next;
  LRUHandle* prev;
  size_t charge;  // TODO(opt): Only allow uint32_t?
  size_t key_length;
  bool in_cache;     // Whether entry is in the cache.
  uint32_t refs;     // References, including cache reference, if present.
  uint32_t hash;     // Hash of key(); used for fast sharding and comparisons
  char key_data[1];  // Beginning of key

  Slice key() const {
    // next_ is only equal to this if the LRU handle is the list head of an
    // empty list. List heads never have meaningful keys.
    assert(next != this);

    return Slice(key_data, key_length);
  }
};
```
依次解释每一项属性：

- value 为缓存存储的数据，类型无关；
- deleter 为键值对的析构函数指针；
- next_hash 为开放式哈希表中同一个桶下存储链表时使用的指针；
- next 和 prev 自然是双向环形链接的前后指针；
- charge 为当前节点的缓存费用，比如一个字符串的费用可能就是它的长度；
- key_length 为 key 的长度；
- in_cache 为节点是否在缓存里的标志；
- refs 为引用计数，当计数为 0 时则可以用 deleter 清理掉；
- hash 为 key 的哈希值；
- key_data 为变长的 key 数据，malloc 时动态指定长度。

这里的 key_data[1] 其实用到了地址溢出的特性，并不是说 key 只有 1 字节，更多的是标记地址，从此处获取真正的 key 值，所以需要 key_length 字段 来记录 key 大小。

具体的大小在 LRUCache 的 insert 函数中通过 key.size()的方式确定。
```cpp
 LRUHandle* e =
      reinterpret_cast<LRUHandle*>(malloc(sizeof(LRUHandle) - 1 + key.size()));
```

接着看哈希表的实现：

```cpp
class HandleTable {
 public:
  
 ...
 
 private:

  uint32_t length_;
  uint32_t elems_;
  LRUHandle** list_;
}
```
依次解释每一项属性：
- length_ 纪录的就是当前hash桶的桶个数
- elems_ 维护在整个hash表中一共存放了多少个元素
- list_ 二维指针，每一个指针指向一个桶的表头位置。

为了提升查找效率，以及尽量减少哈希冲突，所以采用可扩展哈希来进行操作

```cpp
LRUHandle** FindPointer(const Slice& key, uint32_t hash) {
    LRUHandle** ptr = &list_[hash & (length_ - 1)];
    while (*ptr != nullptr && ((*ptr)->hash != hash || key != (*ptr)->key())) {
      ptr = &(*ptr)->next_hash;
    }
    return ptr;
  }
```
HandleTable 的找寻节点是通过调用 FindPointer 函数，先通过 hash 找到对应的桶，再通过链接法遍历该桶处的链表，找寻对应的节点，如果找不到而返回 nullptr。

```cpp
  LRUHandle* Insert(LRUHandle* h) {
    LRUHandle** ptr = FindPointer(h->key(), h->hash);
    LRUHandle* old = *ptr;  //老的元素返回，LRUCache会将相同key的老元素释放。
    h->next_hash = (old == NULL ? NULL : old->next_hash);
    *ptr = h;
    if (old == NULL) {
      ++elems_;
      if (elems_ > length_) {
        // Since each cache entry is fairly large, we aim for a small
        // average linked list length (<= 1).
        Resize();
      }
    }
    return old;
  }
```

当hash桶中元素的个数超过桶的的个数的时候，调用Resize函数，将桶的个数增加一倍，调整原来hash表的布局。正是这种提早扩大桶的个数，良好的hash函数会减少哈希冲突。

```cpp
  void Resize() {
    uint32_t new_length = 4;
    while (new_length < elems_) {
      new_length *= 2;
    }
    LRUHandle** new_list = new LRUHandle*[new_length];
    memset(new_list, 0, sizeof(new_list[0]) * new_length);
    uint32_t count = 0;
    for (uint32_t i = 0; i < length_; i++) {
      LRUHandle* h = list_[i];
      while (h != NULL) {
        LRUHandle* next = h->next_hash;
        uint32_t hash = h->hash;
        LRUHandle** ptr = &new_list[hash & (new_length - 1)]; //各个已有的元素重新计算，应该落在哪个桶的链表中。
        h->next_hash = *ptr;
        *ptr = h;
        h = next;
        count++;
      }
    }
    assert(elems_ == count);
    delete[] list_;
    list_ = new_list;
    length_ = new_length;
  }
```

所以在 LRUCache 中不仅需要哈希表来快速查找，还需要链表能够快速插入和删除。

核心是下面几个变量：

```cpp
class LRUCache {
 public:

 ...
 
 private:

  ...

  // lru_ 是冷链表
  LRUHandle lru_ GUARDED_BY(mutex_);

  //in_use_ 属于热链表, 即经常访问的数据位于此处
  LRUHandle in_use_ GUARDED_BY(mutex_);

  // 哈希桶
  HandleTable table_ GUARDED_BY(mutex_);
}
```

GUARDED_BY(mutex_) 可以不用管，主要作用是确保在访问这些数据时，持有对应的锁。

Ref 函数表示要使用该cache，因此如果对应元素位于冷链表，需要将它从冷链表溢出，链入到热链表：

```cpp
void LRUCache::Ref(LRUHandle* e) {
  if (e->refs == 1 && e->in_cache) {  // If on lru_ list, move to in_use_ list.
    LRU_Remove(e);
    LRU_Append(&in_use_, e);
  }
  e->refs++;
}

void LRUCache::LRU_Remove(LRUHandle* e) {
  e->next->prev = e->prev;
  e->prev->next = e->next;
}

void LRUCache::LRU_Append(LRUHandle* list, LRUHandle* e) {
  // Make "e" newest entry by inserting just before *list
  e->next = list;
  e->prev = list->prev;
  e->prev->next = e;
  e->next->prev = e;
}

```

Unref 与 Ref 对应，表示当前对象放弃访问该元素，将引用计数－－，如果引用计数为0，则删除这个元素，如果引用计数为1，则可以将元素从热链表删除，放入到冷链表：

```cpp
void LRUCache::Unref(LRUHandle* e) {
  assert(e->refs > 0);
  e->refs--;
  if (e->refs == 0) {
    assert(!e->in_cache);
    (*e->deleter)(e->key(), e->value); //元素的deleter函数，此时回调。
    free(e);
  } else if (e->in_cache && e->refs == 1) {  // 移入冷链表 lru_
    LRU_Remove(e);
    LRU_Append(&lru_, e);
  }
}
```

所以在插入一个元素时，必须考虑缓存的容量，超过了容量，缓存必须要移出某些元素，即就是从冷链表中删除。

```cpp
size_t capacity_; // LRUCache的容量
size_t usage_;    // 当前使用的容量


Cache::Handle* LRUCache::Insert(
    const Slice& key, uint32_t hash, void* value, size_t charge,
    void (*deleter)(const Slice& key, void* value)) {
  MutexLock l(&mutex_);

  LRUHandle* e = reinterpret_cast<LRUHandle*>(
      malloc(sizeof(LRUHandle)-1 + key.size()));
  e->value = value;
  e->deleter = deleter;
  e->charge = charge;
  e->key_length = key.size();
  e->hash = hash;
  e->in_cache = false;
  e->refs = 1;
  memcpy(e->key_data, key.data(), key.size());

  if (capacity_ > 0) {
    e->refs++;  // for the cache's reference.
    e->in_cache = true;
    LRU_Append(&in_use_, e);  //链入热链表
    usage_ += charge;         //使用的容量增加
    FinishErase(table_.Insert(e));  // 如果是更新的话，需要回收老的元素
  }

  while (usage_ > capacity_ && lru_.next != &lru_) { 
   //如果容量超过了设计的容量，并且冷链表中有内容，则从冷链表中删除所有元素
    LRUHandle* old = lru_.next;
    assert(old->refs == 1);
    bool erased = FinishErase(table_.Remove(old->key(), old->hash));
    if (!erased) {
      assert(erased);
    }
  }

  return reinterpret_cast<Cache::Handle*>(e);
}

bool LRUCache::FinishErase(LRUHandle* e) {
  if (e != NULL) {
    assert(e->in_cache);
    LRU_Remove(e);
    e->in_cache = false;
    usage_ -= e->charge;
    Unref(e);
  }
  return e != NULL;
}

```

这里的 FinishErase 有什么作用呢？

其实是 LRU 的 Insert 函数内部隐含了更新的操作，会将新的Node加入到Cache中，而老的元素会调用FinishErase函数来决定是移入冷链表还是彻底删除。

最后是分片 ShardedLRUCache 的实现：

```cpp
static const int kNumShardBits = 4;
static const int kNumShards = 1 << kNumShardBits;

class ShardedLRUCache : public Cache {
 private:
  LRUCache shard_[kNumShards];
  port::Mutex id_mutex_;
  uint64_t last_id_;

  static inline uint32_t HashSlice(const Slice& s) {
    return Hash(s.data(), s.size(), 0);
  }

  static uint32_t Shard(uint32_t hash) { return hash >> (32 - kNumShardBits); }

 public:
  explicit ShardedLRUCache(size_t capacity) : last_id_(0) {
    const size_t per_shard = (capacity + (kNumShards - 1)) / kNumShards;
    for (int s = 0; s < kNumShards; s++) {
      shard_[s].SetCapacity(per_shard);
    }
  }
  ~ShardedLRUCache() override {}
  Handle* Insert(const Slice& key, void* value, size_t charge,
                 void (*deleter)(const Slice& key, void* value)) override {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Insert(key, hash, value, charge, deleter);
  }
  Handle* Lookup(const Slice& key) override {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Lookup(key, hash);
  }
  void Release(Handle* handle) override {
    LRUHandle* h = reinterpret_cast<LRUHandle*>(handle);
    shard_[Shard(h->hash)].Release(handle);
  }
  void Erase(const Slice& key) override {
    const uint32_t hash = HashSlice(key);
    shard_[Shard(hash)].Erase(key, hash);
  }
  void* Value(Handle* handle) override {
    return reinterpret_cast<LRUHandle*>(handle)->value;
  }
  uint64_t NewId() override {
    MutexLock l(&id_mutex_);
    return ++(last_id_);
  }
  void Prune() override {
    for (int s = 0; s < kNumShards; s++) {
      shard_[s].Prune();
    }
  }
  size_t TotalCharge() const override {
    size_t total = 0;
    for (int s = 0; s < kNumShards; s++) {
      total += shard_[s].TotalCharge();
    }
    return total;
  }
};
```
代码很好理解，就是在 LRUCache 上做的一层封装，先通过 Shard 来分片，找到对应的 LRUCahce, 然后再执行对应的逻辑。

```cpp
Cache* NewLRUCache(size_t capacity) { return new ShardedLRUCache(capacity); }
```
这里的 NewLRUCache 采用了工厂模式，才生成对应的 LRUCache 对象。
