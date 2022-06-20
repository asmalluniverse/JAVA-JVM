### jvm优化

[JVM](https://so.csdn.net/so/search?q=JVM&spm=1001.2101.3001.7020)调优目标：使用较小的内存占用来获得较高的吞吐量或者较低的延迟。  

程序在上线前的测试或运行中有时会出现一些大大小小的JVM问题，比如cpu load过高、请求延迟、tps降低等，甚至出现[内存](https://so.csdn.net/so/search?q=内存&spm=1001.2101.3001.7020)泄漏（每次垃圾收集使用的时间越来越长，垃圾收集频率越来越高，每次垃圾收集清理掉的垃圾数据越来越少）、内存溢出导致系统崩溃，因此需要对JVM进行调优，使得程序在正常运行的前提下，获得更高的用户体验和运行效率。

**JVM的性能优化可以分为代码层面和非代码层面。**  

1. 在代码层面

   大家可以结合字节码指令进行优化，比如一个循环语句，可以将循环不相关的代码提取到循环体之外，这样在字节码层面就不需要重复执行这些代码了。  

   对于一些集合的使用，设置默认的大小，并且可以设置为懒加载，真正使用到的时候再去创建，平时经验的一个积累。  

2. 在非代码层面 

   一般情况可以从参数、内存、GC以及cpu占用率等方面进行优化。  

**注意**，JVM调优是一个漫长和复杂的过程，而在很多情况下，JVM是不需要优化的，因为JVM本身已经做了很多的内部优化操作，大家千万不要为了调优和调优。  

### 对于jvm层面的优化必须要掌握的相关知识

1. jvm相关参数
2. jdk提供的常用命令
3. jdk的常用分析工具
4. 内存分析工具
5. GC分析工具

##### jvm参数

jvm参数中主要分为两大类：标准参数-XX和非标准参数-X，标准参数是不会因为java版本变化而改变，非标准参数会随之改变。  

其中-XX参数又分为两类  

1. Boolean类型  
   ```shell 
   格式：‐XX:[+‐]<name> +或‐表示启用或者禁用name属性
   比如：‐XX:+UseConcMarkSweepGC 表示启用CMS类型的垃圾回收器
   ‐XX:+UseG1GC 表示启用G1类型的垃圾回收器
   ```

2. 非Boolean类型(key=value)  
```shell
格式：‐XX<name>=<value>表示name属性的值是value
比如：‐XX:MaxGCPauseMillis=500
```

**其他参数**  
```shell
初始Java堆大小:‐Xms1000M等价于 ‐XX:InitialHeapSize=1000M
最大Java堆大小:‐Xmx1000M等价于 ‐XX:MaxHeapSize=1000M
每个线程stack大小:‐Xss100等价于 ‐XX:ThreadStackSize=100
```

##### 常见参数  
官网: https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
![image-20220403150931127](E:\gupao\jvm性能优化\jvm常见参数)

##### **在哪设置**  
（1）开发工具中设置比如IDEA，eclipse
（2）运行jar包的时候:java -XX:+UseG1GC xxx.jar
（3）中间件比如tomcat，可以在脚本中的进行设置
（4）通过jinfo实时调整某个java进程的参数(参数只有被标记为manageable的flags可以被实时修改)

##### **查看参数**  
（1）启动java进程时添加+PrintFlagsFinal参数
（2）通过jinfo命令查看

### 常用的命令  

1. **jps**  
查看当前运行的java进程

2. **jinfo**
查看当前配置的jvm参数信息

```shell
jinfo ‐flag name PID
jinfo ‐flag <name>=<value> PID
jinfo ‐flags PID
```

3. **jstat**

查看某个java进程的类装载信息  

```shell
jstat ‐class PID 1000 10 查看某个java进程的类装载信息，每1000毫秒输出一次，共输出10次
jstat ‐gc PID 1000 10
```

4. **jstack**

```shell
jstack PID
```
jstack用于打印出Java堆栈信息,常用于检测死锁，或者一个应用进程中某些线程cpu一直居高不下，可以查询进程中某些线程的运行状态。

5. **jmap**
可以生成 java 程序的 dump 文件， 也可以查看堆内对象示例的统计信息、查看 ClassLoader 的信息以及 finalizer 队列。  
生成的dump文件可以通过一些工具进行分析,可以通过jvm命令是的程序出现oom是打印dump日志用于分析具体的原因。  

```shell
1 jmap ‐heap PID
2 jmap ‐dump:format=b,file=heap.hprof PID
3 ‐XX:+HeapDumpOnOutOfMemoryError ‐XX:HeapDumpPath=heap.hprof # 关于dump文件的分析，可以用一些工具，暂时先不讨论
```

### jdk内置的分析工具

**1 jconsole**
jconsole工具是JDK自带的可视化监控工具。查看java应用程序的运行概况、监控堆信息、永久区使用情况、类加载情况等。

**2 jvisualvm**
可以监控某个java进程的CPU，类，线程等连接远程java进程



**第三方工具**
github：https://github.com/alibaba/arthas Arthas 是Alibaba开源的Java诊断工具，采用命令行交互模式，是排查jvm相关问题的利器。

### 内存分析工具

1. **MAT**

Java堆分析器，用于查找内存泄漏
Heap Dump，称为堆转储文件，是Java进程在某个时间内的快照。
它在触发快照的时候保存了很多信息：Java对象和类信息。 

（1）获取dump文件 
```shell
1 jmap ‐dump:format=b,file=heap.hprof 44808   
2  ‐XX:+HeapDumpOnOutOfMemoryError ‐XX:HeapDumpPath=heap.hprof   
```
（2）Histogram：可以列出内存中的对象、对象个数及大小 1 Class Name:类名称，java类名 2 Objects:类的对象的数量，这个对象被创建了多少  

2. **HeapHero**
官网: https://heaphero.io/https://heaphero.io/  
这是一款线上分析工具，直接可以通过浏览器打开，把dump文件导入进行分析，提供了可视化的界面  

### GC分析工具
可以使用不同的参数设置不同的日志文件，比如  
```shell
‐XX:+PrintGCDetails ‐XX:+PrintGCTimeStamps ‐XX:+PrintGCDateStamps ‐Xloggc:D:\gc.log
```
当然每种GC收集器所产生的文件格式不同，一般我们都是通过工具去进行分析  

1. **gcviewer**   

2. **gceasy**  
官网: http://gceasy.io  

3. **gcplot**
官网：https://it.gcplot.com/

```shell
1 docker run ‐d ‐p 8088:80 gcplot/gcplot
2 http://192.168.1.8:8088
```

### 常见的GC优化参数

```shell
参数官网：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html
1 （1）‐XX:MaxTenuringThreshold
2 Sets the maximum tenuring threshold for use in adaptive GC sizing. The largest value is 15. The default value is 15
for the parallel (throughput) collector, and 6 for the CMS collector.
3
4 （2）‐XX:PretenureSizeThreshold
5 超过多大的对象直接在老年代分配，避免在新生代的Eden和S区不断复制
6
7 （3）‐XX:+/‐ UseAdaptiveSizePolicy
8 Enables the use of adaptive generation sizing. This option is enabled by default.
9
10 （4）‐XX:SurvivorRatio
11 默认值为8
12
13 （5）‐XX:ConcGCThreads
14 Sets the number of threads used for concurrent GC. The default value depends on the number of CPUs available to th
e JVM.
15
16 （6）‐Xsssize
17 Sets the thread stack size (in bytes). Append the letter k or K to indicate KB
18
19 （7）‐Xms和‐Xmx
20 两者值一般设置成一样大，防止内存空间进行动态扩容
21
22 （8）‐XX:ReservedCodeCacheSize
23 Sets the maximum code cache size (in bytes) for JIT‐compiled code. Append the letter k or K to indicate
kilobytes, m or M to indicate megabytes, g or G to indicate gigabytes.
```

### 排查方法

```shell
01 启动
java ‐jar ‐Xms1000M ‐Xmx1000M ‐XX:+HeapDumpOnOutOfMemoryError ‐XX:HeapDumpPath=jvm.hprof jvm‐case‐0.0.1‐SNAPSHOT.jar
 
02 使用jmeter模拟并发

03 使用top命令查看
top
top ‐Hp PID

04 jstack查看有没有死锁或者IO阻塞
jstack PID

05 查看堆内存使用情况
jmap ‐heap PID
java ‐jar arthas.jar ‐‐‐> dashboard

06 获取到heap的文件，比如jvm.hprof，用相应的工具来分析，比如heaphero.io
```

### **GC调优**

重点分析G1垃圾收集器  

官网：https://docs.oracle.com/javase/8/docs/technotes/guides/vm/G1.html#use_cases   

（1）50%以上的堆被存活对象占用   
（2）对象分配和晋升的速度变化非常大   
（3）垃圾回收时间比较长  

**G1日志文件**
参数：-XX:+UseG1GC -Xloggc:g1-gc.log  
根据当前是设置的堆内存大小查看他的吞吐量，停顿时间，以及gc的收集次数等参数。  
不断的去调整堆内存大小或者设置最大停顿时间进行观察  
是否需要启动并发GC时堆内存占用百分比：-XX:InitiatingHeapOccupancyPercent=45  
G1用它来触发并发GC周期,基于整个堆的使用率,而不只是某一代内存的使用比例。值为0则表示“一直执行GC循环”. 默认值为 45 (例如, 全部的 45% 或者使用了45%).  

**G1调优最佳实战**
官网:
https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#recommendations  

- 不要手动设置新生代和老年代的大小，只要设置整个堆的大小   
1. G1收集器在运行过程中，会自己调整新生代和老年代的大小  
2. 其实是通过adapt代的大小来调整对象晋升的速度和年龄，从而达到为收集器设置的暂停时间目标，如果手动设置了大小就意味着放弃  

了G1的自动调优。

- 不断调优暂停时间目标
1. 一般情况下这个值设置到100ms或者200ms都是可以的(不同情况下会不一样)，但如果设置成50ms就不太合理。暂停时间设置的太短，
就会导致出现G1跟不上垃圾产生的速度。最终退化成Full GC。

2. 所以对这个参数的调优是一个持续的过程，逐步调整到最佳状态。暂停时间只是一个目标，并不能总是得到满足。
（1）使用-XX:ConcGCThreads=n来增加标记线程的数量  
（2）适当增加堆内存大小  
（3）不正常的Full GC  

- 时候会发现系统刚刚启动的时候，就会发生一次Full GC，但是老年代空间比较充足，一般是由Metaspace区域引起的。可以通过MetaspaceSize适当增加其大家，比如256M。

**1.5 CPU占用率过高**   
1.（1）top  
2.2）top ‐Hp PID  
3.查看进程中占用CPU高的线程id，即tid  
4.3）jstack PID | grep tid（1）top  

**1.6 大并发场景**  
在大并发场景下，除了对JVM本身进行调优之外，还需要考虑业务架构的整体调优    
（1）浏览器缓存、本地缓存、验证码  
（2）CDN静态资源服务器  
（3）集群+负载均衡  
（4）动静态资源分离、限流[基于令牌桶、漏桶算法  
（5）应用级别缓存、接口防刷限流、队列、Tomcat性能优化  
（6）异步消息中间件  
（7）Redis热点数据对象缓存  
（8）分布式锁、数据库锁  
（9）5分钟之内没有支付，取消订单、恢复库存等  
