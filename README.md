# 目录
* 基础数据结构
  * Java对象内存布局 
  * G1 Collected Heap
  * Heap Region
  * [Card Table](docs/card_table.md)
  * [Remembered Set](docs/rset.md)
  * [Barrier Set](docs/barrier_set.md)  
  * [Dirty Card Queue Set](docs/dcqs.md)
  * InterpreterOopMap
  * SATB
  * [各类可被回调对象Closure](docs/closure.md)
  * G1 Par Task
* Java对象分配过程
  * [在Java线程私有空间分配（快速分配）](docs/thread_local_mem_alloc.md) 
  * 直接在堆上分配（慢速分配）
* Remembered Set
  * [RSet原理与实例](docs/rset.md)
  * [Refine线程异步更新RSet](docs/refine_thread.md)
* Young GC
  * [YGC并行算法原理]（docs/ygc_principle.md）
  * [YGC过程源码解析](docs/ygc_code_analysis.md)
    * 1 扫描并复制Java根对象直接引用的年轻代对象
    * 2 扫描并复制老年代对象直接引用的年轻代对象
    * 3 深度遍历并复制1&2过程已处理的年轻代对象的所有子对象
  * [YGC多程执行流程与内存架构变迁](docs/ygc_memory_thread.md)
* Mixed GC
  * [Mixed GC并行算法原理]（docs/mixed_gc_principle.md）
  * 初始标记
    * [初始标记过程源码解析](docs/mixed_gc_initial_mark_code_analysis.md)
    * [初始标记多线程执行流程与内存架构变迁](docs/mixed_gc_initial_mark_memory_thread.md)
  * 并发标记
    * [并发标记过程源码解析](docs/mixed_gc_concurrent_mark_code_analysis.md)
    * [并发标记多线程执行流程与内存架构变迁](docs/mixed_gc_concurrent_mark_memory_thread.md)
  * 再标记
    * [再标记过程源码解析](docs/mixed_gc_remark_code_analysis.md)
    * [再标记多线程执行流程与内存架构变迁](docs/mixed_gc_remark_memory_thread.md)
  * 垃圾回收
* Full GC
