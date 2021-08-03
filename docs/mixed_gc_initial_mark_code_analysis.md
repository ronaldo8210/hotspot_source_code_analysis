初始化标记的起始点为VMThread线程执行ConcurrentMark::scanRootRegions()：

```c++
void ConcurrentMark::scanRootRegions() {
  // Start of concurrent marking.
  ClassLoaderDataGraph::clear_claimed_marks();

  // scan_in_progress() will have been set to true only if there was
  // at least one root region to scan. So, if it's false, we
  // should not attempt to do any further work.
  if (root_regions()->scan_in_progress()) {
    _parallel_marking_threads = calc_parallel_marking_threads();
    assert(parallel_marking_threads() <= max_parallel_marking_threads(),
           "Maximum number of marking threads exceeded");
    uint active_workers = MAX2(1U, parallel_marking_threads());

    CMRootRegionScanTask task(this);
    if (use_parallel_marking_threads()) {
      _parallel_workers->set_active_workers((int) active_workers);
      _parallel_workers->run_task(&task);
    } else {
      task.work(0);
    }

    // It's possible that has_aborted() is true here without actually
    // aborting the survivor scan earlier. This is OK as it's
    // mainly used for sanity checking.
    root_regions()->scan_finished();
  }
}
```

具体工作封装在CMRootRegionScanTask类中，其work()函数会被多个GC worker线程调用：
```c++
class CMRootRegionScanTask : public AbstractGangTask {
private:
  ConcurrentMark* _cm;

public:
  CMRootRegionScanTask(ConcurrentMark* cm) :
    AbstractGangTask("Root Region Scan"), _cm(cm) { }

  void work(uint worker_id) {
    assert(Thread::current()->is_ConcurrentGC_thread(),
           "this should only be done by a conc GC thread");

    CMRootRegions* root_regions = _cm->root_regions();
    HeapRegion* hr = root_regions->claim_next();
    while (hr != NULL) {
      _cm->scanRootRegion(hr, worker_id);
      hr = root_regions->claim_next();
    }
  }
};
```

具体工作过程在ConcurrentMark::scanRootRegion()：
```c++
void ConcurrentMark::scanRootRegion(HeapRegion* hr, uint worker_id) {
  // Currently, only survivors can be root regions.
  assert(hr->next_top_at_mark_start() == hr->bottom(), "invariant");
  G1RootRegionScanClosure cl(_g1h, this, worker_id);

  const uintx interval = PrefetchScanIntervalInBytes;
  HeapWord* curr = hr->bottom();
  const HeapWord* end = hr->top();
  while (curr < end) {
    Prefetch::read(curr, interval);
    oop obj = oop(curr);
    int size = obj->oop_iterate(&cl);
    assert(size == obj->size(), "sanity");
    curr += size;
  }
}
```
