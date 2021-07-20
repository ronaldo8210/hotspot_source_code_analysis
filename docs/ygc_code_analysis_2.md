YGC期间，Java根对象直接引用的年轻代对象全部处理完后（这些年轻代对象已被复制到survivor区、其自身的引用属性被加入RefToScanQueue队列用于后续的对象深度遍历），开始以老年代对象为根，遍历老年代对象直接引用的年轻代对象。由于老年代内存空间太大，直接遍历老年代对象耗时太长，所以直接从待回收的年轻代的Remembered Set出发，先找到引用者老年代对象所在的card，再找到老年代对象。

一个YGC worker线程处理老年代直接引用的年轻代对象的函数调用链为：

- G1RootProcessor::scan_remembered_sets() --> G1RemSet::oops_into_collection_set_do() --> G1RemSet::scanRS() --> G1CollectedHeap::collection_set_iterate_from() --> ScanRSClosure::doHeapRegion() --> ScanRSClosure::scanCard() --> 

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
  size_t _cards_done, _cards;
  G1CollectedHeap* _g1h;

  G1ParPushHeapRSClosure* _oc;
  CodeBlobClosure* _code_root_cl;

  G1BlockOffsetSharedArray* _bot_shared;
  G1SATBCardTableModRefBS *_ct_bs;

  double _strong_code_root_scan_time_sec;
  uint   _worker_i;
  int    _block_size;
  bool   _try_claimed;

public:
  ScanRSClosure(G1ParPushHeapRSClosure* oc,
                CodeBlobClosure* code_root_cl,
                uint worker_i) :
    _oc(oc),
    _code_root_cl(code_root_cl),
    _strong_code_root_scan_time_sec(0.0),
    _cards(0),
    _cards_done(0),
    _worker_i(worker_i),
    _try_claimed(false)
  {
    _g1h = G1CollectedHeap::heap();
    _bot_shared = _g1h->bot_shared();
    _ct_bs = _g1h->g1_barrier_set();
    _block_size = MAX2<int>(G1RSetScanBlockSize, 1);
  }

  void set_try_claimed() { _try_claimed = true; }

  void scanCard(size_t index, HeapRegion *r) {
    // Stack allocate the DirtyCardToOopClosure instance
    HeapRegionDCTOC cl(_g1h, r, _oc,
                       CardTableModRefBS::Precise);

    // Set the "from" region in the closure.
    _oc->set_region(r);
    MemRegion card_region(_bot_shared->address_for_index(index), G1BlockOffsetSharedArray::N_words);
    MemRegion pre_gc_allocated(r->bottom(), r->scan_top());
    MemRegion mr = pre_gc_allocated.intersection(card_region);
    if (!mr.is_empty() && !_ct_bs->is_card_claimed(index)) {
      // We make the card as "claimed" lazily (so races are possible
      // but they're benign), which reduces the number of duplicate
      // scans (the rsets of the regions in the cset can intersect).
      _ct_bs->set_card_claimed(index);
      _cards_done++;
      cl.do_MemRegion(mr);
    }
  }

  void printCard(HeapRegion* card_region, size_t card_index,
                 HeapWord* card_start) {
    gclog_or_tty->print_cr("T " UINT32_FORMAT " Region [" PTR_FORMAT ", " PTR_FORMAT ") "
                           "RS names card %p: "
                           "[" PTR_FORMAT ", " PTR_FORMAT ")",
                           _worker_i,
                           card_region->bottom(), card_region->end(),
                           card_index,
                           card_start, card_start + G1BlockOffsetSharedArray::N_words);
  }

  void scan_strong_code_roots(HeapRegion* r) {
    double scan_start = os::elapsedTime();
    r->strong_code_roots_do(_code_root_cl);
    _strong_code_root_scan_time_sec += (os::elapsedTime() - scan_start);
  }

  bool doHeapRegion(HeapRegion* r) {
    assert(r->in_collection_set(), "should only be called on elements of CS.");
    HeapRegionRemSet* hrrs = r->rem_set();
    if (hrrs->iter_is_complete()) return false; // All done.
    if (!_try_claimed && !hrrs->claim_iter()) return false;
    // If we ever free the collection set concurrently, we should also
    // clear the card table concurrently therefore we won't need to
    // add regions of the collection set to the dirty cards region.
    _g1h->push_dirty_cards_region(r);
    // If we didn't return above, then
    //   _try_claimed || r->claim_iter()
    // is true: either we're supposed to work on claimed-but-not-complete
    // regions, or we successfully claimed the region.

    HeapRegionRemSetIterator iter(hrrs);
    size_t card_index;

    // We claim cards in block so as to recude the contention. The block size is determined by
    // the G1RSetScanBlockSize parameter.
    size_t jump_to_card = hrrs->iter_claimed_next(_block_size);
    for (size_t current_card = 0; iter.has_next(card_index); current_card++) {
      if (current_card >= jump_to_card + _block_size) {
        jump_to_card = hrrs->iter_claimed_next(_block_size);
      }
      if (current_card < jump_to_card) continue;
      HeapWord* card_start = _g1h->bot_shared()->address_for_index(card_index);
#if 0
      gclog_or_tty->print("Rem set iteration yielded card [" PTR_FORMAT ", " PTR_FORMAT ").\n",
                          card_start, card_start + CardTableModRefBS::card_size_in_words);
#endif

      HeapRegion* card_region = _g1h->heap_region_containing(card_start);
      _cards++;

      if (!card_region->is_on_dirty_cards_region_list()) {
        _g1h->push_dirty_cards_region(card_region);
      }

      // If the card is dirty, then we will scan it during updateRS.
      if (!card_region->in_collection_set() &&
          !_ct_bs->is_card_dirty(card_index)) {
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

  double strong_code_root_scan_time_sec() {
    return _strong_code_root_scan_time_sec;
  }

  size_t cards_done() { return _cards_done;}
  size_t cards_looked_up() { return _cards;}
};
```


