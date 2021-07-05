[RSet在GC过程中的作用](#jump_1)

[RSet数据结构](#jump_2)

[新增引用期间RSet的工作流程](#jump_3)

## <span id="jump_1">RSet在GC过程中的作用</span>
Remembered Set数据结构用于记录老年代分区到年轻代分区的引用，每一个年轻代的HeapRegion对象都有一个指针指向自己的RSet对象。

在YGC过程中，回收的是年轻代中的不再使用的对象，除了以Java线程栈上的引用变量为根去深度遍历年轻代中的对象外，还要以老年代对象为根去遍历年轻代中的对象。但老年代空间远大于年轻代空间，不可能对老年代的所有对象全部扫描一遍，因此使用空间换时间的算法，为每一个年轻代的Region新建一个RSet数据结构，在RSet中记录引用了该年轻代Region中对象的老年代对象所处的card的索引。在YGC过程中，不用去扫描老年代，直接从每个年轻代的RSet出发，就能找到所有引用了年轻代对象的老年代对象（先找到老年代对象所处的card，再从card中寻找引用了年轻代对象的老年代对象），并以这些老年代对象为根再去深度遍历年轻代中的对象。

卡表（card table）的概念：

在G1中，每一个HeapRegion又分为若干card，每一个card占据空间为512字节，并且维护一个全局卡表，即card table，卡表是一个整型数组，每一个int元素都对应一个card，int值表示card的当前状态（脏card表示card中的对象的引用发生了变化），每一个card的地址值除以512就是该card对应的int元素在卡表中的索引号。

Region与卡表的关系如下图所示：

  <img src="../images/card_table.png" width="100%" height="100%"/>
  
下例中，设老年代对象O_obj_1引用年轻代对象Y_obj_1，老年代对象O_obj_2、O_obj_3引用年轻代对象Y_obj_2，则年轻代Region Y_Region_1和Y_Region_2的RSet结构如下图所示：

  <img src="../images/rset_1.png" width="70%" height="70%"/>

## <span id="jump_2">RSet数据结构</span>


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




RSet在内部使用Per Region Table(PRT)记录分区的引用情况。

在PRT中将会以三种模式记录引用：稀少、细粒度、粗粒度（粗粒度的PRT只是记录了引用数量，需要通过整堆扫描才能找出所有引用，因此扫描速度也是最慢的）

## <span id="jump_3">新增引用期间RSet的工作流程</span>


