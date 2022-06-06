#### 1.什么是jvm

（1）首先它是一个虚拟机，对应的是物理机 。

（2）他会把java源码编译成class文件，java虚拟机运行的是class文件

（3）class文件也不一定是java语言，也可以是一些其他语言，scala,kotlin,groovy等，我们也可以把它们称为jvm languages. 都是通过jvm进行运行.

（4）jvm帮我们屏蔽底层操作系统的细节，能够把Class文件翻译成不同平台的CPU指令集，实现一次编译到处运行.

![image-20220315212337600](E:\gupao\jvm性能优化\图片\image-20220315212337600.png)



#### 2.如何去设计一个jvm

我们可以类比物理机进行思考，下面是一个物理机的模型

![image-20220315212506905](E:\gupao\jvm性能优化\图片\image-20220315212506905.png)

Class文件类比输入设备

CPU指令集类比输出设备

JVM类比存储器、控制器、运算器等



JVM的架构图：

![image-20220315212857209](E:\gupao\jvm性能优化\图片\image-20220315212857209.png)

#### 3.JVM和HotSpot的关系

jvm是一种java虚拟机的规范，不同的厂商基于该规范进行了实现，例如HotSpot，Jrockit, ibm公司的J9 VM,阿里的taobao vm等。

如果我们在安装了jdk或者jre的时候可以通过java -version命令查看

```shell
C:\Users\grace>java -version
java version "1.8.0_151"
Java(TM) SE Runtime Environment (build 1.8.0_151-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.151-b12, mixed mode)
```

#### 4.JDK JRE JVM之间的关系

官网: https://docs.oracle.com/javase/8/docs/index.html

![image-20220315213822886](E:\gupao\jvm性能优化\图片\image-20220315213822886.png)

JDK：JDK 是 （SDK） 软件开发套件的扩展子集，包括用于开发、调试和监控 Java 应用程序的工具。

JRE：JRE 称为 Java 运行时间环境是 JDK 的一部分，是开发 Java 应用程序的一组编程工具。Java 运行时间环境为执行 Java 应用程序提供了最低要求，并且它由 Java 虚拟机器 （JVM） 核心类和核心类库组成。

JVM：JVM是一种可以执行字节码的虚拟机器。它是 Java 平台的代码执行组件。

#### 5. Class File

java源码是如何转化为Class文件的呢？我们带着这个问题进行下面的学习。

下面是一段java源码文件

```java 
public class User {
 private Integer age;
 private String name="Jack";
 private Double salary=100.0;
 private static String address;
 public void say(){
 System.out.println("Jack Say...");
 }
 public static Integer calc(Integer op1,Integer op2){
 op1=3;
 Integer result=op1+op2;
 return result;
 }

 public static void main(String[] args) {
 System.out.println(calc(1,2));
 }
 }
```

我们可以通过 `javac User.java`编译生成User.class 文件，打开后发现他是一堆16进制的文件。那么我们可以类比物理机进行分析，对于物理机它只是被0101这种二进制的源，可以猜想到这些Classes文件转化为16进制的目的就是为了让jvm能够识别，java源文件是对开发人员友好的，而class文件是为了能让jvm看得懂。接下来我们可以进行验证分析。

![image-20220315225217254](E:\gupao\jvm性能优化\图片\image-20220315225217254.png)

通过官网可以查看Class Fromate:

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html

A `class` file consists of a single `ClassFile` structure:

```java
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

**Analyse**

（1）cafebabe

`magic:The magic item supplies the magic number identifying the class file format`

（2）0000+0034：minor_version+major_version

`16进制的34等于10进制的52，表示JDK的版本为8`

（3）0043：constant_pool_count

`The value of the constant_pool_count item is equal to the number of entries in the constant_pool table plus one`

`16进制的43等于10进制的67，表示常量池中常量的数量是66,因为默认已经加一`

**这里大家可能会有疑问，怎么会有66个常量呢？**

这里的常量代表的是：final修饰的变量以及字面量

（4）cp_info：constant_pool[constant_pool_count-1] ：表示具体有哪些常量，

```java
The constant_pool is a table of structures representing various string constants, class and interface names, field names, and other constants that are referred to within the ClassFile structure and its substructures. The format of each constant_pool table entry is indicated by its first "tag" byte.The constant_pool table is indexed from 1 to constant_pool_count ‐ 1.

