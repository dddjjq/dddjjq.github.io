# The JSR-133 Cookbook for Compiler Writers [译]

原文：http://gee.cs.oswego.edu/dl/jmm/cookbook.html

*由于原文访问不了，也可以访问https://www.cnblogs.com/zfyouxi/p/5046432.html阅读原文*

*本文是JSR-133 cookbook的译文，由于本人能力不足，可能部分翻译存在偏差，仅供参考。*



由[Doug Lea](http://gee.cs.oswego.edu/dl)撰写，[JMM mialiling list](http://www.cs.umd.edu/~pugh/java/memoryModel/)提供帮助。

*[dl@cs.oswego.edu](mailto:dl@cs.oswego.edu).*

前言：自从本文第一次撰写10多年以来，许多处理器和语言的内存模型的规格和存在的问题变得更加清晰和易于理解，也有许多没有。为了让本文保持正确，有些涉及到的细节并不完善。如果想获得更多的的信息，请特别参阅Peter Sewell和**[Cambridge  Relaxed Memory Concurrency Group](http://www.cl.cam.ac.uk/~pes20/weakmemory/index.html)**所做的工作。

这是一个非官方的、由JSR-133指定的新Java Memory Model实现指南。本文尽可能提供了一些规则存在的简要背景，而更多专注于他们在指令重排序、多处理屏障指令和原子操作方面对编译器和JVM的影响。并提供了一些推荐的方法用于满足JSR-133的规定。本文之所以非官方是因为它包含了特定处理器属性和规格的解释。我们无法保证这些解释是正确的。同时。处理器规格和实现随时有可能发生改变。

## 重排序

对一个编译器开发者来说，JMM主要由一些规则组成，这些规则禁止获取fields（fields意味着包含许多元素）或者monitor的指令的重排序

### Volatiles和Monitors

volatiles和monitors的主要JMM规则可以视为一个矩阵，其中的单元格表示你无法对特定顺序的字节码相关的指令进行重排序。这张表本身并不是JMM的规格，只是用来观察编译器和运行时系统（runtime system）运行结果的一种有效方法。

![image](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-03-19/image-20220319164300751.png)

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

### Final字段
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

### 内存屏障
编译器和处理器必须遵守重排序规则。由于单处理都保证顺序("as-if-sequential")一致性，所以不需要做额外工作来保证有序性。但是在多处理上，保证一致通常需要发射屏障指令。即使某个编译器优化掉了某个字段的访问（例如有个被加载的值没有用到），屏障依然需要被生成就好像访问依然存在（尽管见下文关于独立优化屏障）TODO
内存屏障仅在一些内存模型中间接地与高级的概念相关联，例如"acquire"和"release"。而且内存屏障并不是“同步屏障”。而且内存屏障和垃圾回收期中的某些“写屏障”并没有关系。内存屏障仅直接控制CPU与其缓存的交互，关于包含有等待被刷新到主存中的写缓冲区，和/或等待加载或预测执行指令的缓冲区。这些影响可能导致缓存、主存和其他处理器的进一步交互。但在JMM中，没有对多处理器之间交互的形式做出要求，只要最终store被全局执行即可，即，在所有处理器上都可见，并且当可见时加载会获取它们。

#### 类别
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

![image](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-03-19/1.png)

加上需要StoreStore屏障的特殊final-field规则如下：

~~~Java
x.finalField = v;
StoreStore;
sharedRef = x;
~~~

这里是一个如何放置屏障的例子：

![image](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-03-19/2.png)

### 数据类别和依赖
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
相反地，如同下面所述，支持数据依赖的处理器提供了一些优化掉LoadLoad和LoadStore屏障指令的机会，否则这些指令需要被发出。（但是，依赖性在任何处理器上都不会自动消除StoreLoad屏障）。
### 与原子指令交互
不同处理器上需要的屏障更进一步地与MonitorEnter和MonitorExit相关。锁定和解锁通常牵涉到原子条件更新操作CAS或LoadLinked/StoreConditional (LL/SC)的使用，这些汉欧volatile读后面跟随一个volatile写的语义。虽然CAS或LL/SC可以较小地满足，一些处理器还支持一些原子指令（例如，一个非条件交换），这些有时可以替代或者与原子条件更新结合。
在所有处理器上，原子操作可以防止读取/更新位置的写入后读取（read-after-write）问题。（否则标准的loop-until-success指令无法按照预期的方式执行）但是处理器的区别在于原子指令是否为其目标位置提供比隐式 StoreLoad 更通用的屏障属性。在某些处理上，这些指令本质上还执行MonitorEnter/Exit需要的屏障，在其他方面，部分或者全部的这些屏障必须特别触发。
Volatiles和Monitors必须被分开来理清它们的影响，给出：
![image](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-03-19/3.png)
加上需要StoreStore屏障的特殊final字段规则：
~~~Java
x.finalField = v; 
StoreStore;
sharedRef = x;
~~~
在这张表中，“Enter”和“Load”是一样的，“Exit”和“Store”是一样的，除非被被原子指令的使用的性质重写，特别地：
* 在任何执行Load的同步块/方法的入口都需要EnterLoad。LoadLoad也是一样，除非在MonitorEnter使用了一个原子指令，且原子指令提供了至少包含LoadLoad的属性，在这个场景下它等价于no-op
* 在任何执行Store的同步代码块/方法的出口，都需要StoreExit。StoreStore也是一样，除非除非在MonitorExit使用了一个原子指令，且原子指令提供了至少包含StoreStore的属性，在这个场景下它等价于no-op
其他专业类型不太可能在编译过程中起作用（见下文），且/或在现在的处理器上退化为no-op操作。例如，在没有干预的Load或Store时，需要EnterEnter来区分嵌套的MonitorEnter。以下示例展示了大多数类型的展示位置：
![image](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-03-19/4.png)
Java层对原子条件更新的操作访问将会通过[JSR-166(并发工具)](http://gee.cs.oswego.edu/dl/concurrency-interest/)在JDK1.5中可用.所以编译器需要产生相关的代码，使用上表的变体，它在语义上折叠了MonitorEnter和MonitorExit，而且有时在实践中，这些Java层的原子更新表现地像是被锁包围。

## 多处理器
这里是一些MPs中普遍使用的处理器列表，同时有提供相关信息的链接。（有些需要在页面上的一些点击，或者免费的注册来获取手册）。这不是个详尽的列表，但是包含了我了解的所有现在和近期多处理器java实现中使用的处理器。下面描述的处理器列表和属性并不明确。某些情况下我只是报告我做的阅读，而且有可能出现漏掉的情况。有些相关的手册对一些JMM相关的属性并不清晰。请帮我使其明确。
比较好的关于屏障和机器相关属性的硬件规格在这里没有列出来的是[Hans Boehm's atomic_ops library](http://www.hpl.hp.com/research/linux/atomic_ops/),[Linux Kernel Source](http://kernel.org/),[Linux Scalability Effort](http://lse.sourceforge.net/)。Linux内核中的屏障与这里讨论屏障的直接对应，并已经移植到多数处理器。有关不同处理器支持的底层模型的描述，见[Sarita Adve et al, Recent Advances in Memory Consistency Models for Hardware Shared-Memory Systems and Sarita Adve and Kourosh Gharachorloo](http://rsim.cs.uiuc.edu/~sadve/)和[Shared Memory Consistency Models: A Tutorial.](http://rsim.cs.uiuc.edu/~sadve/)

sparc-TSO
Ultrasparc 1, 2, 3 (sparcv9) 在TSO(Total Store Order)模式下。Ultra3s只支持TSO模式（在Ultra1/2中没有使用过RMO模式，所以可以忽略。见[UltraSPARC III Cu User's Manual](http://www.sun.com/processors/manuals/index.html)和[The SPARC Architecture Manual, Version 9 .](http://www.sparc.com/resource.htm)

x86（和x64）
Intel 468+，和AMD显然还有其他，2005-2009有过重新规范，但是现在的规格几乎与TSO一样，主要的区别只在于不同缓存模式的支持，和处理边界条件比如未对齐的访问和特殊指令的形式。见    Se[The IA-32 Intel Architecture Software Developers Manuals: System Programming Guide](http://www.intel.com/products/processor/manuals/)和[AMD Architecture Programmer's Manual Programming.ia64](http://www.amd.com/us-en/Processors/DevelopWithAMD/0,,30_2252_875_7044,00.html)

ia64
Itanium,见[Intel Itanium Architecture Software Developer's Manual, Volume 2: System Architecture](http://developer.intel.com/design/itanium/manuals/iiasdmanual.htm)

ppc (POWER)
所有斑斑拥有同样的基础内存模型，但是随着时间推移，一些内存屏障指令的名字和定义有所变动。列出来的版本是从Power4开始；细节见架构手册：[MPC603e RISC Microprocessor Users Manual](http://www.motorola.com/PowerPC/),[MPC7410/MPC7400 RISC Microprocessor Users Manual](http://www.motorola.com/PowerPC/),[Book II of PowerPC Architecture Book](http://www-106.ibm.com/developerworks/eserver/articles/archguide.html),[PowerPC Microprocessor Family: Software reference manual](http://www-3.ibm.com/chips/techlib/techlib.nsf/techdocs/F6153E213FDD912E87256D49006C6541),[Book E- Enhanced PowerPC Architecture](http://www-106.ibm.com/developerworks/eserver/articles/powerpc.html)

arm
7.+版本。见[ARM processor specifications](http://infocenter.arm.com/help/index.jsp)

alpha
21264x以及我认为其他所有的。见[Alpha Architecture Handbook](http://www.alphalinux.org/docs/alphaahb.html)

pa-risc
HP pa-risc实现，见[pa-risc 2.0 Architecture](http://h21007.www2.hp.com/dspp/tech/tech_TechDocumentDetailPage_IDX/1,1701,2533,00.html)

这里是这些处理器如何支持屏障和原子性：
![image](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-03-19/5.png)

注：
* 有些列出来的屏障指令属性比标出来的表格中需要的属性更强，但似乎是达到预期效果最方便的做法。
* 列出的屏障指令是为普通程序内存设计的，但不一定是其他特殊形式/模式的缓存和用于 IO 和系统任务的内存。 例如，在 x86-SPO 上，StoreStorebarriers (“sfence”) 需要与 WriteCombining (WC) 缓存模式一起使用，该模式旨在用于系统级批量传输等。操作系统对程序和数据使用回写模式，这不会 需要 StoreStore 屏障。
* 在 x86 上，任何带锁前缀的指令都可以用作 StoreLoad 屏障。 （Linux 内核中使用的形式是 no-op lock；addl $0,0(%%esp)。）支持“SSE2”扩展的版本（Pentium4 和更高版本）支持 mfence 指令，这似乎更可取，除非带有锁定前缀的指令 无论如何都需要CAS。 cpuid 指令也有效，但速度较慢。
* 在 ia64 上，LoadStore、LoadLoad 和 StoreStore 屏障被折叠成特殊形式的加载和存储指令——没有单独的指令。 ld.acq 充当（load；LoadLoad+LoadStore），st.rel 充当（LoadStore+StoreStore；store）。 这些都没有提供 StoreLoad 屏障——你需要一个单独的 mf 屏障指令。
* 在 ARM 和 ppc 上，可能有机会在存在数据依赖关系的情况下用非基于栅栏的指令序列替换加载栅栏。 [Cambridge Relaxed Memory Concurrency Group](http://www.cl.cam.ac.uk/~pes20/ppc-supplemental/) 在工作中描述了它们应用的序列和案例。
* sparc membar 指令支持所有四种屏障模式以及模式组合。 但在 TSO 中只需要 StoreLoad 模式。 在某些 UltraSparc 上，任何 membar 指令都会产生 aStoreLoad 的效果，无论模式如何。
* 支持“流式 SIMD”SSE2 扩展的 x86 处理器仅在与这些流式指令相关时才需要 LoadLoad“lfence”。
* 尽管 pa-risc 规范没有强制要求，但所有 HP pa-risc 实现都是顺序一致的，因此没有内存屏障指令。
* pa-risc 上唯一的原子原语是 ldcw，它是一种测试和设置形式，您需要使用 HP 自旋锁白皮书中的技术来构建原子条件更新。
* CAS 和 LL/SC 在不同的处理器上采用多种形式，仅在字段宽度方面有所不同，至少包括 4 和 8 字节版本。
* 在 sparc 和 x86 上，CAS 具有隐式的前置和后置完整 StoreLoad 屏障。 sparcv9 架构手册说 CAS 不需要具有 post-StoreLoad 屏障属性，但芯片手册表明它在 ultrasparcs 上具有。
* 在 ppc 和 alpha 上，LL/SC 仅针对正在加载/存储的位置具有隐式障碍，但没有更一般的障碍属性。
* ia64 cmpxchg 指令还具有关于正在加载/存储的位置的隐式障碍，但另外需要一个可选的 .acq（加载加载后+加载存储）或 .rel（存储存储+加载存储前）修饰符。 形式 cmpxchg.acq 可用于 MonitorEnter，cmpxchg.rel 用于 MonitorExit。 在不能保证退出和进入匹配的情况下，可能还需要一个 ExitEnter (StoreLoad) 屏障。
* Sparc、x86 和 ia64 支持无条件交换（swap、xchg）。 Sparc ldstub 是一个单字节的测试和设置。 ia64 fetchadd 返回先前的值并添加到它。 在 x86 上，一些指令（例如 add-to-memory）可以加锁前缀，使它们以原子方式运行。

## 原则
### 单处理器
如果你在生成保证只在单处理器上运行的代码，你可以跳过接下来的部分。因为单处理器保持明显的顺序一致性，除非对象内存是异步可访问的IO内存共享，否则你永远不需要发出屏障。这可能发生在特殊映射的java.nio缓冲区中，但可能仅以影响内部JVM支持代码的方式。此外，可以想象，如果上下文切换不需要足够的同步，则需要一些特殊的屏障。

### 插入屏障
屏障指令适用于在程序执行期间发生的不同类型的访问。 找到一个最小化执行障碍总数的“最佳”位置几乎是不可能的。 编译器通常无法判断给定的加载或存储是否会在另一个需要屏障的之前或之后； 例如，当一个volatile存储后面跟着一个返回时。 最简单的保守策略是假设在为任何给定的加载、存储、锁定或解锁生成代码时，会出现需要“最重”障碍的访问：
1.在每个 volatile 存储之前发出 StoreStore 屏障。
（在 ia64 上，您必须将此和大多数障碍折叠到相应的加载或存储指令中。）
2.在所有存储之后但在从具有final字段的任何类的任何构造函数返回之前发出 StoreStore 屏障。
3.在每个volatile存储之后发出 StoreLoad 屏障。
请注意，您可以改为在每次 volatile 加载之前发出一个，但这对于使用 volatile 的典型程序来说会更慢，其中读取次数大大超过写入次数。 或者，如果可用，您可以将 volatile 存储实现为原子指令（例如 x86 上的 XCHG）并省略屏障。 如果原子指令比 StoreLoad 屏障便宜，这可能更有效。
4.在每次volatile加载后发出 LoadLoad 和 LoadStore 屏障。
在保留数据相关排序的处理器上，如果下一条访问指令取决于加载的值，则无需发出屏障。 特别是，如果后续指令是空值检查或加载该引用的字段，则在加载 volatile 引用后不需要屏障。
5.在每个 MonitorEnter 之前或每个 MonitorExit 之后发出 ExitEnter 屏障。
（如上所述，如果 MonitorExit 或 MonitorEnter 使用提供等效于 StoreLoad 屏障的原子指令，则 ExitEnter 是无操作的。对于在其余步骤中涉及 Enter 和 Exit 的其他人也是如此。）
6.在每个 MonitorEnter 之后发出 EnterLoad 和 EnterStore 屏障。
7.在每个 MonitorExit 之前发出 StoreExit 和 LoadExit 屏障。
8.如果在本质上不提供间接加载排序的处理器上，请在每次加载最终字段之前发出 LoadLoad 屏障。 （此[JMM列表](http://www.cs.umd.edu/~pugh/java/memoryModel/archive/0180.html)发布中讨论了一些替代策略，以及对 [linux 数据相关障碍](http://lse.sourceforge.net/locking/wmbdd.html)的描述。）

许多这些障碍通常会减少到no-op。 事实上，它们中的大多数都减少到no-op，但是在不同的处理器和锁定方案下以不同的方式。 对于最简单的示例，基本符合 x86 上的 JSR-133 或使用 CAS 锁定数量的 sparc-TSO 仅在 volatile 存储之后放置 StoreLoad 屏障。

### 移除屏障
上面的保守策略可能对许多程序来说都是可以接受的。 围绕volatile的主要性能问题发生在与存储关联的 StoreLoad 屏障上。 这些应该是相对少见的——在并发程序中使用 volatile 的主要原因是为了避免在读取时使用锁，这只是在读取大大压倒写入时才会出现的问题。 但至少可以通过以下方式改进这一策略：
* 消除多余的屏障。 上表表明，可以通过以下方式消除障碍：
![image](https://github.com/dddjjq/dddjjq.github.io/raw/main/_posts/image/2022-03-19/6.png)
类似的消除可用于与锁的交互，但取决于锁的实现方式。 在存在循环、调用和分支的情况下执行所有这些操作留给读者作为练习。 :-)
* 重新排列代码（在允许的约束范围内）以进一步消除由于数据依赖于保留此类排序的处理器而不需要的 LoadLoad 和 LoadStore 屏障。
* 移动指令流中发出障碍的点，以改进调度，只要它们仍然出现在所需的间隔内的某个地方。
* 移除不需要的屏障，因为多个线程不可能依赖它们； 例如，可证明仅从单个线程可见的易失性。 此外，当可以证明线程只能存储或只能加载某些字段时，消除一些障碍。 所有这些通常都需要大量的分析。

### 大杂烩
JSR-133 还解决了一些其他问题，这些问题在更专业的情况下可能会带来障碍：
* Thread.start() 需要屏障确保启动的线程在调用点看到调用者可见的所有存储。 相反， Thread.join() 需要屏障以确保调用者看到终止线程的所有存储。 这些通常是由这些结构的实现中需要的同步生成的。
* static final初始化需要 StoreStore 屏障，这些屏障通常包含在遵守 Java 类加载和初始化规则所需的机制中。
* 确保默认的零/空初始字段值通常需要垃圾收集器内的屏障、同步和/或低级缓存控制。
* 在构造函数或静态初始化器之外“神奇地”设置 System.in、System.out 和 System.err 的 JVM 私有例程需要特别注意，因为它们是 JMM 最终字段规则的特殊遗留例外。
* 同样，设置 final 字段的内部 JVM 反序列化代码通常需要 StoreStore 屏障。
* 终结支持可能需要屏障（在垃圾收集器内）以确保 Object.finalize 代码在对象变为未引用之前看到所有字段的所有存储。 这通常通过用于在引用队列中添加和删除引用的同步来确保。
* JNI 例程的调用和返回可能需要屏障，尽管这似乎是一个实现质量问题。
* 大多数处理器都有其他同步指令，主要设计用于 IO 和 OS 操作。 这些不会直接影响 JMM 问题，但可能涉及 IO、类加载和动态代码生成。

## 致谢
Thanks to Bill Pugh, Dave Dice, Jeremy Manson, Kourosh Gharachorloo, Tim Harris, Cliff Click, Allan Kielstra, Yue Yang, Hans Boehm, Kevin Normoyle, Juergen Kreileder, Alexander Terekhov, Tom Deneau, Clark Verbrugge, Peter Kessler, Peter Sewell, Jan Vitek, and Richard Grisenthwaite for corrections and suggestions. 

[*Doug Lea*](http://gee.cs.oswego.edu/dl)
最后修改于: Tue Mar 22 07:11:36 EDT 2011