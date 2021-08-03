# 目录
* Java对象内存布局
* Java对象分配过程
  * [在Java线程私有空间分配（快速分配）](docs/thread_local_mem_alloc.md) 
  * 直接在堆上分配（慢速分配）
* CollectedHeap
* HeapRegion
* 卡表
* Remembered Set
  * [RSet在GC过程中的作用](docs/rset_purpose.md)
  * [RSet内存结构](docs/rset_memory.md)
  * [引用变更期间RSet的工作过程](docs/rset_modify_reference.md)
  * [RSet的遍历](docs/rset_iterator.md)
  * Refine线程异步更新RSet
* BarrierSet
* WorkGroup多线程工作框架
* VMThread线程
* 安全点Safepoint
* Young GC
  * [YGC并行算法原理](docs/ygc_principle.md)
  * YGC过程源码解析
    * [1 扫描并复制Java根对象直接引用的年轻代对象](docs/ygc_code_analysis_1.md)
    * [2 扫描并复制老年代对象直接引用的年轻代对象](docs/ygc_code_analysis_2.md)
    * [3 深度遍历并复制1 & 2过程已处理的年轻代对象的所有引用对象](docs/ygc_code_analysis_3.md)
  * [YGC多线程执行流程与内存架构变迁](docs/ygc_memory_thread.md)
* Mixed GC
  * [Mixed GC并行算法原理](docs/mixed_gc_principle.md)
  * Mixed GC过程源码解析
    * [初始标记](docs/mixed_gc_initial_mark_code_analysis.md)
    * [并发标记](docs/mixed_gc_concurrent_mark_code_analysis.md)
    * [再标记](docs/mixed_gc_remark_code_analysis.md)
    * 垃圾清理
  * [Mixed GC多线程执行流程与内存架构变迁](docs/mixed_gc_memory_thread.md)
* Full GC