字面量：文本字符串，final修饰的常量等
符号引用：类和接口的全限定名、字段名称和描述符、方法名称和描述符
```

（5）The constant pool

官网: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.4

```java
每个常量都应该遵循下面的规范：
All constant_pool table entries have the following general format:

 cp_info {
 u1 tag;
 u1 info[];
 }
```



![image-20220315221355705](E:\gupao\jvm性能优化\图片\image-20220315221355705.png) 

（6）First constant

由0a可以知道第一个常量的类型对应的10进制为10，所以这表示一个方法引用，查找方法引用对应的structure，那么这个方法的引用又是一个什么结构呢？

```java
 CONSTANT_Methodref_info {
 u1 tag; // CONSTANT_Methodref
 u2 class_index; // 代表的是class_index，表示该方法所属的类在常量池中的索引
 u2 name_and_type_index; // 代表的是name_and_type_index，表示该方法的名称和类型的索引

 }
```

经过分析可以得出第一个常量表示的形式为

` #1 = Methodref #16,#35`

（7）Second constant

由08可以知道第一个常量的类型对应的10进制为8，所以这表示一个String引用，查找方法引用对应的structure

```java
 CONSTANT_String_info {
 u1 tag;
 u2 string_index; // 代表的是string_index

 }
```

经过分析可以得出第1、2个常量表示的形式为

```java
 #1 = Methodref #16,#35
 #2 = String #36
```

那么这些常量肯定不是我们进行解析，jdk肯定给我们提供了工具进行解析，我们可以通过javap进行解析

```shell
javap -help
用法: javap <options> <classes>
其中, 可能的选项包括:
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
```

