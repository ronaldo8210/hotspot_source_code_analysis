YGC期间，Java根对象直接引用的年轻代对象全部处理完后（这些年轻代对象已被复制到survivor区、其自身的引用属性被加入RefToScanQueue队列用于后续的对象深度遍历），开始以老年代对象为根，遍历老年代对象直接引用的年轻代对象。由于老年代内存空间太大，直接遍历老年代对象耗时太长，所以直接从待回收的年轻代的Remembered Set出发，先找到引用者老年代对象所在的card，再找到老年代对象。

一个YGC worker线程处理老年代直接引用的年轻代对象的函数调用链为：

- G1RootProcessor::scan_remembered_sets() --> G1RemSet::oops_into_collection_set_do() --> G1RemSet::scanRS() --> G1CollectedHeap::collection_set_iterate_from() --> ScanRSClosure::doHeapRegion() --> ScanRSClosure::scanCard() --> HeapRegionDCTOC::walk_mem_region() --> G1ParPushHeapRSClosure::do_oop_nv()

各个函数主要代码的解析如下：

G1RemSet::oops_into_collection_set_do()
```c++
void G1RemSet::oops_into_collection_set_do(G1ParPushHeapRSClosure* oc,
                                           CodeBlobClosure* code_root_cl,
                                           uint worker_i) {

  // G1RemSet是全局单例对象，oops_into_collection_set_do()被多个YGC worker线程调用，缓存每个YGC worker线程私有的G1ParPushHeapRSClosure回调对象
  _cset_rs_update_cl[worker_i] = oc;

  // 每个YGC worker线程为什么要用到一个DirtyCardQueue，还没有完全理解
  DirtyCardQueue into_cset_dcq(&_g1->into_cset_dirty_card_queue_set());

  // updateRS()的作用还没有完全理解，因为之前复制年轻代对象后，survivor区的Remembered Set已做了同步的更新，暂不理解这里的updateRS()的目的
  updateRS(&into_cset_dcq, worker_i);
  // 开始扫描待回收年轻代的Remembered Set，搜索引用了待回收年轻代内的对象的老年代对象
  scanRS(oc, code_root_cl, worker_i);
}
```

G1RemSet::scanRS()
```c++
void G1RemSet::scanRS(G1ParPushHeapRSClosure* oc,
                      CodeBlobClosure* code_root_cl,
                      uint worker_i) {
  double rs_time_start = os::elapsedTime();
  // 每一个YGC worker线程取得一个待回收的年轻代内存空间对应的HeapRegion对象
  // 不会有多个YGC worker线程处理同一个年轻代的情况
  HeapRegion *startRegion = _g1->start_cset_region_for_worker(worker_i);

  // 遍历一个年轻代的Remembered Set时调用的回调对象
  ScanRSClosure scanRScl(oc, code_root_cl, worker_i);

  // 从startRegion开始处理待回收年轻代的Remembered Set，直到CSet中的年轻代全部被处理完
  _g1->collection_set_iterate_from(startRegion, &scanRScl);
  scanRScl.set_try_claimed();
  // 暂不理解为什么再次调用collection_set_iterate_from()
  _g1->collection_set_iterate_from(startRegion, &scanRScl);

  // 计算本YGC worker线程扫描Remembered Set的耗时
  double scan_rs_time_sec = (os::elapsedTime() - rs_time_start)
                            - scanRScl.strong_code_root_scan_time_sec();

  assert(_cards_scanned != NULL, "invariant");
  // 统计本YGC worker线程处理的老年代card的个数
  _cards_scanned[worker_i] = scanRScl.cards_done();

  _g1p->phase_times()->record_time_secs(G1GCPhaseTimes::ScanRS, worker_i, scan_rs_time_sec);
  _g1p->phase_times()->record_time_secs(G1GCPhaseTimes::CodeRoots, worker_i, scanRScl.strong_code_root_scan_time_sec());
}
```

G1CollectedHeap::collection_set_iterate_from()
```c++
void G1CollectedHeap::collection_set_iterate_from(HeapRegion* r,
                                                  HeapRegionClosure *cl) {
  if (r == NULL) {
    // The CSet is empty so there's nothing to do.
    return;
  }

  // 正在处理的HeapRegion对象对应的代际空间必须是待回收的内存空间
  assert(r->in_collection_set(),
         "Start region must be a member of the collection set.");
  HeapRegion* cur = r;
  while (cur != NULL) {
    HeapRegion* next = cur->next_in_collection_set();
    // 调用ScanRSClosure对象的doHeapRegion()函数
    if (cl->doHeapRegion(cur) && false) {
      cl->incomplete();
      return;
    }
    // 一个待回收年轻代的Remembered Set遍历完成，再处理下一个年轻代
    cur = next;
  }
  // 暂不理解为什么又重新执行一遍上述的操作
  cur = g1_policy()->collection_set();
  while (cur != r) {
    HeapRegion* next = cur->next_in_collection_set();
    if (cl->doHeapRegion(cur) && false) {
      cl->incomplete();
      return;
    }
    cur = next;
  }
}
```

