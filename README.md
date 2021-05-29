# 目录
* 基础数据结构
  * Java对象内存布局 
  * G1 Collected Heap
  * Heap Region
  * [Card Table](docs/card_table.md)
  * [Remembered Set](docs/rset.md)
  * [Barrier Set](docs/barrier_set.md)  
  * Dirty Card Queue Set
  * InterpreterOopMap
  * SATB
  * [各类可被回调对象Closure](docs/closure.md)
  * G1 Par Task
* Java对象分配过程
  * [在Java线程私有空间分配](docs/thread_local_mem_alloc.md) 
* [Young GC](docs/ygc_principle.md)
  * [YGC源码实现解析](docs/ygc_code_analysis.md)
    * 1 扫描并复制Java根对象直接引用的年轻代对象
    * 2 扫描并复制老年代对象直接引用的年轻代对象
    * 3 深度遍历并复制1&2过程已处理的年轻代对象的所有子对象
  * [一次YGC的内存变迁过程&各路线程执行过程](docs/ygc_memory_thread.md)
* [混合回收](docs/mixed_gc_principle.md)
  * 初始化标记
    * [初始化标记源码实现解析](docs/mixed_gc_initial_mark_code_analysis.md)
    * [一次初始化标记的内存变迁过程&各路线程执行过程](docs/mixed_gc_initial_mark_memory_thread.md)
  * 并发标记
    * [并发标记源码实现解析](docs/mixed_gc_concurrent_mark_code_analysis.md)
    * [一次并发标记的内存变迁过程&各路线程执行过程](docs/mixed_gc_initial_mark_memory_thread.md)
  * 再标记
    * [再标记源码实现解析](docs/mixed_gc_remark_code_analysis.md)
    * [一次再标记的内存变迁过程&各路线程执行过程](docs/mixed_gc_initial_mark_memory_thread.md)
  * 垃圾回收
* Full GC
