一次YGC开始时，VMThread线程负责启动N个YGC worker线程，N个YGC worker线程共同负责扫描所有的Java线程栈内存中的引用类型变量，每一个YGC worker线程在处理一个Java线程栈内存时，遍历栈中的所有Frame栈帧，将Frame中所有引用类型变量直接指向的年轻代的对象全部复制到survivor区，并将这些刚被复制的年轻代对象内的所有引用属性加入队列，用于后续的对象深度遍历。

上述过程的YGC worker线程函数调用链为：

  - G1ParTask::work() --> G1RootProcessor::evacuate_roots() --> G1RootProcessor::process_java_roots() --> Threads::possibly_parallel_oops_do() --> 
JavaThread::oops_do() --> frame::oops_do() --> frame::oops_do_internal() --> frame::oops_interpreted_do() --> InterpreterOopMap::iterate_oop() --> G1ParCopyClosure::do_oop_work() --> G1ParScanThreadState::copy_to_survivor_space() --> G1ParScanClosure::do_oop_nv()

各个函数主要代码的解析如下：

一个YGC worker线程的工作入口函数为G1ParTask::work()：
```c++
class G1ParTask : public AbstractGangTask {
protected:
  RefToScanQueueSet      *_queues;  // 全局的Queue Set，每一个YGC worker线程会被分配一个Queue，该队列用于存储YGC过程深度遍历到的对象的引用类型变量

public:
  // 取得一个YGC worker线程私有的用于存放引用类型变量的Queue
  RefToScanQueue *work_queue(int i) {
    return queues()->queue(i);
  }

  void work(uint worker_id) {
    // ...
    
    {
      G1ParScanThreadState            pss(_g1h, worker_id, rp);  // pss对象控制深度遍历过程中引用变量的进、出队列

      G1ParCopyClosure<G1BarrierNone, G1MarkFromRoot>         scan_mark_root_cl(_g1h, &pss, rp);  // 主要的回调对象，负责复制深度遍历到的存活的Java对象

      // 在evacuate_roots()中，一个YGC worker线程去遍历Java线程栈内存，如果一个Java线程栈未被处理，则遍历这个Java线程栈的所有Frame中的所有引用类型变量
      // 如果Frame中的引用类型变量指向的是年轻代Java对象，则将Java对象复制到survivor区，更新survivor区的RSet，并将刚被复制Java对象内的所有引用类型变量加入RefToScanQueue队列
      _root_processor->evacuate_roots(strong_root_cl,
                                      weak_root_cl,
                                      strong_cld_cl,
                                      weak_cld_cl,
                                      trace_metadata,
                                      worker_id);

      G1ParPushHeapRSClosure push_heap_rs_cl(_g1h, &pss);
      // 执行到这里，所有的Java线程栈内存都被遍历完，开始遍历CSet的所有Remembered Set，找到老年代对象直接引用的年轻代对象，复制到survivor区
      // 同时也要将刚被复制Java对象内的所有引用类型变量加入RefToScanQueue队列
      _root_processor->scan_remembered_sets(&push_heap_rs_cl,
                                            weak_root_cl,
                                            worker_id);

      {
        // evac对象负责深度遍历，直到所有存活的Java对象都被复制到survivor区
        G1ParEvacuateFollowersClosure evac(_g1h, &pss, _queues, &_terminator);
        evac.do_void();
      }
    }
  }
};
```

G1RootProcessor::evacuate_roots()
```c++
void G1RootProcessor::evacuate_roots(OopClosure* scan_non_heap_roots,
                                     OopClosure* scan_non_heap_weak_roots,
                                     CLDClosure* scan_strong_clds,
                                     CLDClosure* scan_weak_clds,
                                     bool trace_metadata,
                                     uint worker_i) {
  // ...

  // BufferingOopClosure类型对象内部有缓冲区，等到缓冲区中的对象到达阈值，再对这些对象批量调用OopClosure
  BufferingOopClosure buf_scan_non_heap_roots(scan_non_heap_roots);
  BufferingOopClosure buf_scan_non_heap_weak_roots(scan_non_heap_weak_roots);

  // ...

  // 开始处理Java根对象，不止包括Java线程栈内存引用变量直接引用的对象。其他类型的根对象暂略
  process_java_roots(strong_roots,
                     trace_metadata ? scan_strong_clds : NULL,
                     scan_strong_clds,
                     trace_metadata ? NULL : scan_weak_clds,
                     &root_code_blobs,
                     phase_times,
                     worker_i);

  // ...

  // Finish up any enqueued closure apps (attributed as object copy time).
  buf_scan_non_heap_roots.done();
  buf_scan_non_heap_weak_roots.done();

  // ...
}
```

