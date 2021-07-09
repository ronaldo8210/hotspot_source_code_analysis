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
             将这些Java对象判定为灰色对象，对其引用的所有对象进行深度遍历打标。这部分灰标Java对象的深度遍历全部完成后再去对分配给该并发标记worker线程的Region中的Java对象打标；
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

  // Set up the bitmap and oop closures. Anything that uses them is
  // eventually called from this method, so it is OK to allocate these
  // statically.
  CMBitMapClosure bitmap_closure(this, _cm, _nextMarkBitMap);
  G1CMOopClosure  cm_oop_closure(_g1h, _cm, this);
  set_cm_oop_closure(&cm_oop_closure);

  if (_cm->has_overflown()) {
    // This can happen if the mark stack overflows during a GC pause
    // and this task, after a yield point, restarts. We have to abort
    // as we need to get into the overflow protocol which happens
    // right at the end of this task.
    set_has_aborted();
  }

  // First drain any available SATB buffers. After this, we will not
  // look at SATB buffers before the next invocation of this method.
  // If enough completed SATB buffers are queued up, the regular clock
  // will abort this task so that it restarts.
  drain_satb_buffers();
  // ...then partially drain the local queue and the global stack
  drain_local_queue(true);
  drain_global_stack(true);

  do {
    if (!has_aborted() && _curr_region != NULL) {
      // This means that we're already holding on to a region.
      assert(_finger != NULL, "if region is not NULL, then the finger "
             "should not be NULL either");

      // We might have restarted this task after an evacuation pause
      // which might have evacuated the region we're holding on to
      // underneath our feet. Let's read its limit again to make sure
      // that we do not iterate over a region of the heap that
      // contains garbage (update_region_limit() will also move
      // _finger to the start of the region if it is found empty).
      update_region_limit();
      // We will start from _finger not from the start of the region,
      // as we might be restarting this task after aborting half-way
      // through scanning this region. In this case, _finger points to
      // the address where we last found a marked object. If this is a
      // fresh region, _finger points to start().
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

      // Some special cases:
      // If the memory region is empty, we can just give up the region.
      // If the current region is humongous then we only need to check
      // the bitmap for the bit associated with the start of the object,
      // scan the object if it's live, and give up the region.
      // Otherwise, let's iterate over the bitmap of the part of the region
      // that is left.
      // If the iteration is successful, give up the region.
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
        giveup_current_region();
        regular_clock_call();
      } else {
        assert(has_aborted(), "currently the only way to do so");
        // The only way to abort the bitmap iteration is to return
        // false from the do_bit() method. However, inside the
        // do_bit() method we move the _finger to point to the
        // object currently being looked at. So, if we bail out, we
        // have definitely set _finger to something non-null.
        assert(_finger != NULL, "invariant");

        // Region iteration was actually aborted. So now _finger
        // points to the address of the object we last scanned. If we
        // leave it there, when we restart this task, we will rescan
        // the object. It is easy to avoid this. We move the finger by
        // enough to point to the next possible object header (the
        // bitmap knows by how much we need to move it as it knows its
        // granularity).
        assert(_finger < _region_limit, "invariant");
        HeapWord* new_finger = _nextMarkBitMap->nextObject(_finger);
        // Check if bitmap iteration was aborted while scanning the last object
        if (new_finger >= _region_limit) {
          giveup_current_region();
        } else {
          move_finger_to(new_finger);
        }
      }
    }
    // At this point we have either completed iterating over the
    // region we were holding on to, or we have aborted.

    // We then partially drain the local queue and the global stack.
    // (Do we really need this?)
    drain_local_queue(true);
    drain_global_stack(true);

    // Read the note on the claim_region() method on why it might
    // return NULL with potentially more regions available for
    // claiming and why we have to check out_of_regions() to determine
    // whether we're done or not.
    while (!has_aborted() && _curr_region == NULL && !_cm->out_of_regions()) {
      // We are going to try to claim a new region. We should have
      // given up on the previous one.
      // Separated the asserts so that we know which one fires.
      assert(_curr_region  == NULL, "invariant");
      assert(_finger       == NULL, "invariant");
      assert(_region_limit == NULL, "invariant");
      if (_cm->verbose_low()) {
        gclog_or_tty->print_cr("[%u] trying to claim a new region", _worker_id);
      }
      HeapRegion* claimed_region = _cm->claim_region(_worker_id);
      if (claimed_region != NULL) {
        // Yes, we managed to claim one
        statsOnly( ++_regions_claimed );

        if (_cm->verbose_low()) {
          gclog_or_tty->print_cr("[%u] we successfully claimed "
                                 "region "PTR_FORMAT,
                                 _worker_id, p2i(claimed_region));
        }

        setup_for_region(claimed_region);
        assert(_curr_region == claimed_region, "invariant");
      }
      // It is important to call the regular clock here. It might take
      // a while to claim a region if, for example, we hit a large
      // block of empty regions. So we need to call the regular clock
      // method once round the loop to make sure it's called
      // frequently enough.
      regular_clock_call();
    }

    if (!has_aborted() && _curr_region == NULL) {
      assert(_cm->out_of_regions(),
             "at this point we should be out of regions");
    }
  } while ( _curr_region != NULL && !has_aborted());

  if (!has_aborted()) {
    // We cannot check whether the global stack is empty, since other
    // tasks might be pushing objects to it concurrently.
    assert(_cm->out_of_regions(),
           "at this point we should be out of regions");

    if (_cm->verbose_low()) {
      gclog_or_tty->print_cr("[%u] all regions claimed", _worker_id);
    }

    // Try to reduce the number of available SATB buffers so that
    // remark has less work to do.
    drain_satb_buffers();
  }

  // Since we've done everything else, we can now totally drain the
  // local queue and global stack.
  drain_local_queue(false);
  drain_global_stack(false);

  // Attempt at work stealing from other task's queues.
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

  // If we are about to wrap up and go into termination, check if we
  // should raise the overflow flag.
  if (do_termination && !has_aborted()) {
    if (_cm->force_overflow()->should_force()) {
      _cm->set_has_overflown();
      regular_clock_call();
    }
  }

  // We still haven't aborted. Now, let's try to get into the
  // termination protocol.
  if (do_termination && !has_aborted()) {
    // We cannot check whether the global stack is empty, since other
    // tasks might be concurrently pushing objects on it.
    // Separated the asserts so that we know which one fires.
    assert(_cm->out_of_regions(), "only way to reach here");
    assert(_task_queue->size() == 0, "only way to reach here");

    if (_cm->verbose_low()) {
      gclog_or_tty->print_cr("[%u] starting termination protocol", _worker_id);
    }

    _termination_start_time_ms = os::elapsedVTime() * 1000.0;

    // The CMTask class also extends the TerminatorTerminator class,
    // hence its should_exit_termination() method will also decide
    // whether to exit the termination protocol or not.
    bool finished = (is_serial ||
                     _cm->terminator()->offer_termination(this));
    double termination_end_time_ms = os::elapsedVTime() * 1000.0;
    _termination_time_ms +=
      termination_end_time_ms - _termination_start_time_ms;

    if (finished) {
      // We're all done.

      if (_worker_id == 0) {
        // let's allow task 0 to do this
        if (concurrent()) {
          assert(_cm->concurrent_marking_in_progress(), "invariant");
          // we need to set this to false before the next
          // safepoint. This way we ensure that the marking phase
          // doesn't observe any more heap expansions.
          _cm->clear_concurrent_marking_in_progress();
        }
      }

      // We can now guarantee that the global stack is empty, since
      // all other tasks have finished. We separated the guarantees so
      // that, if a condition is false, we can immediately find out
      // which one.
      guarantee(_cm->out_of_regions(), "only way to reach here");
      guarantee(_cm->mark_stack_empty(), "only way to reach here");
      guarantee(_task_queue->size() == 0, "only way to reach here");
      guarantee(!_cm->has_overflown(), "only way to reach here");
      guarantee(!_cm->mark_stack_overflow(), "only way to reach here");

      if (_cm->verbose_low()) {
        gclog_or_tty->print_cr("[%u] all tasks terminated", _worker_id);
      }
    } else {
      // Apparently there's more work to do. Let's abort this task. It
      // will restart it and we can hopefully find more things to do.

      if (_cm->verbose_low()) {
        gclog_or_tty->print_cr("[%u] apparently there is more work to do",
                               _worker_id);
      }

      set_has_aborted();
      statsOnly( ++_aborted_termination );
    }
  }

  // Mainly for debugging purposes to make sure that a pointer to the
  // closure which was statically allocated in this frame doesn't
  // escape it by accident.
  set_cm_oop_closure(NULL);
  double end_time_ms = os::elapsedVTime() * 1000.0;
  double elapsed_time_ms = end_time_ms - _start_time_ms;
  // Update the step history.
  _step_times_ms.add(elapsed_time_ms);

  if (has_aborted()) {
    // The task was aborted for some reason.

    statsOnly( ++_aborted );

    if (_has_timed_out) {
      double diff_ms = elapsed_time_ms - _time_target_ms;
      // Keep statistics of how well we did with respect to hitting
      // our target only if we actually timed out (if we aborted for
      // other reasons, then the results might get skewed).
      _marking_step_diffs_ms.add(diff_ms);
    }

    if (_cm->has_overflown()) {
      // This is the interesting one. We aborted because a global
      // overflow was raised. This means we have to restart the
      // marking phase and start iterating over regions. However, in
      // order to do this we have to make sure that all tasks stop
      // what they are doing and re-initialise in a safe manner. We
      // will achieve this with the use of two barrier sync points.

      if (_cm->verbose_low()) {
        gclog_or_tty->print_cr("[%u] detected overflow", _worker_id);
      }

      if (!is_serial) {
        // We only need to enter the sync barrier if being called
        // from a parallel context
        _cm->enter_first_sync_barrier(_worker_id);

        // When we exit this sync barrier we know that all tasks have
        // stopped doing marking work. So, it's now safe to
        // re-initialise our data structures. At the end of this method,
        // task 0 will clear the global data structures.
      }

      statsOnly( ++_aborted_overflow );

      // We clear the local state of this task...
      clear_region_fields();

      if (!is_serial) {
        // ...and enter the second barrier.
        _cm->enter_second_sync_barrier(_worker_id);
      }
      // At this point, if we're during the concurrent phase of
      // marking, everything has been re-initialized and we're
      // ready to restart.
    }

    if (_cm->verbose_low()) {
      gclog_or_tty->print_cr("[%u] <<<<<<<<<< ABORTING, target = %1.2lfms, "
                             "elapsed = %1.2lfms <<<<<<<<<<",
                             _worker_id, _time_target_ms, elapsed_time_ms);
      if (_cm->has_aborted()) {
        gclog_or_tty->print_cr("[%u] ========== MARKING ABORTED ==========",
                               _worker_id);
      }
    }
  } else {
    if (_cm->verbose_low()) {
      gclog_or_tty->print_cr("[%u] <<<<<<<<<< FINISHED, target = %1.2lfms, "
                             "elapsed = %1.2lfms <<<<<<<<<<",
                             _worker_id, _time_target_ms, elapsed_time_ms);
    }
  }

  _claimed = false;
}
```

update_region_limit()的实现代码如下：
```c++
void CMTask::update_region_limit() {
  HeapRegion* hr            = _curr_region;
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

