# 目录
* 基础数据结构
  * Java对象内存布局 
  * Region
  * [Card Table](docs/card_table.md)
  * [Remembered Set](docs/rset.md)
  * [Barrier Set](docs/barrier_set.md)  
  * Dirty Card Queue Set
  * SATB
* [Young GC](docs/ygc_principle.md)
  * [YGC源码实现解析](docs/ygc_code_analysis.md)
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
