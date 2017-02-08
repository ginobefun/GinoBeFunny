---
title: 《深入理解Java虚拟机》读书笔记1：Java技术体系、Java内存区域和内存溢出异常
date: 2017-01-04 09:11:00
tags: [Java, JVM, 读书笔记, 垃圾收集, 内存溢出]
categories: JVM
link_title: deep_in_jvm_notes_part1
toc_number: false
---
国内JVM相关书籍NO.1，Java程序员必读。读书笔记第一部分对应原书的前两章，主要介绍了Java的技术体系、Java虚拟机的发展历史、Java运行时区域的划分、对象的创建和访问以及内存溢出的实战。
<!-- more -->

# Part 1: 走进Java
## 第一章 走进Java
### 概述
Java的优点
- 结构严谨、面向对象
- 摆脱平台的束缚，一次编写到处运行
- 提供了相对安全的内存管理和访问机制
- 实现了热点代码检测和运行时编译及优化
- 一套完善的应用程序接口以及无数的第三方类库

### Java技术体系

Sun官方所定义的Java技术体系包括：
- Java程序设计语言
- 各种硬件平台上的Java虚拟机
- Class文件格式
- Java API类库
- 来自商业机构和开源社区的第三方Java类库

JDK是用于支持Java开发的最小环境，JRE是支持Java程序运行的标准环境，整个Java体系如下所示：

![Java技术体系](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/java_tech_structure.png)

### Java发展史
![Java技术发展](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/java_timeline.png)

- JDK 1.0: Java虚拟机、Applet、AWT等；
- JDK 1.1：JAR文件格式、JDBC、JavaBeans、RMI、内部类、反射；
- JDK 1.2：拆分为J2SE/J2EE/J2ME、内置JIT编译器、一系列Collections集合类；
- JDK 1.3：JNDI服务、使用CORBA IIOP实现RMI通信协议、Java 2D改进；
- JDK 1.4：正则表达式、异常链、NIO、日志类、XML解析器和XSLT转换器；
- JDK 1.5：自动装箱、泛型、动态注解、枚举、可变参数、遍历循环、改进了Java内存模型、提供了java.util.concurrent并发包；
- JDK 1.6：提供动态语言支持、提供编译API和微型HTTP服务器API、虚拟机优化（锁与同步、垃圾收集、类加载等）；
- JDK 1.7：G1收集器、加强对Java语言的调用支持、升级类加载架构；
- JDK 1.8：Lambda表达式等；

### Java虚拟机发展史
- Sun Classic/Exact VM：Classic VM是第一款商用虚拟机，纯解析器方式来执行Java代码，如果要使用JIT编译器就必须进行外挂，解析器和编译器不能配合工作，编译器执行效率非常差；Exact VM是Sun虚拟机团队曾在Solaris平台发布的虚拟机，支持两级即时编译器、编译器和解释器混合工作、使用准确内存管理（虚拟机可以知道内存中某个位置的数据具体是什么类型），但很快就被HotSpot VM所取代；
- Sun HotSpot VM：Sun JDK和OpenJDK所带的虚拟机，目前使用范围最广；继承了前两款虚拟机的优点，还支持热点代码探测技术（通过计数器找出最具编译价值的代码）；2006年Sun公司宣布JDK包括HotSpot VM开源，在此基础上建立OpenJDK；
- Sun Mobile-Embedded VM/Meta-Circular VM：还有一些Sun开发的面对移动和嵌入式发布的和实验性质的虚拟机；
- BEA JRockit/IBM J9 VM：JRockit VM号称是世界上最快的Java虚拟机，专注于服务器端应用，不包含解析器实现，全部靠即时编译器编译执行；J9 VM定位于HotSpot比较接近，主要目的是作为IBM公司各种Java产品的执行平台；
- Azul VM/BEA Liquid VM：特定硬件平台专有的高性能虚拟机；
- Apache Harmony/Google Android Dalvik VM：Apache Harmony包含自己的虚拟机和Java库，但没有通过TCK认证；Dalvik VM是Android平台的核心组成部分，其并没有遵循Java虚拟机规范，不能直接执行Class文件，使用的是寄存器架构而不是JVM常见的栈架构；
- Microsoft JVM及其他：微软曾经是Java技术的铁杆支持者，开发过Windows下性能最好的Java虚拟机，但后来被Sun起诉终止其发展；

### 展望Java技术的未来
- 模块化
- 混合语言
- 多核并行
- 进一步丰富语法
- 64位虚拟机

### 实战：自己编译JDK
- 下载OpenJDK：https://jdk7.java.net/source.html
- 系统需求：Ubuntu 64位、5GB的磁盘、1G内存；
- 构建编译环境：需要Bootstrap JDK(JDK6以上)/Ant(1.7.1以上)/GCC。
 

    sudo apt-get install build-essential gawk m4 openjdk-6-jdk libasound2-dev libcups2-dev libxrender-dev xorg-dev xutils-dev x11proto-print-dev binutils libmotif3 libmotif-dev ant


