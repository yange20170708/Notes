
### Young GC 全程stop the world
- 各个线程到达安全点的等待时间；`-XX:+PrintGCApplicationStoppedTime -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1`
- 从 GC Root 扫描对象，进行标记的时间；`-XX:+PrintReferenceGC`
- 存活对象 Copy 到 Survivor 以及晋升 Old Gen 到的时间；
- GC 日志的时间；

#### 排查ygc时间过长 checklist：
- 检查新生代GC的时候存活对象占比：配合stat -gctuil pid 1000
- 检查StringTable hash槽，扫描StringTable：-XX:+PrintStringTableStatistics
- 检查各类引用：-XX:+PrintReferenceGC

##### 附：
[一次 Young GC 的优化实践（FinalReference 相关）](https://www.jianshu.com/p/79d4a0516f11)


#### 内存泄漏 Memory Leak
- 可达，即在有向图中，存在通路可以与其相连；    
- 无用，即无用但无法释放GC回收  

#### 常见内存泄漏案例
- 静态引用 list.add(object);object=null;由于list还在引用，object不会被回收。
- - 解决：list.clear();list=null;    
- 单例引用外部实例
- native资源(jvm通过jni暴漏出来的功能)。如直接内存，数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接必须手动close
- - 常见:FinalReference过多
- - 解决:`-XX:+ParallelRefProcEnabled`


<hr />
young gen预计增量收集失败(old gen没有足够空间来容纳下次young GC晋升对象),full GC时压缩。

不做压缩：
好处：缩短full GC的暂停时间
坏处：CMS的old gen受碎片化问题的困扰

-XX:+CMSScavengeBeforeRemark
在CMS GC前启动一次ygc，目的在于减少old gen对ygc gen的引用，降低remark时的开销-----一般CMS的GC耗时 80%都在remark阶段

### 为什么CMS快？
1. `free-lists`。不用花时间整理老年代
2. 4个阶段，大部分阶段和用户线程同时进行（会竞争CPU的时间），默认的GC的工作线程为服务器物理CPU核数的1/4；

### 日志

### 新生代GC：
[GC(`Allocation Failure`)`ParNew`   
GC代表MinorGC。括号内部是GC原因。ParNew是采用的回收器名称。
回收前后新生代大小，堆总大小。

因为这是新生代的GC，所以没有统计老年代的大小。
老年代需要用总堆-新生代计算出来。
回收前后老年代差值，正=新生代这次GC新增量到老年代的大小

### 老年代GC：
1: Initial Mark
`第一个STW`。标记老年代中所有的`GC Roots`；标记被年轻代中活着的对象`直接引用`的对象。
老年代大小，整个堆大小

2: Concurrent Mark
步骤1中的整个链路遍历。`从直接引用遍历到间接引用`。和用户线程同时进行。

> - [GC[ParNew (`promotion failed`)`Concurrent Mode Failure`：


> `promotion failed`-`Concurrent Mode Failure` :MinorGC时，Survivor晋升老年代，老年代内存足,但是有标记整理算法<b>碎片</b>，导致连续内存不足。（后续会导致提前进行CMS Full GC，带来STW。
> 解决:
> 调整Survivor，减少MinorGC
> 调整碎片，减少过早FullGC：
> -XX:UseCMSCompactAtFullCollection -XX:CMSFullGCBeforeCompaction=0。即：CMS在进行x次Full GC（标记清除）之后进行一次标记整理）


> - `Concurrent Mode Failure` :业务线程直接将对象放入老年代，老年代内存不足
> 解决：
> XX:CMSInitiatingOccupancyFraction

> - `XX:+UseCMSInitiatingOccupancyOnly` 不设置会导致gc回收没有固定规律。
> - 设置后和`XX:+CMSInitiatingOccupancyFraction`搭配使用

3: Concurrent Preclean
处理步骤2中已标记但是由用户线程操作又`发生改变`的对象

4: Concurrent Abortable Preclean

5: Final Remark
`第二个STW`。标记老年代所有对象。包括改变的和新增的对象

6.Concurrent Sweep
清理对象并回收

7: Concurrent Reset
