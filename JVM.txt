young gen预计增量收集失败(old gen没有足够空间来容纳下次young GC晋升对象),full GC时压缩。

不做压缩：
好处：缩短full GC的暂停时间
坏处：CMS的old gen受碎片化问题的困扰

-XX:+CMSScavengeBeforeRemark
在CMS GC前启动一次ygc，目的在于减少old gen对ygc gen的引用，降低remark时的开销-----一般CMS的GC耗时 80%都在remark阶段



为什么CMS快？
1.free-lists。不用花时间整理老年代
2.4个阶段，大部分阶段和用户线程同时进行（会竞争CPU的时间），默认的GC的工作线程为服务器物理CPU核数的1/4；




日志

新生代GC：
[GC(Allocation Failure)ParNew   
GC代表MinorGC。括号内部是GC原因。ParNew是采用的回收器名称。
回收前后新生代大小，堆总大小。

因为这是新生代的GC，所以没有统计老年代的大小。
老年代需要用总堆-新生代计算出来。
回收前后老年代差值，正=新生代这次GC新增量到老年代的大小


老年代GC：
1: Initial Mark
第一个STW。标记老年代中所有的GC Roots；标记被年轻代中活着的对象直接引用的对象。
老年代大小，整个堆大小

2: Concurrent Mark
步骤1中的整个链路遍历。从直接引用遍历到间接引用。和用户线程同时进行。

3: Concurrent Preclean
处理步骤2中已标记但是由用户线程操作又发生改变的对象

4: Concurrent Abortable Preclean

5: Final Remark
第二个STW。标记老年代所有对象。包括改变的和新增的对象

6.Concurrent Sweep
清理对象并回收

7: Concurrent Reset