- 进行编译：设置环境变量、make sanity检查、make编译、复制到JAVA_HOME、编辑env.sh

### 在IDE工具中进行源码调试
NetBeans（支持C/C++开发的版本）

### 本章小结
本章介绍了Java技术体系的过去、现在以及未来的一些发展趋势，并独立编译一个OpenJDK 7的版本。

# Part 2 自动内存管理机制
## 第二章 Java内存区域与内存溢出异常
### 概述
对于Java程序员来说，在虚拟机自动内存管理机制下，不需要为new操作去写配对的delete/free代码，不容易出现内存泄漏。但是如果出现内存泄漏问题，如果不了解虚拟机的机制，便难以定位。

### 运行时数据区域
![运行时数据区域](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/jvm_runtime_data_area.png)

#### 程序计数器
- 一块较小的内存，可以看作是当前线程所执行的字节码的行号指示器；
- 在虚拟机概念模型（各种虚拟机实现可能不一样）中，字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令；
- 程序计数器是属于线程私有的内存；
- 如果执行的是Java方法，该计数器记录的是正在执行的虚拟机字节码指令的地址；如果是Native方法则为空；

#### Java虚拟机栈
- Java虚拟机栈也是线程私有的；
- 描述的是Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机中入栈到出栈的过程；
- 局部变量表存放了编译器可知的各种基本数据类型、对象引用和returnAddress类型；其所需的内存空间在编辑期完成分配，不会再运行期改变；
- 可能存在两种异常：StackOverflowError和OutOfMemoryError；

#### 本地方法栈
- 与虚拟机栈非常相似，只不过是为虚拟机使用到的Native方法服务；
- 可能存在两种异常：StackOverflowError和OutOfMemoryError；

#### Java堆
- Java堆是被所有线程共享的，在虚拟机启动时创建；
- 此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这分配；
- 是垃圾收集器管理的主要区域，可以分为新生代和老年代；
- 可以物理不连续，只要逻辑上是连续的即可；
- 如果堆中没有内存完成实例分配也无法再扩展时，会抛出OutOfMemoryError异常；

#### 方法区
- 是线程共享的区域；
- 用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据；
- 该区域对于垃圾收集来说条件比较苛刻，但是还是非常有必要要进行回收处理；
- 当无法满足内存分配需求时，将抛出OutOfMemoryError异常；

#### 运行时常量池
- 是方法区的一部分；
- Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译器生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放；
- Java虚拟机规范要求较少，通常还会把翻译出来的直接引用也存储在此；
- 另外一个重要特征是具备动态性，可以在运行期间将新的常量放入池中，如String的intern方法；
- 可能存在的异常：OutOfMemoryError；

#### 直接内存
- 并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域；
- JDK 1.4的NIO引入了基于通道（Channel）和缓冲区（Buffer）的IO方法，可以使用Native函数库直接分配对外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作以提升性能；

### HotSpot虚拟机对象探秘
进一步了解虚拟机内存中数据的其他细节，比如它们是如何创建、如何布局以及如何访问的。下面以虚拟机HotSpot和常用的内存区域Java堆为例，深入探讨HotSpot虚拟机在Java堆中对象分配、布局和访问的全过程。

#### 对象的创建
- 虚拟机遇到一条new指令时，先检查指令的参数是否能在常量池中定位到一个类的符号，并且检查这个符号引用代码的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程；
- 接下来虚拟机将为新生对象分配内存。对象所需的内存大小在类加载完成后便完全确定，为对象分配空间等同于把一块确定大小的内存从Java堆中划分出来。在使用Serial、ParNew等带Compact过程的收集器时，系统采用的分配算法是指针碰撞（内存绝对规整，只要通过指针作为分界点标识）；而使用CMS这种基于Mark-Sweep算法收集器时，通常使用空闲列表（内存不规整，通过维护一个列表记录那块内存是可用的）；
- 另外一个需要考虑的并发下的线程安全问题，有两种方案：一是分配内存空间的动作进行同步处理（实际上虚拟机采用CAS配上失败重试的方式保证更新操作的原子性）；二是为每个线程分配一小块内存（称为本地线程分配缓冲，TLAB），各个线程独立分配，只有TLAB用完需要分配新的才需要同步锁定，虚拟机通过-XX:+/-UseTLAB参数来设定；
- 内存分配完后，虚拟机将分配到的内存空间都初始化为零值（不包括对象头），这保证了对象的实例字段在Java代码中可以不赋值就直接使用，程序能访问到这些字段数据类型对应的零值；
- 接下来设置对象的对象头（Object Header）信息，包括对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象GC分代年龄等；
- 接着执行<init>方法，把对象按照程序员的意愿进行初始化，这样一个真正可用的对象才算完全产生出来；
- HotSpot解释器的代码片段：略

