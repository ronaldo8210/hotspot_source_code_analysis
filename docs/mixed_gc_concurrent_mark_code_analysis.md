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