```java
javap -c -p -v User.class
Classfile /E:/User.class
  Last modified 2022-3-15; size 1000 bytes
  MD5 checksum 9b6b8ac3f0605b84e54dbd17a93c049c
  Compiled from "User.java"
public class User
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #16.#35        // java/lang/Object."<init>":()V
   #2 = String             #36            // Jack
   #3 = Fieldref           #15.#37        // User.name:Ljava/lang/String;
   #4 = Double             100.0d
   #6 = Methodref          #38.#39        // java/lang/Double.valueOf:(D)Ljava/lang/Double;
   #7 = Fieldref           #15.#40        // User.salary:Ljava/lang/Double;
   #8 = Fieldref           #41.#42        // java/lang/System.out:Ljava/io/PrintStream;
   #9 = String             #43            // Jack Say...
  #10 = Methodref          #44.#45        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #11 = Methodref          #46.#47        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
  #12 = Methodref          #46.#48        // java/lang/Integer.intValue:()I
  #13 = Methodref          #15.#49        // User.calc:(Ljava/lang/Integer;Ljava/lang/Integer;)Ljava/lang/Integer;
  #14 = Methodref          #44.#50        // java/io/PrintStream.println:(Ljava/lang/Object;)V
  #15 = Class              #51            // User
  #16 = Class              #52            // java/lang/Object
  #17 = Utf8               age
  #18 = Utf8               Ljava/lang/Integer;
  #19 = Utf8               name
  #20 = Utf8               Ljava/lang/String;
  #21 = Utf8               salary
  #22 = Utf8               Ljava/lang/Double;
  #23 = Utf8               address
  #24 = Utf8               <init>
  #25 = Utf8               ()V
  #26 = Utf8               Code
  #27 = Utf8               LineNumberTable
  #28 = Utf8               say
  #29 = Utf8               calc
  #30 = Utf8               (Ljava/lang/Integer;Ljava/lang/Integer;)Ljava/lang/Integer;
  #31 = Utf8               main
  #32 = Utf8               ([Ljava/lang/String;)V
  #33 = Utf8               SourceFile
  #34 = Utf8               User.java
  #35 = NameAndType        #24:#25        // "<init>":()V
  #36 = Utf8               Jack
  #37 = NameAndType        #19:#20        // name:Ljava/lang/String;
  #38 = Class              #53            // java/lang/Double
  #39 = NameAndType        #54:#55        // valueOf:(D)Ljava/lang/Double;
  #40 = NameAndType        #21:#22        // salary:Ljava/lang/Double;
  #41 = Class              #56            // java/lang/System
  #42 = NameAndType        #57:#58        // out:Ljava/io/PrintStream;
  #43 = Utf8               Jack Say...
  #44 = Class              #59            // java/io/PrintStream
  #45 = NameAndType        #60:#61        // println:(Ljava/lang/String;)V
  #46 = Class              #62            // java/lang/Integer
  #47 = NameAndType        #54:#63        // valueOf:(I)Ljava/lang/Integer;
  #48 = NameAndType        #64:#65        // intValue:()I
  #49 = NameAndType        #29:#30        // calc:(Ljava/lang/Integer;Ljava/lang/Integer;)Ljava/lang/Integer;
  #50 = NameAndType        #60:#66        // println:(Ljava/lang/Object;)V
  #51 = Utf8               User
  #52 = Utf8               java/lang/Object
  #53 = Utf8               java/lang/Double
  #54 = Utf8               valueOf
  #55 = Utf8               (D)Ljava/lang/Double;
  #56 = Utf8               java/lang/System
  #57 = Utf8               out
  #58 = Utf8               Ljava/io/PrintStream;
  #59 = Utf8               java/io/PrintStream
  #60 = Utf8               println
  #61 = Utf8               (Ljava/lang/String;)V
  #62 = Utf8               java/lang/Integer
  #63 = Utf8               (I)Ljava/lang/Integer;
  #64 = Utf8               intValue
  #65 = Utf8               ()I
  #66 = Utf8               (Ljava/lang/Object;)V
{
  private java.lang.Integer age;
    descriptor: Ljava/lang/Integer;
    flags: ACC_PRIVATE

  private java.lang.String name;
    descriptor: Ljava/lang/String;
    flags: ACC_PRIVATE

  private java.lang.Double salary;
    descriptor: Ljava/lang/Double;
    flags: ACC_PRIVATE

  private static java.lang.String address;
    descriptor: Ljava/lang/String;
    flags: ACC_PRIVATE, ACC_STATIC

  public User();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #2                  // String Jack
         7: putfield      #3                  // Field name:Ljava/lang/String;
        10: aload_0
        11: ldc2_w        #4                  // double 100.0d
        14: invokestatic  #6                  // Method java/lang/Double.valueOf:(D)Ljava/lang/Double;
        17: putfield      #7                  // Field salary:Ljava/lang/Double;
        20: return
      LineNumberTable:
        line 1: 0
        line 3: 4
        line 4: 10

  public void say();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #9                  // String Jack Say...
         5: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 7: 0
        line 8: 8

  public static java.lang.Integer calc(java.lang.Integer, java.lang.Integer);
    descriptor: (Ljava/lang/Integer;Ljava/lang/Integer;)Ljava/lang/Integer;
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=2
         0: iconst_3
         1: invokestatic  #11                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
         4: astore_0
         5: aload_0
         6: invokevirtual #12                 // Method java/lang/Integer.intValue:()I
         9: aload_1
        10: invokevirtual #12                 // Method java/lang/Integer.intValue:()I
        13: iadd
        14: invokestatic  #11                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        17: astore_2
        18: aload_2
        19: areturn
      LineNumberTable:
        line 10: 0
        line 11: 5
        line 12: 18

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=1, args_size=1
         0: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: iconst_1
         4: invokestatic  #11                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
         7: iconst_2
         8: invokestatic  #11                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        11: invokestatic  #13                 // Method calc:(Ljava/lang/Integer;Ljava/lang/Integer;)Ljava/lang/Integer;
        14: invokevirtual #14                 // Method java/io/PrintStream.println:(Ljava/lang/Object;)V
        17: return
      LineNumberTable:
        line 16: 0
        line 17: 17
}
SourceFile: "User.java"
```

**通过上面反编译出来的文件，我们可以清楚的看到是有66个常量的。**