G1RootProcessor::process_java_roots()
```c++
void G1RootProcessor::process_java_roots(OopClosure* strong_roots,
                                         CLDClosure* thread_stack_clds,
                                         CLDClosure* strong_clds,
                                         CLDClosure* weak_clds,
                                         CodeBlobClosure* strong_code,
                                         G1GCPhaseTimes* phase_times,
                                         uint worker_i) {
  // ...

  {
    G1GCParPhaseTimesTracker x(phase_times, G1GCPhaseTimes::ThreadRoots, worker_i);
    // 多个YGC worker线程会竞争遍历所有的Java线程栈，possibly_parallel_oops_do()返回时，就是所有的Java线程栈都被遍历完
    Threads::possibly_parallel_oops_do(strong_roots, thread_stack_clds, strong_code);
  }
}
```

Threads::possibly_parallel_oops_do()
```c++
void Threads::possibly_parallel_oops_do(OopClosure* f, CLDClosure* cld_f, CodeBlobClosure* cf) {
  // ...  
  ALL_JAVA_THREADS(p) {
    if (p->claim_oops_do(is_par, cp)) {
      // 一个YGC worker线程抢到一个Java栈内存，去遍历栈上所有的Frame
      p->oops_do(f, cld_f, cf);
    }
  }
  // ...
}
```

JavaThread::oops_do()
```c++
void JavaThread::oops_do(OopClosure* f, CLDClosure* cld_f, CodeBlobClosure* cf) {
  // ...

  if (has_last_Java_frame()) {
    // ...

    // 遍历每一个栈帧Frame
    for(StackFrameStream fst(this); !fst.is_done(); fst.next()) {
      // 遍历Frame中的每一个引用类型变量
      fst.current()->oops_do(f, cld_f, cf, fst.register_map());
    }
  }

  // ...
}
```

frame::oops_do()
```c++
void oops_do(OopClosure* f, CLDClosure* cld_f, CodeBlobClosure* cf, RegisterMap* map) { oops_do_internal(f, cld_f, cf, map, true); }
```

frame::oops_do_internal()
```c++
void frame::oops_do_internal(OopClosure* f, CLDClosure* cld_f, CodeBlobClosure* cf, RegisterMap* map, bool use_interpreter_oop_map_cache) {
  // ...
  if (is_interpreted_frame()) {
    // Java线程栈帧属于解释性栈帧，执行这个分支
    oops_interpreted_do(f, cld_f, map, use_interpreter_oop_map_cache);
  } else if (is_entry_frame()) {
    oops_entry_do(f, map);
  } else if (CodeCache::contains(pc())) {
    oops_code_blob_do(f, cf, map);
#ifdef SHARK
  } else if (is_fake_stub_frame()) {
    // nothing to do
#endif // SHARK
  } else {
    ShouldNotReachHere();
  }
}
```

frame::oops_interpreted_do()
```c++
void frame::oops_interpreted_do(OopClosure* f, CLDClosure* cld_f,
    const RegisterMap* map, bool query_oop_map_cache) {
  // ...
  // 回调对象，对Frame中的每一个引用类型变量，都调用一次blk
  InterpreterFrameClosure blk(this, max_locals, m->max_stack(), f);

  // process locals & expression stack
  InterpreterOopMap mask;
  if (query_oop_map_cache) {
    m->mask_for(bci, &mask);
  } else {
    OopMapCache::compute_one_oop_map(m, bci, &mask);
  }
  // 遍历Frame中的每一个引用类型变量，并对引用类型变量调用blk
  mask.iterate_oop(&blk);
}
```

InterpreterOopMap::iterate_oop()
```c++
void InterpreterOopMap::iterate_oop(OffsetClosure* oop_closure) const {
  int n = number_of_entries();
  int word_index = 0;
  uintptr_t value = 0;
  uintptr_t mask = 0;
  // 遍历Frame中的每一个引用类型变量
  for (int i = 0; i < n; i++, mask <<= bits_per_entry) {
    // get current word
    if (mask == 0) {
      value = bit_mask()[word_index++];
      mask = 1;
    }
    // 对每一个引用类型变量调用InterpreterFrameClosure回调对象
    if ((value & (mask << oop_bit_number)) != 0) oop_closure->offset_do(i);
  }
}
```

