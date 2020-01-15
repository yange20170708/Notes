
# 为什么有强引用以外的其他辅助引用？
特殊需求：
> - 弹性回收（内存不够时，对象需要降级）
> - - 缓存数据在内存紧张时自动释放掉空间防止oom、
> - 补充回收（gc只能回收jvm资源，一旦涉及native资源，则需要手动回收）
> - - 直接内存对象回收之前需自动释放掉其占用的堆外内存，
> - - socket对象被回收之前关闭连接，
> - - 文件流对象被回收之前自动关闭打开的文件等操作


# Reference 四种状态
> 只需要通过成员变量next和queue来判断 
#### 1.Active:next = null; 活动状态 
> 对象存在强引用状态,还没有被回收;
#### 2.Pending:next = this;queue = ReferenceQueue;  待处理状态。gc回收第一次触发，识别f类
> 垃圾回收器将没有强引用的Reference对象放入到pending队列中，等待ReferenceHander线程处理（前提是这个Reference对象创建的时候传入了ReferenceQueue，否则的话对象会直接进入Inactive状态）；
#### 3.Enqueued:queue = ReferenceQueue.ENQUEUED; 入列状态。将f类解绑，执行f类的finalize方法
> ReferenceHander线程将pending队列中的对象取出来放到ReferenceQueue队列(单向后进先出链表)里；
#### 4.Inactive:next = this;queue = ReferenceQueue.NULL; 可回收状态 
> 处于此状态的Reference对象可以被回收,并且其内部封装的对象也可以被回收掉了
 

##### 引用

##### SoftReference
##### WeakReference
##### PhantomReference
##### FinalReference  常见用处：释放native资源。缺点：容易内存泄漏
> - java.lang.ref.Finalizer
> -  - 实现了finalize方法的类会生成这个对象
> -  - Finalizer继承FinalReference类,private,由jvm自动封装
> -  - Finalizer有两个队列，一个是unfialized,一个是f-queue队列；

> - 一个对象实现类object的finalize方法，
> - jvm加载时，标记为f类
> - 对象分配好空间或完成构造函数时，jvm调用Finalizer.register，new一个Finalizer对象，将f类注册到全局f-queue（ReferenceQueue）里，将Finalizer对象加入unfialized里;
> - GC时，调用Finalizer.runFinalizer方法，将Finalizer对象从f-queue里取出，native调用f对象的finalize方法，下次GC时就可以将其关联的f对象回收

> - 类的修饰有很多，比如final，abstract，public等，如果某个类用final修饰，我们就说这个类是final类，
> - 类似的，finalizer表示这个类是一个finalizer（f）类。
> - f对象因为Finalizer的引用而变成了一个临时的强引用
> - f对象至少经历两次GC才能被回收
> - Reference对象是在gc的时候来处理的，如果没有触发GC就没有机会触发Reference引用的处理操作