ScanRSClosure::doHeapRegion() & ScanRSClosure::scanCard()
```c++
class ScanRSClosure : public HeapRegionClosure {

public:

  void scanCard(size_t index, HeapRegion *r) {
    HeapRegionDCTOC cl(_g1h, r, _oc,
                       CardTableModRefBS::Precise);

    // Set the "from" region in the closure.
    _oc->set_region(r);
    // _bot_shared->address_for_index(index)得到的是card的内存起始地址
    // G1BlockOffsetSharedArray::N_words指一个card占据的内存字节数
    // card_region指card的内存地址范围
    MemRegion card_region(_bot_shared->address_for_index(index), G1BlockOffsetSharedArray::N_words);
    // r->bottom()是card所属Region的内存区的底部地址
    // r->scan_top()是card所属Region的内存区的已分配对象的顶部地址
    MemRegion pre_gc_allocated(r->bottom(), r->scan_top());
    // 防止计算出的card_region内存越界
    MemRegion mr = pre_gc_allocated.intersection(card_region);
    if (!mr.is_empty() && !_ct_bs->is_card_claimed(index)) {
      _ct_bs->set_card_claimed(index);
      _cards_done++;
      // 遍历card中的所有对象
      cl.do_MemRegion(mr);
    }
  }

  bool doHeapRegion(HeapRegion* r) {
    assert(r->in_collection_set(), "should only be called on elements of CS.");
    // hrrs指向待回收的年轻代的Remembered Set对象
    HeapRegionRemSet* hrrs = r->rem_set();
    if (hrrs->iter_is_complete()) return false; // All done.
    if (!_try_claimed && !hrrs->claim_iter()) return false;
    
    // 暂不理解为什么需要将年轻代HeapRegion加入到全局的_dirty_cards_region_list队列
    _g1h->push_dirty_cards_region(r);

    HeapRegionRemSetIterator iter(hrrs);
    // card_index是指Remembered Set中记录的老年代card在全局卡表中的索引号
    size_t card_index;

    size_t jump_to_card = hrrs->iter_claimed_next(_block_size);
    for (size_t current_card = 0; iter.has_next(card_index); current_card++) {
      // 在循环中不断调用iter.has_next(card_index)，完成Remembered Set的遍历
      // 每一次has_next()返回时，card_index都存储着遍历到的新的老年代card在全局卡表中的索引号
      
      if (current_card >= jump_to_card + _block_size) {
        jump_to_card = hrrs->iter_claimed_next(_block_size);
      }
      if (current_card < jump_to_card) continue;
      // 取得老年代card的内存起始地址
      HeapWord* card_start = _g1h->bot_shared()->address_for_index(card_index);

      // 取得老年代card所属Region内存区域对应的HeapRegion对象
      HeapRegion* card_region = _g1h->heap_region_containing(card_start);
      _cards++;

      if (!card_region->is_on_dirty_cards_region_list()) {
        _g1h->push_dirty_cards_region(card_region);
      }

      // If the card is dirty, then we will scan it during updateRS.
      // card_region肯定不能是待回收的代际空间
      if (!card_region->in_collection_set() &&
          !_ct_bs->is_card_dirty(card_index)) {
        // 扫描card中的老年代对象  
        scanCard(card_index, card_region);
      }
    }
    if (!_try_claimed) {
      // Scan the strong code root list attached to the current region
      scan_strong_code_roots(r);

      hrrs->set_iter_complete();
    }
    return false;
  }
};
```

HeapRegionDCTOC::walk_mem_region()
```c++
void HeapRegionDCTOC::walk_mem_region(MemRegion mr,
                                      HeapWord* bottom,
                                      HeapWord* top) {
  // 遍历card中的所有对象，对每个对象的引用属性调用G1ParPushHeapRSClosure回调对象
  
  G1CollectedHeap* g1h = _g1;
  size_t oop_size;
  HeapWord* cur = bottom;

  // Start filtering what we add to the remembered set. If the object is
  // not considered dead, either because it is marked (in the mark bitmap)
  // or it was allocated after marking finished, then we add it. Otherwise
  // we can safely ignore the object.
  if (!g1h->is_obj_dead(oop(cur), _hr)) {
    oop_size = oop(cur)->oop_iterate(_rs_scan, mr);
  } else {
    oop_size = _hr->block_size(cur);
  }

  cur += oop_size;

  if (cur < top) {
    oop cur_oop = oop(cur);
    oop_size = _hr->block_size(cur);
    HeapWord* next_obj = cur + oop_size;
    while (next_obj < top) {
      // Keep filtering the remembered set.
      if (!g1h->is_obj_dead(cur_oop, _hr)) {
        // Bottom lies entirely below top, so we can call the
        // non-memRegion version of oop_iterate below.
        cur_oop->oop_iterate(_rs_scan);
      }
      cur = next_obj;
      cur_oop = oop(cur);
      oop_size = _hr->block_size(cur);
      next_obj = cur + oop_size;
    }

    // Last object. Need to do dead-obj filtering here too.
    if (!g1h->is_obj_dead(oop(cur), _hr)) {
      oop(cur)->oop_iterate(_rs_scan, mr);
    }
  }
}
```

G1ParPushHeapRSClosure::do_oop_nv()
```c++
template <class T>
inline void G1ParPushHeapRSClosure::do_oop_nv(T* p) {
  T heap_oop = oopDesc::load_heap_oop(p);

  if (!oopDesc::is_null(heap_oop)) {
    oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);
    if (_g1->is_in_cset_or_humongous(obj)) {
      Prefetch::write(obj->mark_addr(), 0);
      Prefetch::read(obj->mark_addr(), (HeapWordSize*2));

      // 将引用放入RefToScanQueue队列，用于后续的对象深度遍历
      _par_scan_state->push_on_queue(p);
    } else {
      assert(!_g1->obj_in_cs(obj), "checking");
    }
  }
}
```
