
RSet在内部使用Per Region Table(PRT)记录分区的引用情况。

在PRT中将会以三种模式记录引用：稀少、细粒度、粗粒度（粗粒度的PRT只是记录了引用数量，需要通过整堆扫描才能找出所有引用，因此扫描速度也是最慢的）

Refine线程  RefineCardTableEntryClosure
