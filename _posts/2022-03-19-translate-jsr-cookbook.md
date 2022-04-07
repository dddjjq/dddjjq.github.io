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

![image](https://github.com/dddjjq/dddjjq.github.io/blob/main/_posts/image/2022-03-19/image-20220319164300751.png)

1、Normal Loads即getfield、getstatic、non-volatile field的一系列load

2、Normal Stores即putfield、putstatic、non-volatile field的一系列store

3、Volatile Loads即volatile fields的getfield、getstatic，对多线程可见

4、Volatile Stores即volatile fields的putfield、putstatic，对多线程可见

5、MonitorEnters（包含synchronized方法进入）用于对多线程可见的锁对象

6、MonitorExits（包含synchronized方法推出）用于对多线程可见的锁对象

Normal Loads和Normal Store在表格中一样，Volatile Loads与MonitorEnter一样，Volatiles Store和MonitorExit一样，所以这里被折叠了（根据需要在后续表格中展开），我们在这里只考虑读写作为一个原子操作的对象，也就是没有位字段、未对齐的访问或者大于平台上面可用字大小的访问。

在表格中，1st和2nd操作之间，可能存在任意数量的其他操作。例如，表格[Normal Store,Volatile Store]中的“No”表示一个volatile store不能与任何后续的volatile store重排序；这些至少可以在多线程语义中起作用。

JSR-133规范的规则只指定了volatiles和monitors可能被多个线程访问的情况，如果编译器以某种方式（通常要付出很多努力）确保锁仅仅被单个线程访问，则锁会被消除。相似地，一个volatile filed如果只在一个线程内部访问的话，它的表现就像一个normal field。还可以进行更细粒度的优化，例如，在确定的时间间隔内依赖于证明多个线程不可访问。

表格中的空单元格意味着如果访问不依赖于基础的Java语义（如[JLS](http://www.javasoft.com/doc/language_specification/index.html)所示），是可以重排序的。比如，尽管表格没有说明，你仍然不能把同一个序列里面对同个位置的store进行重排序。但是两个分别对不同位置的load和store是可以重排序的，并希望在编译器转换和优化的过程中依然这么做。这包括一些通常不被认为是重排序的案例；例如重用基于已加载的字段计算出的值，而不是重新加载和计算值，而这表现得像是重排序。然而，JMM规范允许消除可避免的依赖转换，然后允许重排序。

在所有例子中，即使被程序员错误的同步，重排序至少要保证Java安全性。所有被观察的字段值必须是预置的零/空值，或者被某些线程写的值。 这通常需要所有堆内存持有的对象在被构造器中使用之前清零,并且不与对这些字段的加载做重排序。做这件事一个好的方法是在garbage collector回收内存的时候清零。请参阅JSR-133有关安全保证的其他比较生僻的案例。

这里描述的规则和属性是关于Java层字段的访问。实际上，这些将另外与内部的簿记（bookkeeping）字段和数据交互，例如对象头、gc表和动态生成代码。

#### Final字段
对final字段的读和写表现地像普通字段有关锁和volatiles，但是有两条额外的重排序规则:
1、（在构造器内部）对final字段的写，如果字段是一个引用，任何对这个final字段的引用，不能与后续的store（构造器之外），也就是持有该引用且对其他线程可见的变量，进行重排序，例如，不能重排序：
~~~Java
x.finalField = v;
...;
sharedRef = x;
~~~
这会在像[内联构造器](http://forums.codeguru.com/showthread.php?%3C/p%3E%3Cp%3E382250-inline-constructor)中发挥作用，'...'代表构造器中剩余的逻辑。你不能把构造器中对final的store移动到构造器之外的某个store后面，这会使得对象对其他线程可见（像之前提到的，这可能要求使用一个屏障）。相似地，像下面的代码，前两个不能与第三个任务进行重排序：
~~~Java
v.afield = 1;
x.finalField = v;
...;
sharedRef = x;
~~~
2、对final字段的初始化加载（即第一次被某个线程使用）无法与包含此字段的引用的初次加载重排序，在如下代码中起作用：
~~~Java
x = sharedRef;
...;
i = x.finalField;
~~~
由于二者之间是依赖关系，编译器永远也不应该重排序，但此规则可能对某些处理器产生影响。

#### 内存屏障
编译器和处理器必须遵守重排序规则。由于单处理都保证顺序("as-if-sequential")一致性，所以不需要做额外工作来保证有序性。但是在多处理上，保证一致通常需要发射屏障指令。即使某个编译器优化掉了某个字段的访问（例如有个被加载的值没有用到），屏障依然需要被生成就好像访问依然存在（尽管见下文关于独立优化屏障）TODO
内存屏障仅在一些内存模型中间接地与高级的概念相关联，例如"acquire"和"release"。而且内存屏障并不是“同步屏障”。而且内存屏障和垃圾回收期中的某些“写屏障”并没有关系。内存屏障仅直接控制CPU与其缓存的交互，关于包含有等待被刷新到主存中的写缓冲区，和/或等待加载或预测执行指令的缓冲区。这些影响可能导致缓存、主存和其他处理器的进一步交互。但在JMM中，没有对多处理器之间交互的形式做出要求，只要最终store被全局执行即可，即，在所有处理器上都可见，并且当可见时加载会获取它们。

##### 类别
几乎所有的处理器都支持至少一种粗粒度的屏障指令，通常被成为栅栏(Fence)，它保证了所有在栅栏之前的store和load会被严格地排序在任何栅栏之后的store和load。这在任何处理器上都通常是最耗时的指令（通常接近、甚至比原子指令更昂贵）。大多数处理器额外地支持更多细粒度的屏障。
内存屏障一个需要点时间来适应的属性是它们在内存的访问之间添加。尽管在一些处理器上叫屏障指令，对的/好的屏障取决于它分开的访问类型。这里有常见的屏障分类，它们很好地映射到现有处理器上的特定指令（有时候是no-ops）

LoadLoad 屏障：
顺序：Load1;LoadLoad;Load2
确保Load1的数据在Load2以及后续的Load之前加载。通常，在执行预测预测加载和/或等待Load可以绕过等待Store的无序处理的处理器上需要显式的LoadLoad屏障。单处理器保证通常避免load排序。屏障相当于no-ops。

StoreStore 屏障：
顺序：Store1;StoreStore;Store2
确保Store1的数据对其他处理器可见要先于Store2相关的数据以及后续的Store指令。通常，在没有其他方式保证从写缓冲区和/或缓存到其他处理器或主存的处理器上需要StoreStore屏障。

LoadStore 屏障：
顺序：Load1;LoadStore;Store2
确保Load1的数据先于Store2相关的数据以及后续的Store指令被刷新。LoadStore屏障只有在那些等待Store指令可以绕过Load的乱序处理器上才需要.

StoreLoad 屏障:
顺序：Store1;StoreLoad;Load2
确保Store1的数据对其他处理可见（即，刷新到主存）先于Load2访问的数据以及后续加载的Load指令。StoreLoad屏障使用Store1的数据而不是使用被最近被其他处理器Store到同个位置的数据，以此来避免后续Load的错误加载。因为如此，在下面讨论的处理器上。StoreLoad仅在将存储与在屏障之前存储的数据的相同位置的后续加载分开时才是必须的（译注：Store1代表存储，Load2包含对屏障之前Store的数据的加载）。StoreLoad屏障几乎在所有最近的多处理器上都需要，而且通常是最昂贵的类型。昂贵的一部分原因是必须禁止通常绕过缓存来实现对写缓冲区的Load机制。这有可能通过缓冲区全刷新以及其他可能的停顿来实现，
在下面讨论的所有处理器上面，事实证明执行了StoreLoad的指令也含有其他三种屏障的效果，所以StoreLoad用作通用(但通常非常昂贵)的栅栏。（这是经验上的事实，并不是必须的），反之不成立。其他屏障的组合并不能经常相当于StoreLoad。
接下来的表格展示了屏障是怎么与JSR-133的排序规则相对应的：
//TODO

加上需要StoreStore屏障的特殊final-field规则如下：

~~~Java
x.finalField = v;
StoreStore;
sharedRef = x;
~~~

这里是一个如何放置屏障的例子：
//TODO
##### 数据类别和依赖
在某些处理器上，对LoadLoad和LoadStore屏障的需求与对依赖指令的排序保证相关。在有些（大多数）处理器上，依赖前面load的值的load或者store操作在排序时不需要屏障，这通常出现在两种情况下，间接：
~~~Java
Load x;
Load x.filed;
~~~
和控制：
~~~Java
Load x;
if (predicate(x)) 
    Load or Store y;
~~~
没有间接排序的处理器尤其需要对最开始通过共享引用获得的final-field引用的访问设置屏障（有点绕口）(Processors that do NOT respect indirection ordering in particular require barriers for final field access for references initially obtained through shared references)：
~~~Java
x = sharedRef; 
... ; 
LoadLoad;
i = x.finalField;
~~~