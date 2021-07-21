YGC worker线程处理完Java根对象直接引用的年轻代对象和老年代对象直接引用的年轻代对象后，再以这些已被处理的年轻代对象为根，深度遍历其他存活的年轻代Java对象。具体过程是不断从RefToScanQueue队列中取得引用（待处理的Java对象的地址），如果Java对象在CSet中，则复制到区，再将该Java对象的所有引用属性加入RefToScanQueue队列。如此循环，直到队列清空，则完成遍历过程。

YGC worker线程执行Java对象深度遍历的函数调用链为：

- G1ParEvacuateFollowersClosure::do_void() --> G1ParScanThreadState::trim_queue() --> G1ParScanThreadState::deal_with_reference() --> G1ParScanThreadState::do_oop_evac() --> G1ParScanThreadState::copy_to_survivor_space()

各个函数主要代码的解析如下：

G1ParEvacuateFollowersClosure::do_void()
```c++
void G1ParEvacuateFollowersClosure::do_void() {
  G1ParScanThreadState* const pss = par_scan_state();
  // 从YGC worker线程私有的RefToScanQueue队列中不断取出引用，如果引用所指的Java对象在CSet，则将Java对象复制到survivor区，并将该Java对象的引用属性加入RefToScanQueue队列尾部
  // 直到队列清空，则这个YGC worker线程的深度优先遍历Java对象过程结束
  pss->trim_queue();
  do {
    // work steal，窃取并处理其他YGC worker线程的RefToScanQueue队列中的引用
    pss->steal_and_trim_queue(queues());
  } while (!offer_termination());
}
```

G1ParScanThreadState::trim_queue()
```c++
void G1ParScanThreadState::trim_queue() {
  assert(_evac_failure_cl != NULL, "not set");

  StarTask ref;
  do {
    // Drain the overflow stack first, so other threads can steal.
    while (_refs->pop_overflow(ref)) {
      dispatch_reference(ref);
    }

    while (_refs->pop_local(ref)) {
      // 从YGC worker线程私有的RefToScanQueue队列头部不断pop出引用并处理，直到队列清空
      dispatch_reference(ref);
    }
  } while (!_refs->is_empty());
}
```

G1ParScanThreadState::deal_with_reference()
```c++
template <class T> inline void G1ParScanThreadState::deal_with_reference(T* ref_to_scan) {
  if (!has_partial_array_mask(ref_to_scan)) {
    // Note: we can use "raw" versions of "region_containing" because
    // "obj_to_scan" is definitely in the heap, and is not in a
    // humongous region.
    HeapRegion* r = _g1h->heap_region_containing_raw(ref_to_scan);
    do_oop_evac(ref_to_scan, r);
  } else {
    do_oop_partial_array((oop*)ref_to_scan);
  }
}
```

G1ParScanThreadState::do_oop_evac()
```c++
template <class T> void G1ParScanThreadState::do_oop_evac(T* p, HeapRegion* from) {
  assert(!oopDesc::is_null(oopDesc::load_decode_heap_oop(p)),
         "Reference should not be NULL here as such are never pushed to the task queue.");
  oop obj = oopDesc::load_decode_heap_oop_not_null(p);

  const InCSetState in_cset_state = _g1h->in_cset_state(obj);
  if (in_cset_state.is_in_cset()) {
    oop forwardee;
    // 取得Java对象头信息指针
    markOop m = obj->mark();
    if (m->is_marked()) {
      // 如果Java对象已被复制到survivor区，直接取得复制对象的地址
      forwardee = (oop) m->decode_pointer();
    } else {
      // Java对象未被处理过，执行复制，并将其所有引用属性加入RefToScanQueue队列尾部
      forwardee = copy_to_survivor_space(in_cset_state, obj, m);
    }
    oopDesc::encode_store_heap_oop(p, forwardee);
  } else if (in_cset_state.is_humongous()) {
    _g1h->set_humongous_is_live(obj);
  } else {
    assert(!in_cset_state.is_in_cset_or_humongous(),
           err_msg("In_cset_state must be NotInCSet here, but is " CSETSTATE_FORMAT, in_cset_state.value()));
  }

  assert(obj != NULL, "Must be");
  // 引用关系发生变化，需要更新Remembered Set，将card地址入队列，具体更新操作由Refine线程执行
  update_rs(from, p, queue_num());
}
```

G1ParScanThreadState::copy_to_survivor_space()的代码在之前的文档分析过，主要是完成Java对象的复制，并将其所有引用属性加入RefToScanQueue队列尾部，用于后续的深度优先遍历。
