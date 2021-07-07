[OtherRegionsTable数据结构](#jump_1)

[常用原子操作](#jump_2)

[SparsePRT数据结构](#jump_3)

[]


## <span id="jump_1">OtherRegionsTable数据结构</span>
在hotspot源码中，RSet整体数据结构的定义为HeapRegionRemSet类，RSet的三级存储结构以及对RSet的操作都封装在HeapRegionRemSet的成员变量OtherRegionsTable类实例中：
```c++
class HeapRegionRemSet : public CHeapObj<mtGC> {
private:
  OtherRegionsTable _other_regions;
}
```

OtherRegionsTable类的主要成员变量有：
```c++
class OtherRegionsTable VALUE_OBJ_CLASS_SPEC {
  // 指向OtherRegionsTable所属的HeapRegion实例的指针，所有引用了_hr所指的HeapRegion内存区中对象的其他对象所在的Card或HeapRegion，都在本OtherRegionsTable中被记录。
  HeapRegion*      _hr;
  Mutex*           _m;

  BitMap      _coarse_map;  // 如果
  size_t      _n_coarse_entries;
  static jint _n_coarsenings;

  PerRegionTable** _fine_grain_regions;
  size_t           _n_fine_entries;

  PerRegionTable * _first_all_fine_prts;
  PerRegionTable * _last_all_fine_prts;

  // Used to sample a subset of the fine grain PRTs to determine which
  // PRT to evict and coarsen.
  size_t        _fine_eviction_start;
  static size_t _fine_eviction_stride;
  static size_t _fine_eviction_sample_size;

  SparsePRT   _sparse_table;

  // These are static after init.
  static size_t _max_fine_entries;
  static size_t _mod_max_fine_entries_mask;
};
```


## <span id="jump_2">常用原子操作</span>
使用到的CAS原子操作

```c++
// 判断dest是否等于compare_value，如果等于，将dest赋值为exchange_value，并返回dest的原值；
// 如果不等于，什么都不做，返回dest的值
inline void* Atomic::cmpxchg_ptr(void* exchange_value, volatile void* dest, void* compare_value) {
  // cmpxchg内部为汇编实现
  return (void*)cmpxchg((jlong)exchange_value, (volatile jlong*)dest, (jlong)compare_value);
}
```


三级存储的数据结构分别为：


## <span id="jump_3">SparsePRT数据结构</span>
SparsePRT类主要的成员变量有：
```c++
class SparsePRT VALUE_OBJ_CLASS_SPEC {
  RSHashTable* _cur;
  RSHashTable* _next;

  HeapRegion* _hr;  // 存储引用者Java对象的HeapRegion实例的地址

  // 一个SparsePRT对象初始化时，内部_next指向的第一个RSHashTable对象的存储容量
  enum SomeAdditionalPrivateConstants {
    InitialCapacity = 16
  };

  SparsePRT* _next_expanded;

  // 静态变量，全局唯一的SparsePRT对象列表，各个RSet的执行过expand()操作的SparsePRT对象都被加入该链表中
  static SparsePRT* _head_expanded_list;
};
```

SparsePRT类主要的成员函数的源码解析：

构造函数：
```c++
SparsePRT::SparsePRT(HeapRegion* hr) :
  _hr(hr), _expanded(false), _next_expanded(NULL)  // 新建的SparsePRT对象不在_head_expanded_list链表中，_expanded初始化为false
{
  // 新建第一个RSHashTable对象，_cur和_next都指向该RSHashTable对象
  _cur = new RSHashTable(InitialCapacity);
  _next = _cur;
}
```

将SparsePRT对象加入到_head_expanded_list链表：
```c++
// 会被多线程调用，需要CAS来做多线程同步
void SparsePRT::add_to_expanded_list(SparsePRT* sprt) {
  // 已经执行过expand()的SparsePRT对象肯定已经在_head_expanded_list链表中，直接返回
  if (sprt->expanded()) return;
  sprt->set_expanded(true);
  SparsePRT* hd = _head_expanded_list;
  while (true) {
    // 将sprt所指的SparsePRT对象的_next_expanded指针指向链表头节点
    sprt->_next_expanded = hd;
    // CAS原子操作，判断当前的链表头节点的地址是否与之前看到的头节点地址相等，
    // 1、如果相等，说明链表没有被其他线程更改过，将_head_expanded_list指针的值赋值为sprt，即sprt所指的SparsePRT对象成为链表头节点，
    //    cmpxchg_ptr返回时res赋值为链表原先头节点的地址
    // 2、如果不等，说明链表已被其他线程更改，什么也不做，cmpxchg_ptr返回时res赋值为链表当前最新头节点的地址
    SparsePRT* res =
      (SparsePRT*)
      Atomic::cmpxchg_ptr(sprt, &_head_expanded_list, hd);
    // 将sprt所指的SparsePRT对象成功插入到链表的最前端，返回  
    if (res == hd) return;
    // 更新hd指针为链表当前最新头节点的地址，下一轮while循环内继续尝试将将sprt所指的SparsePRT对象插入到链表的最前端
    else hd = res;
  }
}
```

从_head_expanded_list链表中摘取最靠前的一个SparsePRT对象：
```c++
// 会被多线程调用，需要CAS来做多线程同步
SparsePRT* SparsePRT::get_from_expanded_list() {
  SparsePRT* hd = _head_expanded_list;
  while (hd != NULL) {
    SparsePRT* next = hd->next_expanded();
    SparsePRT* res =
      (SparsePRT*)
      Atomic::cmpxchg_ptr(next, &_head_expanded_list, hd);
    if (res == hd) {
      hd->set_next_expanded(NULL);
      return hd;
    } else {
      hd = res;
    }
  }
  return NULL;
}
```

将SparsePRT的存储量加大一倍：
```c++
void SparsePRT::expand() {
  RSHashTable* last = _next;
  _next = new RSHashTable(last->capacity() * 2);

#if SPARSE_PRT_VERBOSE
  gclog_or_tty->print_cr("  Expanded sparse table for %u to %d.",
                         _hr->hrm_index(), _next->capacity());
#endif
  for (size_t i = 0; i < last->capacity(); i++) {
    SparsePRTEntry* e = last->entry((int)i);
    if (e->valid_entry()) {
#if SPARSE_PRT_VERBOSE
      gclog_or_tty->print_cr("    During expansion, transferred entry for %d.",
                    e->r_ind());
#endif
      _next->add_entry(e);
    }
  }
  if (last != _cur) {
    delete last;
  }
  add_to_expanded_list(this);
}
```

RSHashTable类主要的成员变量有：
```c++
class RSHashTable : public CHeapObj<mtGC> {
  size_t _capacity;
  size_t _capacity_mask;
  size_t _occupied_entries;  // 存储的
  size_t _occupied_cards;  // 存储的卡表索引号的数量

  SparsePRTEntry* _entries;
  int* _buckets;
  int  _free_region;
  int  _free_list;
};
```

