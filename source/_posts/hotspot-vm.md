# HotSpot VM相关知识 

* HotSpot VM主要由三部分构成：GC、JIT编译器、Runtime
    ![](../images/profile/hotspot.png)
* HotSpot VM的内部架构: 字节码、JIT编译器、本地方法、垃圾回收、方法栈以及堆。
    ![](../images/profile/hotspot-inside.png)
* JVM还有一些内部的线程，包括：

    * VM thread：这个线程是jvm里面的线程母体，是一个单例的对象（最原始的线程）会产生或触发所有其他的线程，这个单个的VM线程是会被其他线程所使用来做一些VM操作（如，清扫垃圾等）。
    * Periodic task thread：该线程是JVM周期性任务调度的线程，它由WatcherThread创建，是一个单例对象。
    * Garbage collection threads： 进行垃圾回收的线程
    * JIT compiler threads: 进行JIT编译的线程
    * Signal dispatcher thread: 当外部jvm命令接收成功后，会交给signal dispather线程去进行分发到各个不同的模块处理命令，并且返回处理结果。

    其他更为具体的可见：[JVM 内部运行线程介绍](http://ifeve.com/jvm-thread/) 
* VM的运行时架构: 类加载器、执行引擎、运行时数据区
    ![](../images/profile/vm-runtime.png)
    * 对于其中的java字节码的格式不再多讲。
    * 类加载器是双亲委派模型，具体的可见: <http://blog.csdn.net/xyang81/article/details/7292380>
        ![](../images/profile/classloader.png)
    * 执行引擎
        ![](../images/profile/execute-engine.png)
    * 运行时数据区
        ![](../images/profile/rundata.png)