InterpreterFrameClosure::offset_do()
```c++
class InterpreterFrameClosure : public OffsetClosure {
 private:
  frame* _fr;
  OopClosure* _f;
  int    _max_locals;
  int    _max_stack;

 public:
  InterpreterFrameClosure(frame* fr, int max_locals, int max_stack,
                          OopClosure* f) {
    _fr         = fr;
    _max_locals = max_locals;
    _max_stack  = max_stack;
    _f          = f;
  }

  void offset_do(int offset) {
    oop* addr;
    if (offset < _max_locals) {
      addr = (oop*) _fr->interpreter_frame_local_at(offset);
      assert((intptr_t*)addr >= _fr->sp(), "must be inside the frame");
      // 将引用类型变量加入到BufferingOopClosure对象中的缓冲区，待后续批量处理
      _f->do_oop(addr);
    } else {
      addr = (oop*) _fr->interpreter_frame_expression_stack_at((offset - _max_locals));
      // In case of exceptions, the expression stack is invalid and the esp will be reset to express
      // this condition. Therefore, we call f only if addr is 'inside' the stack (i.e., addr >= esp for Intel).
      bool in_stack;
      if (frame::interpreter_frame_expression_stack_direction() > 0) {
        in_stack = (intptr_t*)addr <= _fr->interpreter_frame_tos_address();
      } else {
        in_stack = (intptr_t*)addr >= _fr->interpreter_frame_tos_address();
      }
      if (in_stack) {
        _f->do_oop(addr);
      }
    }
  }

  int max_locals()  { return _max_locals; }
  frame* fr()       { return _fr; }
};
```

G1ParCopyClosure::do_oop_work()（在g1CollectedHeap.cpp中定义）
```c++
template <G1Barrier barrier, G1Mark do_mark_object>
template <class T>
void G1ParCopyClosure<barrier, do_mark_object>::do_oop_work(T* p) {
  T heap_oop = oopDesc::load_heap_oop(p);

  if (oopDesc::is_null(heap_oop)) {
    return;
  }

  oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);

  assert(_worker_id == _par_scan_state->queue_num(), "sanity");

  const InCSetState state = _g1->in_cset_state(obj);
  if (state.is_in_cset()) {
    // 一个Java线程栈帧中的引用变量指向的年轻代中的对象可能同时也被另一个Java线程栈帧中的引用变量指向，
    // 先根据对象头的当前信息判断对象是否已被复制过，若已复制过，直接从对象头中取得复制后的对象的地址
    // 这里处理的对象都是根对象，不会处理根对象的子对象
    oop forwardee;
    markOop m = obj->mark();
    if (m->is_marked()) {
      // 对象已复制过，从对象头中取得复制后的对象的地址
      forwardee = (oop) m->decode_pointer();
    } else {
      // 执行对象复制过程，返回复制后的对象的地址
      forwardee = _par_scan_state->copy_to_survivor_space(state, obj, m);
    }
    assert(forwardee != NULL, "forwardee should not be NULL");
    // 令Java线程栈帧中的引用变量指向复制后的对象的地址
    oopDesc::encode_store_heap_oop(p, forwardee);
    if (do_mark_object != G1MarkNone && forwardee != obj) {
      // If the object is self-forwarded we don't need to explicitly
      // mark it, the evacuation failure protocol will do so.
      mark_forwarded_object(obj, forwardee);
    }

    if (barrier == G1BarrierKlass) {
      do_klass_barrier(p, forwardee);
    }
  } else {
    if (state.is_humongous()) {
      _g1->set_humongous_is_live(obj);
    }
    // The object is not in collection set. If we're a root scanning
    // closure during an initial mark pause then attempt to mark the object.
    // 如果在YGC阶段发现Java线程栈中的引用变量指向的是老年代中的对象，则对该老年代的对象进行标记处理，
    // 如果后续有混合回收阶段，该标记会在并发标记阶段体现作用
    if (do_mark_object == G1MarkFromRoot) {
      mark_object(obj);
    }
  }

  if (barrier == G1BarrierEvac) {
    _par_scan_state->update_rs(_from, p, _worker_id);
  }
}
```