#### 对象的内存布局
- 对象在内存中存储的布局可以分为3块区域：对象头（Object Header）、实例数据（Instance Data）和对齐填充（Padding）；
- 对象头包括两部分信息：第一部分用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等；另一部分是类型指针，即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例（并不是所有虚拟机都必须在对象数据上保留类型指针）。另外如果对象是一个Java数组，对象头中还必须有一块用于记录数组长度的数据。
- 实例数据部分是真正存储的有效信息，也是在代码中所定义的各种类型字段内容。无论是父类继承的还是子类中定义的都需要记录下来。这部分存储的顺序会受到虚拟机分配策略参数和字段在Java源码中定义顺序的影响。
- 对齐填充不是必然存在的，主要是由于HotSpot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍。

#### 对象的访问定位
- 栈上的reference类型在虚拟机规范中只规定了一个指向对象的引用，并没有定义这个引用应该通过何种方式去定位、访问堆栈对象的具体位置，目前主流的方式方式有句柄和直接直接两种。
- 通过句柄：Java堆中划出一块内存作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息。其最大好处就是reference存储的是稳定的句柄地址，在对象被移到（垃圾收集时移到）只改变实例数据指针，而reference不需要修改；

![通过句柄访问对象](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/object_access_reference_1.png)

- 通过直接指针：Java堆对象的布局中必须考虑如果放置访问类型数据的相关信息，而reference中存在的直接就是对象地址。其最大好处在于速度更快，节省了一次指针定位的时机开销。HotSpot采用该方式进行对象访问，但其他语言和框架采用句柄的也非常常见。

![通过直接指针访问对象](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/object_access_reference_2.png)

### 实战：OutOfMemoryError异常
- 通过代码验证Java虚拟机规范中描述各个运行时区域存储的内容；
- 在实际遇到内存溢出异常时，能根据异常的信息快速判断是哪个区域内存溢出；

#### Java堆溢出
![Java堆溢出](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/outofmemory_jvm_heap.png)

解决思路：先通过内存映像分析工具对dump出来的堆转储快照进行分析，先分清楚是内存泄漏还是内存溢出；如果是内存泄漏，进一步查看泄漏对象到GC Roots的引用链，从而确认为什么无法回收；如果是内存溢出，则应当检查虚拟机堆参数（-Xmx与-Xmx）或检查是否存在对象生命周期过长、持有状态时间过长的情况；

#### 虚拟机栈和本地方法栈溢出
- HotSpot不区分虚拟机栈和本地方法栈；
- StackOverflowError和OutOfMemoryError存在互相重叠的地方；
- 栈容量由-Xss参数设定；

![虚拟机栈溢出](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/stackoverflow_jvm_stack.png)

虚拟机的默认参数对于通常的方法调用（1000~2000层）完全够用，通常根据异常的堆栈日志就可以很容易定位问题。

#### 方法区和运行时常量池溢出
对于这个区域的测试，基本思路是运行时产生大量的类去填满方法区（比如使用反射和动态代理），这里我们借助CGLib直接操作字节码运行时产生大量的动态类（很对主流框架如Spring、Hibernate都会采用类似的字节码技术）。在这里需要特别注意垃圾回收的状况。

![借助CGLib使方法区出现内存溢出异常1](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/outofmemory_java_methodarea_part1.png)
![借助CGLib使方法区出现内存溢出异常2](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/outofmemory_java_methodarea_part2.png)

#### 本机直接内存溢出
![本机直接内存溢出1](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/outofmemory_jvm_direct.png)
![本机直接内存溢出2](http://oi46mo3on.bkt.clouddn.com/10_deep_in_jvm/outofmemory_jvm_direct_part2.png)

DirectMemory导致的内存溢出，在Heap Dump里不会看见明显的异常。如果发现OouOfMemory之后Dump文件很小，程序又使用了NIO，那就可以检查下是否这方面的原因。

### 本章小结
学习了虚拟机的内存是如何划分的，对象是如何创建、布局和访问的，哪部分区域、什么样的代码和操作可能导致内存的溢出异常。

**系列读书笔记**
- [《深入理解Java虚拟机》读书笔记1：Java技术体系、Java内存区域和内存溢出异常](http://ginobefunny.com/post/deep_in_jvm_notes_part1)
- [《深入理解Java虚拟机》读书笔记2：垃圾收集器与内存分配策略](http://ginobefunny.com/post/deep_in_jvm_notes_part2)
- [《深入理解Java虚拟机》读书笔记3：虚拟机性能监控与调优实战](http://ginobefunny.com/post/deep_in_jvm_notes_part3)
- [《深入理解Java虚拟机》读书笔记4：类文件结构](http://ginobefunny.com/post/deep_in_jvm_notes_part4)
- [《深入理解Java虚拟机》读书笔记5：类加载机制与字节码执行引擎](http://ginobefunny.com/post/deep_in_jvm_notes_part5)
- [《深入理解Java虚拟机》读书笔记6：程序编译与代码优化](http://ginobefunny.com/post/deep_in_jvm_notes_part6)
- [《深入理解Java虚拟机》读书笔记7：高效并发](http://ginobefunny.com/post/deep_in_jvm_notes_part7)