**那我们进一步分析什么是常量，首先`String name="Jack";`这样定义的一个我们可以理解为常量，`#15 = Class              #51            // User`类名 `#28 = Utf8               say` 方法名都是可以是常量，这些也就是我们说的字面量的意思。**

#### 6.Class文件时如何加载到内存中的呢

我们知道，一个程序的运行是需要分配内存空间的，目的就是把Class文件加载到内存中，让Cpu去运行。上面我们已经分析了Class文件是如何生成的，接下来我们就开始分析Class文件是如何加载到内存空间的，也就是我们常说的JVM中的**类加载机制**。 

![image-20220315220900818](E:\gupao\jvm性能优化\图片\image-20220315220900818.png)

我们通过如上步骤并遵循java文件的一个编译规则生成了class文件。

1.类加载机制
1. 装载 Loading
    找到Class文件所在的全路径，获取此类的二进制字节流，然后装载到内存中(运行时数据区域)，在内存中生成一个代表该类的 Class 对象,作为方法区这些数据的访问入口。

  而类的加载过程使用到了ClassLoader。
  一个项目中肯定有多种类型的Class文件，使用了不同的类加载器进行加载，分别是：
  Bootstrap,Extension,Application,Custom不同类型的加载器会加载不同路径下的文件，各司其职。

  ![image-20220321213838219](E:\gupao\springboot\类加载器)

  但此时又会存在一个问题，我们知道在rt.jar包下存在`java.lang.String`类，该类会通过Bootstrap类加载器进行加载，
  如果我们应用程序中也存在了一个这样的类岂不是混乱了，这个时候双亲委派模式就出现了。
  **双亲委派机制**： 如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的加载器都是如此，因此所有的类加载请求都会传给顶层的启动类加载器，只有当父加载器反馈自己无法完成该加载请求（该加载器的搜索范围中没有找到对应的类）时，子加载器才会尝试自己去加载。
  有了双亲委派机制就可以解决上面的问题了，如果开发者尝试编写一个与rt.jar类库中重名的Java类，可以正常编译，但是永远无法被加载运行。

**如何打破双亲委派模型？**
（1）复写loadClass
（2）Spi机制 dubbo中大量使用了spi机制
（3）OSGR
2. 连接 Linking
	连接又分为三个过程：验证，准备和解析
	（1）验证 Verify:验证类的正确性：文件格式校验，元数据校验，字节码校验等等
	（2）准备 Prepare：**正式为类变量分配内存并设置类变量初始值的阶段**，例如private static int a = 0,此时a变量会被赋值为默认值0.当然不同的类型具有不同的默认值。
	（3）解析 Resoulution：动态的将运行时常量池中的符号引用转化为直接引用
	a.运行时常量池
	这里引出一个问题，**常量池（constant_poll）和运行时常量池（run-time constant_poll）的区别**：他们两个其实是一个东西，只是保存的位置不同，**常量池保存在磁盘中，运行时常量池保存在内存中**。常量池我们在java文件转化为Class文件已经说过了。
	b.符号引用和直接引用
	符号引用：就是Class文件中的16进制
	直接引用：把一些运行时常量池，映射到内存地址中，即对应的物理内存，把符号引用加载到内存中，然后映射到具体的物理内存
	范围：解析的是运行时常量池的内容。
3. 初始化
对变量进行赋值操作，例如privat static int a = 10；

#### 7.jvm运行时数据区域

**如果根据是否是线程内共享的进行划分可以划分为两大类：**
(1)进程生命周期，所有线程共享的：方法区和堆
(2)线程生命周期，线程私有的：java虚拟机栈，本地方法栈和程序计数器

堆，方法区，java虚拟机栈，本地方法栈，程序计数器

- 方法区：
他保存这类的元数据信息和模板信息以及常量池，字段的描述，方法的描述，运行时常量池，如果方法区内存不够也会抛出oom异常

**常量池包括：静态常量池和运行时常量池，静态常量池中保存着符号引用而运行时常量池中是类加载时生成的直接引用，从逻辑分区的角度理解，常量池属于方法区的，但是jdk7之后就放入到堆中进行存储。**

