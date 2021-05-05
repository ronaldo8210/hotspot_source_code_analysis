[以Java线程栈局部变量为根扫描&复制活跃对象](#jump_1)

[以老年代对象为根扫描&复制活跃对象](#jump_2)

## <span id="jump_1">以Java线程栈局部变量为根扫描&复制活跃对象</span>
一个YGC工作线程的调用链为：
G1ParTask::work() --> G1RootProcessor::evacuate_roots() --> G1RootProcessor::process_java_roots() --> Threads::possibly_parallel_oops_do() --> 
JavaThread::oops_do() --> 

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



## <span id="jump_2">以老年代对象为根扫描&复制活跃对象</span>


G1中还有一个额外的数组称为BlockOffsetTable，记录了卡表对象的起始地址
