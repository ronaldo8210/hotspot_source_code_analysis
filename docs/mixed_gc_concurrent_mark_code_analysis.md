并发标记的执行过程如下：

1. VMThread线程创建标记Task对象，启动多个并发标记worker线程去执行Task对象的work()方法，并将自身挂起，等待所有并发标记worker线程完成任务；

2. 每个并发标记worker线程，


VMThread线程启动多个并发标记worker线程的主要代码如下：
```c++
void ConcurrentMark::markFromRoots() {

  _restart_for_overflow = false;
  force_overflow_conc()->init();

  // 计算需要的并发标记worker线程的数量
  _parallel_marking_threads = calc_parallel_marking_threads();
  assert(parallel_marking_threads() <= max_parallel_marking_threads(),
    "Maximum number of marking threads exceeded");

  // 并发标记worker线程的数量最小为1
  uint active_workers = MAX2(1U, parallel_marking_threads());

  // Parallel task terminator is set in "set_concurrency_and_phase()"
  set_concurrency_and_phase(active_workers, true /* concurrent */);

  // 定义具体的标记Task对象，多线程的并发标记全部结束前，markFromRoots()不会return，所以将该Task对象定义在线程栈上是安全的
  CMConcurrentMarkingTask markingTask(this, cmThread());
  if (use_parallel_marking_threads()) {
    // 设置并发标记worker线程的数量
    _parallel_workers->set_active_workers((int)active_workers);
    assert(_parallel_workers->active_workers() > 0, "Should have been set");
    // 多个并发标记worker线程同时启动，都去执行markingTask对象的work()方法
    // 在run_task内部，VMThread线程会挂起，所有并发标记worker线程都完成各自工作后，run_task()才会return
    _parallel_workers->run_task(&markingTask);
  } else {
    markingTask.work(0);
  }
  print_stats();
}
```

每个并发标记worker线程的工作入口为CMConcurrentMarkingTask::work()，主要代码如下：
```c++
class CMConcurrentMarkingTask: public AbstractGangTask {
public:
  void work(uint worker_id) {
    // 执行CMConcurrentMarkingTask::work()的线程必须是GC工作线程，不能是其他类型线程
    assert(Thread::current()->is_ConcurrentGC_thread(),
           "this should only be done by a conc GC thread");
    ResourceMark rm;

    // 一个并发标记worker线程开始执行的时间戳
    double start_vtime = os::elapsedVTime();

    SuspendibleThreadSet::join();

    // 并发标记worker线程的worker_id值必须在[0, 并发标记worker线程个数)之间
    assert(worker_id < _cm->active_tasks(), "invariant");
    // 从ConcurrentMark对象的CMTask数组中取一个CMTask对象，CMTask对象存储并发标记worker线程工作过程中的各类状态
    CMTask* the_task = _cm->task(worker_id);
    // 记录一个并发标记worker线程开始执行的时间戳
    the_task->record_start_time();
    if (!_cm->has_aborted()) {
      do {
        double start_vtime_sec = os::elapsedVTime();
        // 由于每一个并发标记worker线程除了要对存活的Java对象打标外，还要处理SATB队列，所以限制打标操作的时间，mark_step_duration_ms就是一轮打标操作的限制时间
        // 一轮打标操作耗时超过mark_step_duration_ms，就要停止打标，去处理ASTB队列，指定数量的ASTB队列元素处理完成后，再继续打标
        double mark_step_duration_ms = G1ConcMarkStepDurationMillis;

        // 在do_marking_step()中执行打标操作、处理SATB队列
        // 1、在do_marking_step()开始阶段先判断SATB队列内是否已存有自身引用发生变更的Java对象，若是则先处理一部分SATB队列中的Java对象，
        //   将这些Java对象判定为灰色对象，对其引用的所有对象进行深度遍历打标。这部分灰标Java对象的深度遍历全部完成后再去对分配给该并发标记worker线程的Region中的Java对象打标；
        // 2、如果SATB队列为空，则直接开始对分配给该并发标记worker线程的Region中的Java对象打标
        // 3、在对Region中的Java对象打标的过程中，如果耗时时间超过mark_step_duration_ms，或者发现SATB中新加入的引用发生变更的Java对象累积到了一定数量，
        //   则在CMTask对象中将工作状态设置为abort，并从do_marking_step()调用中return，
        //   后续会在while循环中继续进入do_marking_step()，优先去处理SATB队列中新加入的引用发生变更的Java对象
        the_task->do_marking_step(mark_step_duration_ms,
                                  true  /* do_termination */,
                                  false /* is_serial*/);

        // 一轮打标操作的停止时间戳
        double end_vtime_sec = os::elapsedVTime();
        double elapsed_vtime_sec = end_vtime_sec - start_vtime_sec;
        _cm->clear_has_overflown();

        _cm->do_yield_check(worker_id);

        jlong sleep_time_ms;
        if (!_cm->has_aborted() && the_task->has_aborted()) {
          sleep_time_ms =
            (jlong) (elapsed_vtime_sec * _cm->sleep_factor() * 1000.0);
          SuspendibleThreadSet::leave();
          os::sleep(Thread::current(), sleep_time_ms, false);
          SuspendibleThreadSet::join();
        }
      } while (!_cm->has_aborted() && the_task->has_aborted());  // 如果是一轮打标操作耗时超过预定时间或者SATB队列内新加入的引用发生变更的Java对象累积到了一定数量导致打标操作中止，
                                                                 // 则继续while循环，重新开始执行do_marking_step()
    }
    the_task->record_end_time();
    // 走到这里，该并发标记worker线程已完成全部打标任务，它对应的CMTask对象记录的工作状态肯定不会是abort
    guarantee(!the_task->has_aborted() || _cm->has_aborted(), "invariant");

    SuspendibleThreadSet::leave();

    double end_vtime = os::elapsedVTime();
    _cm->update_accum_task_vtime(worker_id, end_vtime - start_vtime);
  }
};
```

