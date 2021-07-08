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
        double mark_step_duration_ms = G1ConcMarkStepDurationMillis;

        the_task->do_marking_step(mark_step_duration_ms,
                                  true  /* do_termination */,
                                  false /* is_serial*/);

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
      } while (!_cm->has_aborted() && the_task->has_aborted());
    }
    the_task->record_end_time();
    guarantee(!the_task->has_aborted() || _cm->has_aborted(), "invariant");

    SuspendibleThreadSet::leave();

    double end_vtime = os::elapsedVTime();
    _cm->update_accum_task_vtime(worker_id, end_vtime - start_vtime);
  }
};
```

每个并发标记worker线程对各自处理的HeapRegion中存活的Java对象打标的具体过程在CMTask::do_marking_step()，主要代码如下：
```c++

```
