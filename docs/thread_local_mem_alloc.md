[线程私有缓存分配对象原理](#jump_1)

[TLAB分配对象源码解析](#jump_2)

## <span id="jump_1">线程私有缓存分配对象原理</span>

在JVM运行过程中，各Java业务线程为避免竞争，首先从线程私有的缓存区上分配对象，私有缓存的实现类为ThreadLocalAllocBuffer：

```c++
class ThreadLocalAllocBuffer: public CHeapObj<mtThread> {

private:
  HeapWord* _start;                              // TLAB内存块起始地址
  HeapWord* _top;                                // TLAB中已分配出去的地址
  HeapWord* _pf_top;                             // 与CPU缓存优化相关，先不考虑
  HeapWord* _end;                                // TLAB内存块结束地址
  size_t    _desired_size;                       // TLAB内存块大小
  size_t    _refill_waste_limit;                 // 最大浪费空间

};
```

在TLAB上分配对象的策略为：

1. 如果 _top + newObj_size 小于 _end，直接在TLAB上分配；

2. 如果 _top + newObj_size 大于 _end 且 TLAB剩余空间大于 _refill_waste_limit，保留TLAB，但newObj的分配不在TLAB进行，而是在年轻代Eden区进行；

3. 如果 _top + newObj_size 大于 _end 且 TLAB剩余空间小于 _refill_waste_limit，重新分配一块TLAB内存，在新的TLAB中分配newObj，原TLAB归还给Eden区。

例如，设TLAB内存块大小为10MB，_refill_waste_limit为5KB：

1. 假设当前TLAB空间还剩下3KB，Java线程new一个10KB的对象，3KB < _refill_waste_limit，此时需要申请一个新的TLAB，在新的TLAB上分配这个10KB的对象；

2. 假设当前TLAB空间还剩下10KB，Java线程new一个15KB的对象，10KB > _refill_waste_limit，此时还需要保留原TLAB，从年轻代Eden区为这个15KB的对象分配空间。

## <span id="jump_2">TLAB分配对象源码解析</span>

在TLAB上为一个普通的Java对象分配内存空间时，JVM虚拟机的工作流程为：

- InstanceKlass::allocate_instance() --> CollectedHeap::obj_allocate() --> CollectedHeap::common_mem_allocate_noinit() --> CollectedHeap::allocate_from_tlab()

allocate_from_tlab()函数：
```c++
HeapWord* CollectedHeap::allocate_from_tlab(KlassHandle klass, Thread* thread, size_t size) {
  assert(UseTLAB, "should use UseTLAB");

  // 先尝试从线程私有缓存TLAB上直接分配
  HeapWord* obj = thread->tlab().allocate(size);
  if (obj != NULL) {
    return obj;
  }
  // 如果从TLAB上直接分配失败，看是否需要申请新的TLAB，或者直接从年轻代Eden区分配
  return allocate_from_tlab_slow(klass, thread, size);
}
```

ThreadLocalAllocBuffer::allocate()函数：
```c++
inline HeapWord* ThreadLocalAllocBuffer::allocate(size_t size) {
  invariants();
  HeapWord* obj = top();
  if (pointer_delta(end(), obj) >= size) {
    // 如果 _top + size 小于 _end，则在TLAB上直接分配对象    
    size_t hdr_size = oopDesc::header_size();
    Copy::fill_to_words(obj + hdr_size, size - hdr_size, badHeapWordVal);

    // 更新_top指针
    set_top(obj + size);

    invariants();
    return obj;
  }
  return NULL;
}
```

CollectedHeap::allocate_from_tlab_slow()函数：
```c++
HeapWord* CollectedHeap::allocate_from_tlab_slow(KlassHandle klass, Thread* thread, size_t size) {

  // Retain tlab and allocate object in shared space if
  // the amount free in the tlab is too large to discard.
  if (thread->tlab().free() > thread->tlab().refill_waste_limit()) {
    thread->tlab().record_slow_allocation(size);
    return NULL;
  }

  // 计算新TLAB的内存大小
  size_t new_tlab_size = thread->tlab().compute_size(size);

  thread->tlab().clear_before_allocation();

  if (new_tlab_size == 0) {
    return NULL;
  }

  // 从Heap堆上分配一块新的TLAB内存
  HeapWord* obj = Universe::heap()->allocate_new_tlab(new_tlab_size);
  if (obj == NULL) {
    return NULL;
  }

  AllocTracer::send_allocation_in_new_tlab_event(klass, new_tlab_size * HeapWordSize, size * HeapWordSize);

  if (ZeroTLAB) {
    // ..and clear it.
    Copy::zero_to_words(obj, new_tlab_size);
  } else {
    // ...and zap just allocated object.
#ifdef ASSERT
    // Skip mangling the space corresponding to the object header to
    // ensure that the returned space is not considered parsable by
    // any concurrent GC thread.
    size_t hdr_size = oopDesc::header_size();
    Copy::fill_to_words(obj + hdr_size, new_tlab_size - hdr_size, badHeapWordVal);
#endif // ASSERT
  }
  thread->tlab().fill(obj, obj + size, new_tlab_size);
  return obj;
}
```
