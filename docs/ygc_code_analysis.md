[以Java线程栈局部变量为根扫描&复制活跃对象](#jump_1)

[以老年代对象为根扫描&复制活跃对象](#jump_2)

## <span id="jump_1">以Java线程栈局部变量为根扫描&复制活跃对象</span>
一个YGC工作线程的调用链为：

1. 搜索Java线程栈中引用类型变量直接指向的年轻代的对象：
   - G1ParTask::work() --> G1RootProcessor::evacuate_roots() --> G1RootProcessor::process_java_roots() --> Threads::possibly_parallel_oops_do() --> 
JavaThread::oops_do() --> frame::oops_do() --> frame::oops_do_internal() --> frame::oops_interpreted_do() --> InterpreterOopMap::iterate_oop() --> G1ParCopyClosure::do_oop_work()

2. 

G1ParTask::work()：一次YGC启动时，多个YGC工作线程的入口函数
```c++
class G1ParTask : public AbstractGangTask {
protected:
  // ...

public:
  // ...

  void work(uint worker_id) {
    // work函数被多个YGC工作线程调用
    if (worker_id >= _n_workers) return;  // no work needed this round

    // ...

    {
      // ...

      // 扫描各种根对象，深度遍历根对象可达的活跃对象，将活跃对象复制到Survivor区。
      // 最主要的根对象是Java应用程序线程栈中的各个局部变量。
      _root_processor->evacuate_roots(strong_root_cl,
                                      weak_root_cl,
                                      strong_cld_cl,
                                      weak_cld_cl,
                                      trace_metadata,
                                      worker_id);

      G1ParPushHeapRSClosure push_heap_rs_cl(_g1h, &pss);
      // 遍历所有CSet的
      _root_processor->scan_remembered_sets(&push_heap_rs_cl,
                                            weak_root_cl,
                                            worker_id);
      // ...
    }
    // ...
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

  BufferingOopClosure buf_scan_non_heap_roots(scan_non_heap_roots);
  BufferingOopClosure buf_scan_non_heap_weak_roots(scan_non_heap_weak_roots);

  // ...

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

    // Traverse the execution stack
    for(StackFrameStream fst(this); !fst.is_done(); fst.next()) {
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

  InterpreterFrameClosure blk(this, max_locals, m->max_stack(), f);

  // process locals & expression stack
  InterpreterOopMap mask;
  if (query_oop_map_cache) {
    m->mask_for(bci, &mask);
  } else {
    OopMapCache::compute_one_oop_map(m, bci, &mask);
  }
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
  // iterate over entries
  for (int i = 0; i < n; i++, mask <<= bits_per_entry) {
    // get current word
    if (mask == 0) {
      value = bit_mask()[word_index++];
      mask = 1;
    }
    // test for oop
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
    // 先根据对象头的当前信息判断对象是否已被复制过，若已复制过，直接从对象头中取得复制后的对象的地址。
    // 这里处理的对象都是根对象，不会处理根对象的子对象。
    oop forwardee;
    markOop m = obj->mark();
    if (m->is_marked()) {
      // 对象已复制过，从对象头中取得复制后的对象的地址。
      forwardee = (oop) m->decode_pointer();
    } else {
      // 执行对象复制过程，返回复制后的对象的地址。
      forwardee = _par_scan_state->copy_to_survivor_space(state, obj, m);
    }
    assert(forwardee != NULL, "forwardee should not be NULL");
    // 令Java线程栈帧中的引用变量指向复制后的对象的地址。
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
    // 如果后续有混合回收阶段，该标记会在并发标记阶段体现作用。
    if (do_mark_object == G1MarkFromRoot) {
      mark_object(obj);
    }
  }

  if (barrier == G1BarrierEvac) {
    _par_scan_state->update_rs(_from, p, _worker_id);
  }
}
```

一个YGC工作线程的栈内存结构：



## <span id="jump_2">以老年代对象为根扫描&复制活跃对象</span>


G1中还有一个额外的数组称为BlockOffsetTable，记录了卡表对象的起始地址
