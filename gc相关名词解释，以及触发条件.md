### GC类型 

新生代进行一次垃圾清理：Young GC = Minor GC 

老生代进行一次垃圾清理 Old GC = Major GC

Full GC = Young GC + Old GC + Metaspace GC (Young GC + Old GC)



1. 如果单个 Survivor 区已经被占用了 50% (对应虚拟机参数: -XX:TargetSurvivorRatio)，那么较高复制次数的对象也会被晋升至老年代。

2. Full GC 就是收集整个堆，包括新生代，老年代，永久代(在JDK 1.8及以后，永久代会被移除，换为metaspace)等收集所有部分的模式。
   

新生代和老年代的比例默认是 1：2，该值可以通过参数 `–XX:NewRatio` 来指定 

新生代又分为Eden、survivor（from+to）,Eden:From:To=8:1:1( 可以通过参数 `–XX:SurvivorRatio` 来设定 )

**什么时候会发生垃圾收集**

GC是由JVM自动完成的，根据JVM系统环境而定，所以时机是不确定的。

当然，我们可以手动进行垃圾回收，比如调用System.gc()方法通知JVM进行一次垃圾回收，但是具体什么时刻运行也无法控制。

也就是说System.gc()只是通知要回收，什么时候回收由JVM决定。但是不建议手动调用该方法，因为GC消耗的资源比较大。 

（1）当Eden区或者S区不够用了： Young GC 或 Minor GC

（2）老年代空间不够用了： Old GC 或 Major GC

（3）方法区空间不够用了： Metaspace GC

（4）System.gc()

Full GC=Young GC+Old GC+Metaspace GC



### 准寻的规则

- 一定量的尽可能避免gc
- 尽可能发生young gc 
- 如果避免不了发生major gc,那么jvm内部会有一个算法：每一次major gc之前，会触发一次young gc
- 通过上面jvm提供的规则我们可以直到在进行old gc的时候就等同于发生了full gc 



### 如何优化GC？

1. 尽量不要创建过大的对象或数组。可以通过jvm参数进行设置，如果过大的对象直接分配到老年代，减少新生代中对象在s区中来回复制
2. 通过虚拟机的 -Xmn 参数适当调大新生代的大小，让对象尽量在新生代中被回收掉。
3. 通过 -XX:MaxTenuringThreshold 参数调大对象进入老年代的年龄，让对象尽量在新生代中被回收掉。 