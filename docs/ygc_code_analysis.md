[以Java线程栈局部变量为根扫描&复制活跃对象](#以Java线程栈局部变量为根扫描&复制活跃对象)

[以老年代对象为根扫描&复制活跃对象](#以老年代对象为根扫描&复制活跃对象)

## 以Java线程栈局部变量为根扫描&复制活跃对象
一个YGC工作线程的调用链为：
G1ParTask::work() -> 

YGC启动时，多个YGC工作线程的入口为G1ParTask::work()：

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

## 以老年代对象为根扫描&复制活跃对象
G1中还有一个额外的数组称为BlockOffsetTable，记录了卡表对象的起始地址
