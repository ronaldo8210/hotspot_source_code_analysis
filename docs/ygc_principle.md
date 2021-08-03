[YGC原理](#jump_1)

[一次YGC过程的示例](#jump_2)

## <span id="jump_1">YGC原理</span>

在JVM运行期间，当需要进行一次YGC时，需要从根对象出发，去遍历存活的年轻代对象，将存活的年轻代对象复制到Survivor区，并清理掉已被复制的对象和从根引用不可达的对象。

根对象包括：

1. Java线程栈的引用变量指向的对象；

2. 老年代对象，以老年代为根时，因为老年代太大，不去扫描老年代，而是从年轻代的Remembered Set出发，看Remembered Set是否记录有老年代的Card。因为Remembered Set记录的是引用者的Card，所以扫描年轻代所有的Remembered Set即可达到目的，找到所有老年代对象到年轻代的引用。是空间换时间的算法。

## <span id="jump_2">一次YGC过程的示例</span>

假设一次YGC即将开始时，JVM内部运行状态如下：

- 年轻代1中的Card 1分配了对象obj_1、obj_2、obj_3，Card 2分配了对象obj_4、obj_5、obj_6

- 年轻代2中的Card 3分配了对象obj_7、obj_8、obj_9，Card 4分配了对象obj_10、obj_11

- 老年代1中的Card 5分配了对象obj_12、obj_13、obj_14，Card 6分配了对象obj_15

- 老年代2中的Card 7分配了对象obj_16、obj_17

- obj_1引用了obj_5、obj_10，obj_5引用了obj_7，obj_11引用了obj_13，obj_15引用了obj_8，obj_16引用了obj_15，obj_17引用了obj_8

- 年轻代2的Remembered Set记录了Card6的卡表索引和Card7的卡表索引，老年代1的Remembered Set记录了Card7的卡表索引

- JVM运行了3个Java业务线程，线程栈1中的引用变量ref_1引用年轻代对象obj_1，线程栈2中的引用变量ref_4引用年轻代对象obj_11，线程栈3中的引用变量ref_5引用老年代对象obj_12

上述内存布局如下图所示：

<img src="../images/ygc_phase_1.png" width="100%" height="100%"/>

根据YGC的规则，完成YGC后，内存布局如下图所示：

<img src="../images/ygc_phase_2.png" width="100%" height="100%"/>