每个并发标记worker线程对各自处理的Region中存活的Java对象打标、以及处理SATB队列的具体过程在CMTask::do_marking_step()，主要代码如下：
```c++
void CMTask::do_marking_step(double time_target_ms,
                             bool do_termination,
                             bool is_serial) {
  assert(time_target_ms >= 1.0, "minimum granularity is 1ms");
  assert(concurrent() == _cm->concurrent(), "they should be the same");

  G1CollectorPolicy* g1_policy = _g1h->g1_policy();
  assert(_task_queues != NULL, "invariant");
  assert(_task_queue != NULL, "invariant");
  assert(_task_queues->queue(_worker_id) == _task_queue, "invariant");

  assert(!_claimed,
         "only one thread should claim this task at any one time");

  // 标记已有GC线程占用CMTask对象
  _claimed = true;

  _start_time_ms = os::elapsedVTime() * 1000.0;
  statsOnly( _interval_start_time_ms = _start_time_ms );

  // 判断是否需要work steal，work steal指一个并发标记worker线程处理完分配给它的Region内的Java对象打标工作后，
  // 继续去抢其他worker线程的工作，去处理原本分配给其他worker线程的Region内的Java对象打标工作，
  // 从总体上尽可能的缩短全局标记工作的执行时间
  bool do_stealing = do_termination && !is_serial;

  double diff_prediction_ms =
    g1_policy->get_new_prediction(&_marking_step_diffs_ms);
  _time_target_ms = time_target_ms - diff_prediction_ms;

  // 标记的字节数、处理的引用数清零，一旦标记的字节数或处理的引用数超过预定值，do_marking_step()会return
  _words_scanned = 0;
  _refs_reached  = 0;
  // 重新计算本轮标记需要处理的Region内的对象范围
  recalculate_limits();

  // clear all flags
  clear_has_aborted();
  _has_timed_out = false;
  _draining_satb_buffers = false;

  ++_calls;

  if (_cm->verbose_low()) {
    gclog_or_tty->print_cr("[%u] >>>>>>>>>> START, call = %d, "
                           "target = %1.2lfms >>>>>>>>>>",
                           _worker_id, _calls, _time_target_ms);
  }

  // 下面两个对象都是分配在线程栈上的，一轮标记结束后就销毁，下一轮标记过程再创建
  // bitmap_closure对象负责处理_nextMarkBitMap位图上的被设置为1的bit，也就是处理bit映射到的灰色对象
  CMBitMapClosure bitmap_closure(this, _cm, _nextMarkBitMap);
  // cm_oop_closure对象
  G1CMOopClosure  cm_oop_closure(_g1h, _cm, this);
  set_cm_oop_closure(&cm_oop_closure);

  if (_cm->has_overflown()) {
    set_has_aborted();
  }

  // 为了尽量维持并发标记开始时刻的整体内存的快照，优先处理SATB队列中的引用发生变化的Java对象，重新对这些Java对象执行深度遍历打标
  // 将SATB中的Java对象全部处理完再往下执行
  drain_satb_buffers();
  // 处理_task_queue中的一部分数据，在打标过程中，当前正在处理的灰标Java对象，其直接引用的对象也会被当作灰标对象，放入_task_queue中
  // 后续再从_task_queue中pop出灰标对象，遍历其引用的对象
  drain_local_queue(true);
  drain_global_stack(true);

  do {
    if (!has_aborted() && _curr_region != NULL) {
      // _finger指向_curr_region内本轮标记开始的起点
      assert(_finger != NULL, "if region is not NULL, then the finger "
             "should not be NULL either");

      // 更新本轮标记的最大范围，之前可能由于标记过程被abort，重新进入do_marking_step()，因为标记线程和Java业务线程同时执行，
      // Region内可能已分配了新的Java对象，所以要重新计算本轮标记的处理范围
      update_region_limit();
      MemRegion mr = MemRegion(_finger, _region_limit);

      if (_cm->verbose_low()) {
        gclog_or_tty->print_cr("[%u] we're scanning part "
                               "["PTR_FORMAT", "PTR_FORMAT") "
                               "of region "HR_FORMAT,
                               _worker_id, p2i(_finger), p2i(_region_limit),
                               HR_FORMAT_PARAMS(_curr_region));
      }

      assert(!_curr_region->isHumongous() || mr.start() == _curr_region->bottom(),
             "humongous regions should go around loop once only");

      if (mr.is_empty()) {
        giveup_current_region();
        regular_clock_call();
      } else if (_curr_region->isHumongous() && mr.start() == _curr_region->bottom()) {
        if (_nextMarkBitMap->isMarked(mr.start())) {
          // The object is marked - apply the closure
          BitMap::idx_t offset = _nextMarkBitMap->heapWordToOffset(mr.start());
          bitmap_closure.do_bit(offset);
        }
        // Even if this task aborted while scanning the humongous object
        // we can (and should) give up the current region.
        giveup_current_region();
        regular_clock_call();
      } else if (_nextMarkBitMap->iterate(&bitmap_closure, mr)) {
        // 主要关注这里，遍历_nextMarkBitMap位图的mr标识范围的区域，对每个置1的bit回调bitmap_closure对象的do_bit()函数
        // 如果iterate返回值==true，则mr标识范围内的置1的bit全部处理完，即相应的灰标对象都完成了引用的遍历，它们直接引用的Java对象的地址都被放入_task_queue队列
        // 如果iterate返回值==false，说明CMTask对象已被设置为aborted状态，需要重新进入do_marking_step()去处理SATB队列
        
        // 当前的Region已处理完，清空相关指针
        giveup_current_region();
        // 即使当前Region全部处理完且没有abort，还是要检查下各类abort触发条件，判断是否需要abort
        regular_clock_call();
      } else {
        // 遍历_nextMarkBitMap位图过程中发生abort，走到这里
        assert(has_aborted(), "currently the only way to do so");
        assert(_finger != NULL, "invariant");

        assert(_finger < _region_limit, "invariant");
        // 计算Region中下次将被处理的灰标Java对象的地址
        HeapWord* new_finger = _nextMarkBitMap->nextObject(_finger);
        // Check if bitmap iteration was aborted while scanning the last object
        if (new_finger >= _region_limit) {
          giveup_current_region();
        } else {
          // 更新_finger指向Region中下次将被处理的灰标Java对象
          move_finger_to(new_finger);
        }
      }
    }
    // At this point we have either completed iterating over the
    // region we were holding on to, or we have aborted.

    // 处理一部分_task_queue队列中的对象（是否有必要？）
    drain_local_queue(true);
    drain_global_stack(true);

    while (!has_aborted() && _curr_region == NULL && !_cm->out_of_regions()) {
      // 只有上一个Region成功处理完成才会走到这里
      assert(_curr_region  == NULL, "invariant");
      assert(_finger       == NULL, "invariant");
      assert(_region_limit == NULL, "invariant");
      if (_cm->verbose_low()) {
        gclog_or_tty->print_cr("[%u] trying to claim a new region", _worker_id);
      }
      // 给当前标记线程分配一个新的待处理的Region
      HeapRegion* claimed_region = _cm->claim_region(_worker_id);
      if (claimed_region != NULL) {
        statsOnly( ++_regions_claimed );

        if (_cm->verbose_low()) {
          gclog_or_tty->print_cr("[%u] we successfully claimed "
                                 "region "PTR_FORMAT,
                                 _worker_id, p2i(claimed_region));
        }

        setup_for_region(claimed_region);
        assert(_curr_region == claimed_region, "invariant");
      }
      regular_clock_call();
    }

    if (!has_aborted() && _curr_region == NULL) {
      assert(_cm->out_of_regions(),
             "at this point we should be out of regions");
    }
  } while ( _curr_region != NULL && !has_aborted());  // 在下一个while循环中处理新分配的Region

  // 走到这里，说明已没有剩余的可被处理的Region能够分配给该标记线程了
  
  if (!has_aborted()) {
    assert(_cm->out_of_regions(),
           "at this point we should be out of regions");

    if (_cm->verbose_low()) {
      gclog_or_tty->print_cr("[%u] all regions claimed", _worker_id);
    }

    drain_satb_buffers();
  }

  // 将_task_queue队列中的对象全部处理完
  drain_local_queue(false);
  drain_global_stack(false);

  // 开启work steal，暂不深究
  if (do_stealing && !has_aborted()) {
    // We have not aborted. This means that we have finished all that
    // we could. Let's try to do some stealing...

    // We cannot check whether the global stack is empty, since other
    // tasks might be pushing objects to it concurrently.
    assert(_cm->out_of_regions() && _task_queue->size() == 0,
           "only way to reach here");

    if (_cm->verbose_low()) {
      gclog_or_tty->print_cr("[%u] starting to steal", _worker_id);
    }

    while (!has_aborted()) {
      oop obj;
      statsOnly( ++_steal_attempts );

      if (_cm->try_stealing(_worker_id, &_hash_seed, obj)) {
        if (_cm->verbose_medium()) {
          gclog_or_tty->print_cr("[%u] stolen "PTR_FORMAT" successfully",
                                 _worker_id, p2i((void*) obj));
        }

        statsOnly( ++_steals );

        assert(_nextMarkBitMap->isMarked((HeapWord*) obj),
               "any stolen object should be marked");
        scan_object(obj);

        // And since we're towards the end, let's totally drain the
        // local queue and global stack.
        drain_local_queue(false);
        drain_global_stack(false);
      } else {
        break;
      }
    }
  }

  // 一些收尾工作的代码暂略

  _claimed = false;
}
```

