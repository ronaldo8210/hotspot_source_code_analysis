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

三级存储结构的定义分别为：

1. Region哈希表的


RSet在内部使用Per Region Table(PRT)记录分区的引用情况。

在PRT中将会以三种模式记录引用：稀少、细粒度、粗粒度（粗粒度的PRT只是记录了引用数量，需要通过整堆扫描才能找出所有引用，因此扫描速度也是最慢的）