方法区是jvm的一种规范，不同的jdk版本具有不同的名称和实现方式
jdk7会在以前是通过永久代实现的方法区，保存在java内存中
jdk8或者以后是通过元空间进行实现，保存在本地内存或者是系统内存中

我们可以通过jdk提供的插件VisualVM进行查看



**面试题**:
(1)String类型的常量到底存储在哪里呢？ 
java7之前是存储正在方法区的，java7之后移动到堆内存，因为方法区很少发生垃圾回收，而堆内存的垃圾回收比价频繁，所有放到堆中可以方便管理
(2)String a = "jack" String b = "jack" String c = new String("jack") String d = str3.intern();
首先确定一点是java8之后常量池是保存到堆中的， a,b这两个字符串对象指向的都是常量池中的一个对象，c对象是在堆中的，而d对象有点特殊，如果常量池中有jack那么就会指向常量池，否则会创建一个对象放入到堆中

- 堆：
类的实例和数组都在堆中保存，如果空间不足会发生oom，
堆内存中又分为新生代和老年代，新生代又分为Eden和Survivor,Survivor又分为s0,s1,s0和s1任何一个时候都会是保留的区域即，只能是有一个，用于解决空间碎片的问题

如果新生代中对象存活很久就会移动到老年代，
如果s0或者s1空间不够了会向老年代借用空间，也就是空间担保机制
如果老年代空间也不够了也会触发老年代的垃圾回收，如果老年代通过gc后还没有足够的空间进行分配就会触发oom 
如果有一个超大的对象在eden区分配不下会直接分配到老年代

绝大部分对象都是潮生夕死

priva static Object obj = new Object（）； 左边静态成员obj会放入到方法区，右边new的实例会放入到堆中，也就是方法区中的obj的引用会执行方法区中

**什么是TLAB？**
TLAB是虚拟机在堆内存的eden划分出来的一块专用空间，是线程专属的。在虚拟机的TLAB功能启动的情况下，在线程初始化时，虚拟机会为每个线程分配一块TLAB空间，只给当前线程使用，这样每个线程都单独拥有一个空间，如果需要分配内存，就在自己的空间上分配，这样就不存在竞争的情况，可以大大提升分配效率。

比如一个线程的TLAB空间有100KB，其中已经使用了80KB，当需要再分配一个30KB的对象时，就无法直接在TLAB中分配，遇到这种情况时，有两种处理方案：

1、如果一个对象需要的空间大小超过TLAB中剩余的空间大小，则直接在堆内存中对该对象进行内存分配。

2、如果一个对象需要的空间大小超过TLAB中剩余的空间大小，则废弃当前TLAB，重新申请TLAB空间再次进行内存分配。

以上两个方案各有利弊，如果采用方案1，那么就可能存在着一种极端情况，就是TLAB只剩下1KB，就会导致后续需要分配的大多数对象都需要在堆内存直接分配。

如果采用方案2，也有可能存在频繁废弃TLAB，频繁申请TLAB的情况，而我们知道，虽然在TLAB上分配内存是线程独享的，但是TLAB内存自己从堆中划分出来的过程确实可能存在冲突的，所以，TLAB的分配过程其实也是需要并发控制的。而频繁的TLAB分配就失去了使用TLAB的意义。

- java虚拟机栈 
在我们程序中某个方法的执行，肯定是要交给线程执行的，那他是如何进行执行的呢，
假如我们现在有一个a方法，内部会调用b方法和c方法，一个线程先去执行main方法，main方法中调用a方法，那么此时main方法先执行肯定是需要a方法执行完成main方法才能结束，
那么这里和栈的结构复合呢，先进后出的过程。

每个线程的创建都会创建一个java虚拟机栈，每个方法的执行都会创建一个栈帧，放入到java虚拟机栈中，虚拟机栈是可以动态扩展的,如果栈无法申请到足够的内存会出现oom，如何线程请求的栈的深度大于虚拟机所允许的深度就会抛出stackOverflowError。
栈帧中又会包含：局部变量表，操作数栈，动态链接，方法的返回
**方法的返回**:目的就是让后续的方法可以继续执行
**动态链接**：将符号引用转化为直接引用
**局部变量表**；用于保存方法中的局部变量 可以通过javap反编译查看 
**操作数栈**；为了运算