update_region_limit()的实现代码如下：
```c++
void CMTask::update_region_limit() {
  HeapRegion* hr            = _curr_region;
  // bottom指针指向Region的尾部
  HeapWord* bottom          = hr->bottom();
  HeapWord* limit           = hr->next_top_at_mark_start();

  if (limit == bottom) {
    if (_cm->verbose_low()) {
      gclog_or_tty->print_cr("[%u] found an empty region "
                             "["PTR_FORMAT", "PTR_FORMAT")",
                             _worker_id, p2i(bottom), p2i(limit));
    }
    // The region was collected underneath our feet.
    // We set the finger to bottom to ensure that the bitmap
    // iteration that will follow this will not do anything.
    // (this is not a condition that holds when we set the region up,
    // as the region is not supposed to be empty in the first place)
    _finger = bottom;
  } else if (limit >= _region_limit) {
    assert(limit >= _finger, "peace of mind");
  } else {
    assert(limit < _region_limit, "only way to get here");
    // This can happen under some pretty unusual circumstances.  An
    // evacuation pause empties the region underneath our feet (NTAMS
    // at bottom). We then do some allocation in the region (NTAMS
    // stays at bottom), followed by the region being used as a GC
    // alloc region (NTAMS will move to top() and the objects
    // originally below it will be grayed). All objects now marked in
    // the region are explicitly grayed, if below the global finger,
    // and we do not need in fact to scan anything else. So, we simply
    // set _finger to be limit to ensure that the bitmap iteration
    // doesn't do anything.
    _finger = limit;
  }

  _region_limit = limit;
}
```

遍历_nextMarkBitMap位图的回调函数为CMBitMapClosure::do_bit()，实现代码如下：
```c++
class CMBitMapClosure : public BitMapClosure {
public:
  bool do_bit(size_t offset) {
    HeapWord* addr = _nextMarkBitMap->offsetToHeapWord(offset);
    assert(_nextMarkBitMap->isMarked(addr), "invariant");
    assert( addr < _cm->finger(), "invariant");

    statsOnly( _task->increase_objs_found_on_bitmap() );
    assert(addr >= _task->finger(), "invariant");

    // We move that task's local finger along.
    _task->move_finger_to(addr);

    _task->scan_object(oop(addr));
    // we only partially drain the local queue and global stack
    _task->drain_local_queue(true);
    _task->drain_global_stack(true);

    // if the has_aborted flag has been raised, we need to bail out of
    // the iteration
    return !_task->has_aborted();
  }
};
```

