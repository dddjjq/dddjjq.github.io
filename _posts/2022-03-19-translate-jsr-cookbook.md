# The JSR-133 Cookbook for Compiler Writers [译]

原文：http://gee.cs.oswego.edu/dl/jmm/cookbook.html

*由于原文访问不了，也可以访问https://www.cnblogs.com/zfyouxi/p/5046432.html阅读原文*

*本文是JSR-133 cookbook的译文，由于本人能力不足，可能部分翻译存在偏差，仅供参考。*



由[Doug Lea](http://gee.cs.oswego.edu/dl)撰写，[JMM mialiling list](http://www.cs.umd.edu/~pugh/java/memoryModel/)提供帮助。

*[dl@cs.oswego.edu](mailto:dl@cs.oswego.edu).*

前言：自从本文第一次撰写10多年以来，许多处理器和语言的内存模型的规格和存在的问题变得更加清晰和易于理解，也有许多没有。为了让本文保持正确，有些涉及到的细节并不完善。如果想获得更多的的信息，请特别参阅Peter Sewell和**[Cambridge  Relaxed Memory Concurrency Group](http://www.cl.cam.ac.uk/~pes20/weakmemory/index.html)**所做的工作。

这是一个非官方的、由JSR-133指定的新Java Memory Model实现指南。本文尽可能提供了一些规则存在的简要背景，而更多专注于他们在指令重排序、多处理屏障指令和原子操作方面对编译器和JVM的影响。并提供了一些推荐的方法用于满足JSR-133的规定。本文之所以非官方是因为它包含了特定处理器属性和规格的解释。我们无法保证这些解释是正确的。同时。处理器规格和实现随时有可能发生改变。

### 重排序

对一个编译器开发者来说，JMM主要由一些规则组成，这些规则禁止获取fields（fields意味着包含许多元素）或者monitor的指令的重排序

#### Volatiles和Monitors

volatiles和monitors的主要JMM规则可以视为一个矩阵，其中的单元格表示你无法对特定顺序的字节码相关的指令进行重排序。这张表本身并不是JMM的规格，只是用来观察编译器和运行时系统（runtime system）运行结果的一种有效方法。

![image-20220319164300751](/Users/dingyunlong/Code/Blog/dddjjq.github.io/_posts/image/2022-03-19/image-20220319164300751.png)

1、Normal Loads包含getfield、getstatic、non-volatile field的一系列load

2、Normal Stores包含putfield、putstatic、non-volatile field的一系列store

3、Volatile Loads包含volatile fields的getfield、getstatic，对多线程可见

4、Volatile Stores包含volatile fields的putfield、putstatic，对多线程可见

5、MonitorEnters（包含synchronized方法进入）用于对多线程可见的锁对象

6、MonitorExits（包含synchronized方法推出）用于对多线程可见的锁对象

Normal Loads和Normal Store在表格中一样，Volatile Loads与MonitorEnter一样，Volatiles Store和MonitorExit一样，所以这里被折叠了（根据需要在后续表格中展开），我们在这里只考虑读写作为一个原子操作的对象，也就是没有位字段、未对齐的访问或者大于平台上面可用字大小的访问。

在表格中，1st和2nd操作之间，可能存在任意数量的其他操作。例如，表格[Normal Store,Volatile Store]中的“No”表示一个volatile store不能与任何后续的volatile store重排序；这些至少可以在多线程语义中起作用。

JSR-133规范的规则只指定了volatiles和monitors可能被多个线程访问的情况，如果编译器以某种方式（通常要付出很多努力）确保lock仅仅被单个线程访问，则lock会被消除。相似地，一个volatile filed如果只在一个线程内部访问的话，它的表现就像一个normal field。还可以进行更细粒度的优化，例如，在确定的时间间隔内依赖于证明多个线程不可访问。