- 本地方法栈 
可以类比java虚拟机栈，他用于保存的是本地方法，一般是c语言写的。

- 程序计数器

 		可以把它看作是当前线程执行的字节码的行号指示器

#### 8.垃圾收集

##### 1.如何确定垃圾对象

（1）引用计数法

对于某个对象而言，只要应用程序中持有该对象的引用，就说明该对象不是垃圾，如果一个对象没有任

何指针对其引用，它就是垃圾。 

**存在的问题：循环引用**

（2）可达性分析

通过GC Root的对象，开始向下寻找，看某个对象是否可达。

能作为GC Root: 类加载器、Thread、虚拟机栈的本地变量表、static成员、常量引用、本地方法栈的变量等。

![image-20220321222244102](E:\gupao\springboot\可达性分析)

##### 2.垃圾收集算法

1. 标记清除

标记：找出内存中需要回收的对象，并且把它们标记出来

![image-20220321222647944](E:\gupao\springboot\垃圾收集算法-标记)

清除：清除掉被标记需要回收的对象，释放出对应的内存空间

![image-20220321222711040](E:\gupao\springboot\垃圾收集算法-清除)

缺点

1 标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不

得不提前触发另一次垃圾收集动作。

​	(1)标记和清除两个过程都比较耗时，效率不高

​	(2)会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前

触发另一次垃圾收集动作

1. 标记整理

效率不高，但是解决了空间碎片的问题

1. 标记复制

缺点

1 (1)存在大量的复制操作，效率会降低

2 (2)空间利用率降低

##### 3.垃圾收集算法选择

Young区：复制算法(对象在被分配之后，可能生命周期比较短，Young区复制效率比较高)

Old区：标记清除或标记整理(Old区对象存活时间比较长，复制来复制去没必要，不如做个标记再清理)

##### 4.垃圾收集器

如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。

Java8 官网： https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/collectors.html#sthref27 

Java17 官网：https://docs.oracle.com/en/java/javase/17/gctuning/availablecollectors.html#GUID-F215A508-9E58-40B4-90A5-74E29BF3BD3C

1. **Serial**

它是一个单线程收集器，可以用于新老年代

新生代: 复制算法

老年代: 标记-整理算法

The serial collector uses a single thread to perform all garbage collection work

![image-20220321223217718](E:\gupao\springboot\serial)

2. ParNew 收集器

ParNew 收集器是 Serial 收集器的多线程版本，除了使用多线程，其他像收集算法,STW,对象分配规则，回收策略与 Serial 收集器完成一样，在底层上，这两种收集器也共用了相当多的代码，它的垃圾收集过程如下

![image-20220321224221544](E:\gupao\springboot\parnew)

3. **Parallel**

它是一个多线程收集器，可用于新老年代

新生代：复制算法

老年代：标记整理算法

The parallel collector is also known as throughput collector, it's a generational collector similar

to the serial collector. The primary difference between the serial and parallel collectors is that

the parallel collector has

multiple threads that are used to speed up garbage collection.

Parallel Scavenge 收集器也是一个使用**复制算法**，**多线程**，工作于新生代的垃圾收集器，看起来功能和 ParNew 收集器一样，它有啥特别之处吗

**关注点不同**，CMS 等垃圾收集器关注的是尽可能缩短垃圾收集时用户线程的停顿时间，而 Parallel Scavenge 目标是达到一个可控制的吞吐量（吞吐量 = 运行用户代码时间 / （运行用户代码时间+垃圾收集时间）），也就是说 CMS 等垃圾收集器更适合用到与用户交互的程序，因为停顿时间越短，用户体验越好，而 Parallel Scavenge 收集器关注的是吞吐量，所以更适合做后台运算等不需要太多用户交互的任务。

Parallel Scavenge 收集器提供了两个参数来精确控制吞吐量，分别是控制最大垃圾收集时间的 -XX:MaxGCPauseMillis 参数及直接设置吞吐量大小的 -XX:GCTimeRatio（默认99%）