G1ParScanThreadState::copy_to_survivor_space()
```c++
oop G1ParScanThreadState::copy_to_survivor_space(InCSetState const state,
                                                 oop const old,
                                                 markOop const old_mark) {
  const size_t word_sz = old->size();
  HeapRegion* const from_region = _g1h->heap_region_containing_raw(old);
  // +1 to make the -1 indexes valid...
  const int young_index = from_region->young_index_in_cset()+1;
  assert( (from_region->is_young() && young_index >  0) ||
         (!from_region->is_young() && young_index == 0), "invariant" );
  const AllocationContext_t context = from_region->allocation_context();

  uint age = 0;
  InCSetState dest_state = next_state(state, old_mark, age);
  HeapWord* obj_ptr = _g1_par_allocator->plab_allocate(dest_state, word_sz, context);

  // PLAB allocations should succeed most of the time, so we'll
  // normally check against NULL once and that's it.
  if (obj_ptr == NULL) {
    obj_ptr = _g1_par_allocator->allocate_direct_or_new_plab(dest_state, word_sz, context);
    if (obj_ptr == NULL) {
      obj_ptr = allocate_in_next_plab(state, &dest_state, word_sz, context);
      if (obj_ptr == NULL) {
        // This will either forward-to-self, or detect that someone else has
        // installed a forwarding pointer.
        return _g1h->handle_evacuation_failure_par(this, old);
      }
    }
  }

  assert(obj_ptr != NULL, "when we get here, allocation should have succeeded");
#ifndef PRODUCT
  // Should this evacuation fail?
  if (_g1h->evacuation_should_fail()) {
    // Doing this after all the allocation attempts also tests the
    // undo_allocation() method too.
    _g1_par_allocator->undo_allocation(dest_state, obj_ptr, word_sz, context);
    return _g1h->handle_evacuation_failure_par(this, old);
  }
#endif // !PRODUCT

  // We're going to allocate linearly, so might as well prefetch ahead.
  Prefetch::write(obj_ptr, PrefetchCopyIntervalInBytes);

  const oop obj = oop(obj_ptr);
  const oop forward_ptr = old->forward_to_atomic(obj);
  if (forward_ptr == NULL) {
    Copy::aligned_disjoint_words((HeapWord*) old, obj_ptr, word_sz);

    if (dest_state.is_young()) {
      if (age < markOopDesc::max_age) {
        age++;
      }
      if (old_mark->has_displaced_mark_helper()) {
        // In this case, we have to install the mark word first,
        // otherwise obj looks to be forwarded (the old mark word,
        // which contains the forward pointer, was copied)
        obj->set_mark(old_mark);
        markOop new_mark = old_mark->displaced_mark_helper()->set_age(age);
        old_mark->set_displaced_mark_helper(new_mark);
      } else {
        obj->set_mark(old_mark->set_age(age));
      }
      age_table()->add(age, word_sz);
    } else {
      obj->set_mark(old_mark);
    }

    if (G1StringDedup::is_enabled()) {
      const bool is_from_young = state.is_young();
      const bool is_to_young = dest_state.is_young();
      assert(is_from_young == _g1h->heap_region_containing_raw(old)->is_young(),
             "sanity");
      assert(is_to_young == _g1h->heap_region_containing_raw(obj)->is_young(),
             "sanity");
      G1StringDedup::enqueue_from_evacuation(is_from_young,
                                             is_to_young,
                                             queue_num(),
                                             obj);
    }

    size_t* const surv_young_words = surviving_young_words();
    surv_young_words[young_index] += word_sz;

    if (obj->is_objArray() && arrayOop(obj)->length() >= ParGCArrayScanChunk) {
      // We keep track of the next start index in the length field of
      // the to-space object. The actual length can be found in the
      // length field of the from-space object.
      arrayOop(obj)->set_length(0);
      oop* old_p = set_partial_array_mask(old);
      push_on_queue(old_p);
    } else {
      HeapRegion* const to_region = _g1h->heap_region_containing_raw(obj_ptr);
      _scanner.set_region(to_region);
      // 遍历刚刚被复制的Java对象的每个引用属性，对每个引用属性调用G1ParScanClosure对象，将引用加入到RefToScanQueue队列
      obj->oop_iterate_backwards(&_scanner);
    }
    return obj;
  } else {
    _g1_par_allocator->undo_allocation(dest_state, obj_ptr, word_sz, context);
    return forward_ptr;
  }
}
```

G1ParScanClosure::do_oop_nv()
```c++
template <class T>
inline void G1ParScanClosure::do_oop_nv(T* p) {
  T heap_oop = oopDesc::load_heap_oop(p);

  if (!oopDesc::is_null(heap_oop)) {
    oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);
    const InCSetState state = _g1->in_cset_state(obj);
    if (state.is_in_cset()) {
      // We're not going to even bother checking whether the object is
      // already forwarded or not, as this usually causes an immediate
      // stall. We'll try to prefetch the object (for write, given that
      // we might need to install the forwarding reference) and we'll
      // get back to it when pop it from the queue
      Prefetch::write(obj->mark_addr(), 0);
      Prefetch::read(obj->mark_addr(), (HeapWordSize*2));

      // slightly paranoid test; I'm trying to catch potential
      // problems before we go into push_on_queue to know where the
      // problem is coming from
      assert((obj == oopDesc::load_decode_heap_oop(p)) ||
             (obj->is_forwarded() &&
                 obj->forwardee() == oopDesc::load_decode_heap_oop(p)),
             "p should still be pointing to obj or to its forwardee");

      // 将引用属性加入到RefToScanQueue队列
      _par_scan_state->push_on_queue(p);
    } else {
      if (state.is_humongous()) {
        _g1->set_humongous_is_live(obj);
      }
      _par_scan_state->update_rs(_from, p, _worker_id);
    }
  }
}
```