除了以上两个参数，还可以用 Parallel Scavenge 收集器提供的第三个参数 -XX:UseAdaptiveSizePolicy，开启这个参数后，就不需要手工指定新生代大小,Eden 与 Survivor 比例（SurvivorRatio）等细节，只需要设置好基本的堆大小（-Xmx 设置最大堆）,以及最大垃圾收集时间与吞吐量大小，虚拟机就会根据当前系统运行情况收集监控信息，动态调整这些参数以尽可能地达到我们设定的最大垃圾收集时间或吞吐量大小这两个指标。自适应策略也是 Parallel Scavenge  与 ParNew 的重要区别！

![image-20220321223336505](E:\gupao\springboot\parallel)

3. **CMS(ConcMarkSweepGC)**

官网：

https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#concurrent_mark_sweep_cms_collector

可以用于老年代

采用标记-清除算法

回收过程：https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

The Concurrent Mark Sweep (CMS) collector is designed for applications that prefer shorter

garbage collection pauses and that can afford to share processor resources with the garbagecollector while the application is

running.

(1) 初始标记 CMS initial mark 标记GC Roots直接关联对象，不用Tracing，速度很快

(2) 并发标记 CMS concurrent mark 进行GC Roots Tracing

(3) 重新标记 CMS remark 修改并发标记因用户程序变动的内容

(4) 并发清除 CMS concurrent sweep 清除不可达对象回收空间，同时有新垃圾产生，留着下次清理称为浮动垃圾

![image-20220321223512420](E:\gupao\springboot\cms)

4. **G1(Garbage First)**

可以用于新老年代

整体上采用标记-整理算法

回收过程：https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

G1 is a mostly concurrent collector. Mostly concurrent collectors perform some expensive work

concurrently to the application. This collector is designed to scale from small machines to large

multiprocessor machines

with a large amount of memory. It provides the capability to meet a pause-time goal with high

probability, while achieving high throughput.

(1) 初始标记（Initial Marking） 标记以下GC Roots能够关联的对象，并且修改TAMS的值，需要暂停用户线程

(2) 并发标记（Concurrent Marking） 从GC Roots进行可达性分析，找出存活的对象，与用户线程并发执行

(3) 最终标记（Final Marking） 修正在并发标记阶段因为用户程序的并发执行导致变动的数据，需暂停用户线程

(4) 筛选回收（Live Data Counting and Evacuation） 对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间制定回收计划

![image-20220321223609154](E:\gupao\springboot\g1)

1 使用G1收集器时，Java堆的内存布局与就与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有

新生代和老年代的概念，但新生代和老年代不再是物理隔离的了，它们都是一部分Region（不需要连续）的集合

2 每个Region大小都是一样的，可以是1M到32M之间的数值，但是必须保证是2的n次幂

3 如果对象太大，一个Region放不下[超过Region大小的50%]，那么就会直接放到H中

4 设置Region大小：‐XX:G1HeapRegionSize=M

5 所谓Garbage‐Frist，其实就是优先回收垃圾最多的Region区域

![image-20220321223718169](E:\gupao\springboot\g1-收集图)

5. **ZGC**

Java11引入的垃圾收集器

不管是物理上还是逻辑上，ZGC中已经不存在新老年代的概念了

会分为一个个page，当进行GC操作时会对page进行压缩，因此没有碎片问题

只能在64位的linux上使用，目前用得还比较少

The Z Garbage Collector (ZGC) is a scalable low latency garbage collector. ZGC performs all

expensive work concurrently, without stopping the execution of application threads.

（1）可以达到10ms以内的停顿时间要求

（2）支持TB级别的内存

（3）堆内存变大后停顿时间还是在10ms以内

![image-20220321223748670](E:\gupao\springboot\zgc收集图)

**垃圾收集器分类**

（1）串行：Serial 适合内存比较小的嵌入式设备

（2）并行：Parallel 更加关注吞吐量：适合科学计算、后台处理等若交互场景

（3）并发：CMS、G1 更加关注停顿时间：适合web交互场景6                            