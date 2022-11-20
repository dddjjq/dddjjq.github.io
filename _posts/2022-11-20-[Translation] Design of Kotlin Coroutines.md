# [译] Kotlin协程的设计-coroutine创建的过程是什么样的？

原文：[Design of Kotlin Coroutines](https://proandroiddev.com/design-of-kotlin-coroutines-879bd35e0f34)

*注：这是在medium上面读到的一篇关于Kotlin协程的文章，个人觉得写的很好，对Kotlin协程的理解有很大的帮助，故此翻译。由于个人水平问题，可能存在错漏的地方，仅供参考。*

*部分术语直接使用英文单词，对开发者来说，可能这样更直观一点*

*很多地方的英文单词和汉语术语是可以互换的*

*coroutine-协程*

*coroutineContext-协程上下文*

*context-上下文*

*continuation-延续*



我们很多人都使用coroutines，但是谁知道coroutine创建的过程是什么样的？这篇blog的结构如下所示：

1、定义

2、CPS — 延续传递风格（Continuation Passing Style）

3、Kotlin coroutine原则

- 3.1 Coroutine构建

- - 3.1.1 launch()

- - 3.1.2 start()

- - 3.1.3 invoke()

- - 3.1.4 startCoroutineCancellable()
  
- -  3.1.5 resumeWithCancellable()

- - 3.1.6 resumeWith()

- - 3.1.7 invokeSuspend()

- - 3.1.8 Summary of coroutine construction
- 3.2 字节码分析

# 1、定义
coroutine是什么？

> 一个coroutine是一个可挂起的计算的实例，从概念上讲，它类似于一个thread，从某种意义上讲，它包含一个代码块与其他代码同时执行，然而一个coroutine并不和任何thread绑定。它有可能在某个线程挂起执行，然后在另一个线程恢复

在异步编程里面，tasks在不同的线程里面并行执行，而且不需要等待其他tasks的结束。不恰当的多线程使用有可能导致CPU的高度使用，或者增加CPU的运行周期，从而导致你的应用性能急剧下降，因此，线程是一种昂贵的资源。Coroutines是thread的轻量级替代品。

**suspend函数是什么？**

> suspend函数是可以被started、paused和resume的一种函数。关于suspend函数需要被记忆的最重要的其中一点是：它们只允许被其他的coroutine或者另一个suspend函数调用

![](https://miro.medium.com/max/1400/1*vo7XdbX5O2WrT7yMNPWsNQ.jpeg)

当一个coroutine被挂起时，其运行的线程可以被其他coroutines使用。coroutine的延续不一定是同一个线程。在这里，我们得出的结论是，我们可以在少数线程上面同时运行许多coroutines。接下来你会看到它是怎么工作的。

**suspending vs Non-suspending/常规 函数**

- 一个suspend函数包含一个或多个挂起点（suspention Points），一个常规函数没有挂起点。一个挂起点代表了方法体内的一个声明，代表在方法体内，可以暂停执行，以在后面的一段时间后恢复

- 普通函数无法直接调用suspend函数，因为它不支持挂起点

- suspend函数可以调用普通函数，因为它没有挂起点

这是一个suspend函数例子：

~~~kotlin
suspend fun functionA(): String {
     return  "hello"
}
~~~

正如我们上面说的，我们不能在普通函数里面调用suspend函数。让我们反编译一下functionA()，看看里面发生了什么。

使用Tools->Kotlin->Show Kotlin Bytecode.

~~~Java
@Nullable
public static final Object functionA(@NotNull Continuation $completion) {
  return "hello";
}
~~~

![](https://miro.medium.com/max/1400/1*VuWjEjoDEAD-68O12tEp4A.jpeg)

你可能疑惑Continuation参数是怎么来的，以及是什么意思。现在我们会解释它是怎么来的，一会我们会展示它是什么。每个suspend函数都会经过一个**CPS-Continuation passing style**转换。

我们可以通过接下来的例子观察coroutine的挂起。想象我们正在玩一款游戏，我们想要suspend（暂停），然后继续在我们离开的地方玩。在这个场景下，我们将游戏存档，当我们想继续的时候，我们可以在档案中的上一个暂停点恢复。当过程被suspend时，它会返回一个Continuation（延续）。

# 2、 CPS — Continuation Passing Style（延续传递风格）

在一个有n个参数(p_1,p_2,p_3,...,p_n)以及返回类型为T的CPS转换suspend函数中，它们获得了一个`Continuation<T>`类型的参数p_n+1，返回类型是`Any?`，返回类型被转换为`Any?`，原因是：

- 如果方法返回了一个结果，我们得到T
- 如果方法suspend（挂起），它返回信号值`COROUTINE_SUSPENDED`，它代表挂起状态

![](https://miro.medium.com/max/1400/1*-3IF7S5Sng-Z1q4-Y60L4A.jpeg)

# 3、Kotlin coroutine原则
让我们用一个示例来浏览coroutine过程，了解coroutine是如何创建的，并解释器流程。

~~~Kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        lifecycleScope.launch {
            val randomNum = getRandomNum() // suspension point #1
            val sqrt = getSqrt(randomNum.toDouble()) // suspension point #2
            log(sqrt.toString())
        }
    }

    private suspend fun getRandomNum(): Int {
        delay(1000)
        return (1..1000).shuffled().first()
    }

    private suspend fun getSqrt(num: Double): Double {
        delay(2000)
        return sqrt(num)
    }

    private fun log(text: String) {
        Log.i(this@MainActivity::class.simpleName, text)
    }
}
~~~

在上面的例子中，我们有两个suspend函数。`getRandomNum()`函数获取一个从1到1000的随机数，将其传递给另一个suspend函数`getSqrt()`来获取其根。

我们在第六行添加一个断点，在debug模式中运行来获得执行coroutine体之前的堆栈。我们想通过这个来看coroutine创建过程是什么样的。

![](https://miro.medium.com/max/1400/1*ewTP61lN6BGZePrCPW43Qg.jpeg)

这是一个堆栈：

~~~
invokeSuspend:75, MainActivity$startCoroutine$1 (me.aleksandarzekovic.exploringcoroutines)
resumeWith:33, BaseContinuationImpl (kotlin.coroutines.jvm.internal)
resumeCancellableWith:266, DispatchedContinuationKt (kotlinx.coroutines.internal)
startCoroutineCancellable:30, CancellableKt (kotlinx.coroutines.intrinsics)
startCoroutineCancellable$default:25, CancellableKt (kotlinx.coroutines.intrinsics)
invoke:110, CoroutineStart (kotlinx.coroutines)
start:126, AbstractCoroutine (kotlinx.coroutines)
launch:56, BuildersKt__Builders_commonKt (kotlinx.coroutines)
launch:1, BuildersKt (kotlinx.coroutines)
launch$default:47, BuildersKt__Builders_commonKt (kotlinx.coroutines)
launch$default:1, BuildersKt (kotlinx.coroutines)
startCoroutine:75, MainActivity (me.aleksandarzekovic.exploringcoroutines)
onCreate:71, MainActivity (me.aleksandarzekovic.exploringcoroutines)
...
~~~

执行顺序是：

![](https://miro.medium.com/max/1400/1*WHjUyEwSIklIbjK_F0E2bA.jpeg)

## 3.1 coroutine构建

### 3.1.1 launch()

![](https://miro.medium.com/max/1400/1*ca_HZEcMbHWHfaTAaaQKLQ.jpeg)

> 启动一个新的coroutine，无需阻塞当前线程，并返回类型为Job的coroutine引用

~~~Kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
~~~

**CoroutineScope**

>为新的coroutines定义一个作用域。每个协程构建器(像launch,async，等等)都是CoroutineScope的一个扩展函数，并继承其CoroutineContext用来自动传播其所有的元素（Element）和取消(cancellation)

CoroutineScope是仅有一个`coroutineContext`属性的接口。

~~~Kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
~~~

>注意：在我们的例子中，我们使用`launch` coroutine构建器，并且我们将通过它解释创建coroutine。其他coroutine构建器相对这个是非常直观的

`launch` coroutine构建器包含三个参数：

- context - 附加到CoroutineScope.coroutineContext的协程上下文
- start - coroutine启动选项，默认值是CoroutineStart.DEFAULT
- block - 将会在提供的作用域中执行的协程代码。编译器在编译阶段为suspend lambda函数生成一个继承SuspendLambda且实现Function2的内部类：

~~~Kotlin
final class me/aleksandarzekovic/exploringcoroutines/MainActivity$onCreate$1 
  extends kotlin/coroutines/jvm/internal/SuspendLambda 
    implements kotlin/jvm/functions/Function2
~~~

这意味着kotlin为每个coroutine在编译阶段生成了一个SuspendLambda匿名内部类。这是**协程体类(coroutine body class)**。内部实现了两个方法：
- `invokeSuspend（）` - 包含我们协程体中的代码，它在内部处理状态值。coroutine中最重要的状态变化逻辑，被称之为状态机，正是被包含在它里面的
- `create()` - 方法接收一个Continuation对象，然后创建并返回一个协程体类的对象。

**Coroutine state machine（协程状态机）**

>Kotlin以状态机来实现挂起函数，因为这样的实现不需要特定的运行时支持。这决定了Kotlin coroutines必须要显式的suspend标记（方法染色）：编译器必须知道哪些方法可能是suspend的，以将其变为状态机

状态机的状态对应着挂起点，在我们的例子中：

![](https://miro.medium.com/max/1400/1*b2Y4_huMCvcsrJI1Ivw6zw.jpeg)

该代码块中有三个方法：

- L0:直到挂起点#1
- L1:直到挂起点#2
- L2:直到结束

![](https://miro.medium.com/max/1400/1*3eX6d8Bd7LPUJxYaFd6pWg.jpeg)

带有这三个状态的状态机生成的伪代码像这样：

~~~Kotlin
// The initial state of the state machine
int label = 0
A a = null
B b = null

void resumeWith(Object result) {
    if (label == 0) goto L0
    if (label == 1) goto L1
    if (label == 2) goto L2
    else throw IllegalStateException("call to 'resume' before 'invoke' with coroutine")

  L0:
    // result is expected to be `null` at this invocation
    label = 1
    // 'this' is passed as a continuation
    result = getRandomNum(this) 
    if (result == COROUTINE_SUSPENDED) return
  L1:
    A a = (A) result
    label = 2
    val result = getSqrt(a, this) // 'this' is passed as a continuation
    if (result == COROUTINE_SUSPENDED) return
  L2:
    B b = (B) result
    log(String.valueOf(b))
    label = -1 // No more steps are allowed
    return
}    
~~~

coroutine的逻辑被封装在`invokeSuspend`方法中，我们之前有提到。

SuspendLambda的继承关系是：

![](https://miro.medium.com/max/1400/1*Wscb3gP0K5PcgqKEz0Vdjg.jpeg)

**Continuation(延续)**

>代表一个挂起点之后的延续，返回一个类型为T的值

~~~Kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
~~~

Continuations非常重要，因为它允许coroutine的延续。每个suspend函数都与`Continuation`生成的子类型相关联，它处理挂起的实现

- `context` - 与该延续对应的coroutine的上下文
- `resumeWith()` - 用来在挂起点之间传递结果。它以最后一个挂起点的结果（或异常）调用并恢复coroutine的执行

**BaseContinuationImpl**

`BaseContinuationImpl` 的主要源码如下：

~~~Kotlin
internal abstract  class  BaseContinuationImpl (...) {
     // Implement resumeWith of Continuation
     // It is final and cannot be overridden!
    public  final override fun resumeWith (result: Result<Any?>) {
        // ...
        val  outcome  = invokeSuspend(param)
        // ...
    }
    // For implementation 
    protected abstract fun invokeSuspend (result: Result<Any?>) : Any?
}
~~~

`invokeSuspend()` - 是一个抽象方法，是在编译阶段生成的协程体类中实现的

`resumeWith()` 方法的实现通常被`invokeSuspend()`方法调用

**ContinuationImpl**

ContinuationImpl继承了BaseContinuationImpl类。它的功能是通过拦截器生成一个DispatchedContinuation对象，这也是一个Continuation。我们将在3.2.4节讨论它。

让我们继续分析`launch()`的方法体

![](https://miro.medium.com/max/900/1*iWEGaCGp1APeDF-XxZooWg.png)

**newCoroutineContext()**

>newCoroutineContext - 为一个新的coroutine创建上下文。如果没有指定其他dispatcher或者ContinuationInterceptor，它将会使用Dispatchers.Default ，并且为 调试设施（如果开关为开的话）和 JVM上的copyable-thread-local设施 添加额外的可选支持。

`newCoroutineContext`是CoroutineScope的一个扩展函数。它的功能是合并CoroutineScope继承的context和通过参数传进来的context，并返回一个新的context。

![](https://miro.medium.com/max/1400/1*WvIDWwfBTREHu3ENANlMvg.jpeg)

让我们简要介绍一下`CoroutineContext`

**CoroutineContext**

CoroutineContext是一个由Elements对象组成的不可变的索引联合集（immutable indexed union set），Element对象比如CoroutineName、CoroutineId、CoroutineExceptionHandler、ContinuationInterceptor、CoroutineDispatcher、Job。此集合中的每个元素都包含一个独特的`Key`。

想象我们想控制我们的coroutine运行在某个线程或者线程池上。取决于我们想把任务运行在主线程上，或者任务是CPU或者IO型的，我们将会使用`dispatcher`。

`Dispatchers`是coroutine提供的线程调度器，使用其来切换线程或者是指定coroutine运行的线程。Dispatchers中有四种类型的调度器：

- Dispatchers.Default
- Dispatchers.IO
- Dispatchers.Main
- Dispatchers.Unconfined

这些调度器都是CoroutineDispatcher（协程调度器）。我们在上面提到，协程调度器是CoroutineContext的一个元素，这意味着这些调度器都是CoroutineContext的元素。

我们说CoroutineContext类似于包含不同元素的集合。我们可以通过adding/removing一个元素，或者合并两个已有的context来创建一个新的context。plus（+）操作符作为Set.plus的扩展，返回两个context的组合，加号右边的元素会替代左边相同key的元素。

没有任何元素的上下文可以被创建为EmptyCoroutineContext的一个实例。

![](https://miro.medium.com/max/1400/1*sAYxX03FsB7thm3QiNSfWA.jpeg)

~~~Kotlin
import kotlinx.coroutines.*

fun main() {
    val coroutineName = CoroutineName("C#1") + CoroutineName("C#2")
    println(coroutineName)
}

// result
// CoroutineName(C#2)
~~~

~~~Kotlin
import kotlinx.coroutines.*

fun main() {
    val coroutineContext = CoroutineName("C#1") + Dispatchers.Default
    println(coroutineContext)
}

// result
// [CoroutineName(C#1), Dispatchers.Default]
~~~

~~~Kotlin
import kotlinx.coroutines.*

fun main() {
    val firstCoroutineContext = CoroutineName("C#1") + Dispatchers.Default
    println(firstCoroutineContext)
    
    val secondCoroutineContext = Job() + Dispatchers.IO
    println(secondCoroutineContext)
    
    val finalCoroutineContext = firstCoroutineContext + secondCoroutineContext
    println(finalCoroutineContext)
}

// result
// [CoroutineName(C#1), Dispatchers.Default]
// [JobImpl{Active}@39a054a5, Dispatchers.IO]
// [CoroutineName(C#1), JobImpl{Active}@39a054a5, Dispatchers.IO]
~~~

![](https://miro.medium.com/max/1400/1*xJhm7Le6Ve6au6glw5yidw.jpeg)

一个CoroutineContext从来不会被重写，而是与一个已有的合并。现在我们了解了一些CoroutineContext的知识，我们可以回到刚才离开的地方，launch构建器中的newCoroutineContext。

让我们定义不同的context，便于我们理解：

- scope context - `CoroutineScope`中定义的context
- (passed context)传入的context - 构建器函数接收一个`CoroutineContext`实例作为第一个参数
- (parent context)父上下文 - 构建器中的suspend代码块参数拥有一个`CoroutineScope`接收器，该接收器本身提供了`CoroutineContext`，这个context并不是一个新的context!(译注：这里有些绕，指的context是StandaloneCoroutine里面的context)
- 新的coroutine创建它自己的子Job实例（使用此作业的上下文作为其parent），并将父context+它自己的Job定义为它的coroutine context（子context）。我们在稍后更详细地看到我们是怎么得出这个结论的。
  
(译注：上面这一段有些绕口，其实就是`public final override val context: CoroutineContext = parentContext + this
`这段代码)

![](https://miro.medium.com/max/1400/1*aiPCcv88MVuN9-kVtBWViw.jpeg)

在定义了新的上下文（父上下文）之后，我们可以创建一个新的coroutine：

![](https://miro.medium.com/max/900/1*iWEGaCGp1APeDF-XxZooWg.png)

**StandaloneCoroutine**
我们使用新的上下文（父上下文）来创建coroutine。`start`参数的默认值是CoroutineStart.DEFAULT，在这个例子中，我们创建了`StandaloneCoroutine`（继承自AbstractCoroutine），返回值是一个Job。StandaloneCoroutine是一个协程对象。

>注意：如果我们把start设置为lazy，我们会创建一个LazyStandaloneCoroutine，LazyStandaloneCoroutine是继承自StandaloneCoroutine的，StandaloneCoroutine是继承自AbstractCoroutine的。

~~~Kotlin
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = true, active = active) {
    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}
~~~

StandaloneCoroutine中，只有`handleJobException`方法被重写，用来处理没有被父协程处理的异常。这里调用的`start`方法是父类AbstractCoroutine的方法。

~~~Kotlin
@InternalCoroutinesApi
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
  
  // ...
  
  public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
      start(block, receiver, this)
  }
  
  // ...
}
~~~

AbstractCoroutine类实现了`JobSupport`类和`Job`,`Continuation`以及`CoroutineScope`接口。AbstractCoroutine类主要负责协程的恢复和结果的返回。

![](https://miro.medium.com/max/1400/1*TqaiW-GBonjPIb3HFAQSeQ.jpeg)

**JobSupport**
JobSupport是`Job`的特殊实现。AbstractCoroutine可以被当做一个`Job`来控制coroutine的生命周期，它可以实现`Continuation`接口，也可以被当做`Continuation`使用。

**Job**

AbstractCoroutine的上下文是我们通过参数传递（父context）加上当前的coroutine，由于我们知道AbstractCoroutine既是Job又是CoroutineScope，我们知道我们的协程上下文包含一个Job元素。这个context是协程上下文(子context)。

~~~Kotlin
@InternalCoroutinesApi
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {

    // ...

    /**
     * The context of this coroutine that includes this coroutine as a [Job].
     */
    @Suppress("LeakingThis")
    public final override val context: CoroutineContext = parentContext + this
    
    // ...
}
~~~

![](https://miro.medium.com/max/1400/1*mVuiEpHvwxsxJ0b6INDv9Q.jpeg)

堆栈中的第二步是**coroutine.start(start, coroutine, block)**

### 3.1.2 start()

![](https://miro.medium.com/max/1400/1*Wxts56_bxvx_rkfIRQzP0A.jpeg)

>使用给定的代码块和启动策略启动协程。该方法在协程中最多被执行一次。

~~~Kotlin
@InternalCoroutinesApi
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
  
  // ...
  
  public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
      start(block, receiver, this)
  }
  
  // ...
}
~~~

`AbstractCoroutine#start()`方法调用`start()`方法。CoroutineStart是一个枚举类，invoke()方法是内部重写的。在这个场景中，`start()`方法会调用`CoroutineStart.invoke()`方法。

### 3.1.3 invoke()

![](https://miro.medium.com/max/1400/1*PfuX_I4YAWfwsv0_Z8KrRg.jpeg)

>定义协程构建器的启动选项，它在launch、async以及其他协程构建器中使用`start`参数传递

~~~Kotlin
@InternalCoroutinesApi
public operator fun <T> invoke(block: suspend () -> T, completion: Continuation<T>): Unit =
    when (this) {
        DEFAULT -> block.startCoroutineCancellable(completion)
        ATOMIC -> block.startCoroutine(completion)
        UNDISPATCHED -> block.startCoroutineUndispatched(completion)
        LAZY -> Unit // will start lazily
}
~~~

CoroutineStart是一个有四个类型的枚举类：

- DEFAULT - 根据其上下文，立即执行coroutine
- LAZY - 懒启动coroutine，或者被需要的时候
- ATOMIC - 根据上下文原子化（一种无法被取消的方式）执行coroutine
- UNDISPATCHED - 立即执行，直到遇到当前线程的第一个挂起点

在这里，DEFAULT用于作为实例

### 3.1.4 startCoroutineCancellable()

![](https://miro.medium.com/max/1400/1*IxhD8H4rzx3cuu0X06p6Zg.jpeg)

>以一种可取消的方式使用此方法启动coroutine，以便在等待调度时可以取消。

~~~Kotlin
/**
 * param completion is AbstractCoroutine
 * return a Continuation
 */
internal fun <R , T> (suspend ( R ) -> T).startCoroutineCancellable(receiver: R, completion: Continuation<T>) =   
  runSafely(completion) {   
      createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellableWith(Result.success(Unit))   
  }
~~~

>`runSafely()`运行给定的代码块，如果发生异常，则将completion完成。理由：我们在自己的调度器上异步运行协程的时候会调用startCoroutineCancellable。因此，如果在coroutine启动的过程中调度器抛出异常，协程永远无法被结束，所以我们应该将调度器异常视为原因并恢复completion（译注：这里的completion指的都是传递的那个参数）

~~~Kotlin
private inline fun runSafely(completion: Continuation<*>, block: () -> Unit) {
    try {
        block()
    } catch (e: Throwable) {
        completion.resumeWith(Result.failure(e))
    }
}
~~~

startCoroutineCancellable的实现是一个链式调用，让我们看下这个链：

1、createCoroutineUnintercepted() - 是调用协程体的扩展函数，协程体被编译成SuspendLambda的子类（协程体类），所以这就是BaseContinuationImpl。

~~~Kotlin
// kotlin/libraries/stdlib/jvm/src/kotlin/coroutines/intrinsics/IntrinsicsJvm.kt

@SinceKotlin("1.3")
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(receiver, probeCompletion)
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
~~~

`create()`方法创建一个协程体的实例，在这里我们获得了协程类的实例。

~~~Kotlin
@NotNull
public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
    Intrinsics.checkNotNullParameter(completion, "completion");
    Function2 var3 = new <anonymous constructor>(completion);
    return var3;
 }
 ~~~

 2、intercepted() - 使用ContinuationInterceptor来拦截continuation(延续).

 ~~~Kotlin
 public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
~~~

`this`是一个继承自ContinuationImpl的协程体的实例

~~~Kotlin
public fun intercepted(): Continuation<Any?> =
    intercepted
      ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
~~~

如果intercepted为空，通过使用context中指定的interceptor拦截协程体类，并返回包装后的协程体对象。

`context[ContinuationInterceptor]` - 获取集合中的调度器(译注：这里指的应该是拦截器)，并调用interceptContinuation()。
interceptContinuation()方法用来将协程体的延续包装成一个DispatchedContinuation。

~~~Kotlin
// CoroutineDispatcher

public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> 
  = DispatchedContinuation(this, continuation)
~~~

**DispatchedContinuation**

DispatchedContinuation代表了协程体的延续对象，并持有线程调度器。它的功能是使用线程调度器来调度协程体到指定的线程上执行。

注意在构造器中，它需要一个调度器和continuation，并且它实现了`Continuation<T>`和`DispatchedTask<T>`。

3、resumeCancellableWith() - Continuation的一个扩展函数

~~~Kotlin
@InternalCoroutinesApi
public fun <T> Continuation<T>.resumeCancellableWith(
    result: Result<T>,
    onCancellation: ((cause: Throwable) -> Unit)? = null
): Unit = when (this) {
    is DispatchedContinuation -> resumeCancellableWith(result, onCancellation)
    else -> resumeWith(result)
}
~~~

如果this不是拦截或者包装的协程体对象，`resumeWith(result)`将会被调用。

否则，如果this是拦截的且用DispatchedContinuation类包装的对象，`resumeCancellableWith(result, onCancellation)`会被调用

~~~Kotlin
inline fun resumeCancellableWith(
        result: Result<T>,
        noinline onCancellation: ((cause: Throwable) -> Unit)?
    ) {
        val state = result.toState(onCancellation)
        if (dispatcher.isDispatchNeeded(context)) {
            _state = state
            resumeMode = MODE_CANCELLABLE
            dispatcher.dispatch(context, this)
        } else {
            executeUnconfined(state, MODE_CANCELLABLE) {
                if (!resumeCancelled(state)) {
                    resumeUndispatchedWith(result)
                }
            }
        }
    }
~~~

如果你查看CoroutineDispatcher的源码，`dispatcher.isDispatchNeeded()`的返回值总是true，只有Dispatchers.Unconfined会重写并返回false。

如果`dispatcher.isDispatchNeeded()`返回false，我们将直接在协程体中调用resumeWith()方法。

`dispatcher.dispatch(context, this)`实际上等同于将代码的执行过程分发给默认的线程池。第二个参数是Runnable，我们在这里传递的是this，因为DispatchedContinuation间接地实现了Runnable接口。

Dispatchers.Defaul是DefaultScheduler，DefaultScheduler是一个单例类，所以只有一个对象会被创建并在任何地方使用。DefaultScheduler是SchedulerCoroutineDispatcher的一个子类。

~~~Kotlin
@JvmStatic
public actual val Default: CoroutineDispatcher = DefaultScheduler

internal object DefaultScheduler : SchedulerCoroutineDispatcher(
    CORE_POOL_SIZE, MAX_POOL_SIZE,
    IDLE_WORKER_KEEP_ALIVE_NS, DEFAULT_SCHEDULER_NAME
) { 
    ... 
}

internal open class SchedulerCoroutineDispatcher(
    private val corePoolSize: Int = CORE_POOL_SIZE,
    private val maxPoolSize: Int = MAX_POOL_SIZE,
    private val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    private val schedulerName: String = "CoroutineScheduler",
) : ExecutorCoroutineDispatcher() {
    
    ...
    
    override fun dispatch(context: CoroutineContext, block: Runnable): Unit = coroutineScheduler.dispatch(block)
    
    ...
}
~~~

`Dispatchers.Default#dispatch()`调用SchedulerCoroutineDispatcher的dispatch()方法，它将会调用`coroutineScheduler.dispatch()`。

**CoroutineScheduler**

CoroutineScheduler是Kotlin中实现的线程池，它提供了coroutine可以运行的线程，这意味着它会生成它们（译注：应该是线程池可以生成线程）

CoroutineScheduler是Executor的一个子类，它的`execute()`方法也是转发给`dispatch()`方法。

~~~Kotlin
internal class CoroutineScheduler(
    @JvmField val corePoolSize: Int,
    @JvmField val maxPoolSize: Int,
    @JvmField val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    @JvmField val schedulerName: String = DEFAULT_SCHEDULER_NAME
) : Executor, Closeable {
  
  ...
  
  override fun execute(command: Runnable) = dispatch(command)
  
  ...
  
  fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {
        trackTask() // this is needed for virtual time support
        val task = createTask(block, taskContext)
        // try to submit the task to the local queue and act depending on the result
        val currentWorker = currentWorker()
        val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)
        if (notAdded != null) {
            if (!addToGlobalQueue(notAdded)) {
                // Global queue is closed in the last step of close/shutdown -- no more tasks should be accepted
                throw RejectedExecutionException("$schedulerName was terminated")
            }
        }
        val skipUnpark = tailDispatch && currentWorker != null
        // Checking 'task' instead of 'notAdded' is completely okay
        if (task.mode == TASK_NON_BLOCKING) {
            if (skipUnpark) return
            signalCpuWork()
        } else {
            // Increment blocking tasks anyway
            signalBlockingWork(skipUnpark = skipUnpark)
        }
    }
  
  internal inner class Worker private constructor() : Thread() { ... }
}
~~~

在dispatch方法中，我们看到以下内容：

- createTask() - 我们通过传递的Runnable创建Task，该task实际上是一个DispatchedContinuation
- currentWorker() - 获得当前正在执行的线程。worker是CoroutineScheduler的一个内部类
- currentWorker.submitToLocalQueue() - 添加task到工作线程的本地队列并等待执行

**Worker**

Worker是Kotlin coroutine的线程。Worker继承自Thread，其实际上是Java线程的一个封装。我们得出结论，Worker就是Thread。

让我们分析一下Worker是怎么执行task的。

~~~Kotlin
internal inner class Worker private constructor() : Thread() {
    ...
    override fun run() = runWorker()

    private fun runWorker() {
        ...
        while (!isTerminated && state != WorkerState.TERMINATED) {
            val task = findTask(mayHaveLocalTasks)
            if (task != null) {
                ...
                executeTask(task)
                continue
            } 
            ...
        }
        ...
    }
    ...
}
~~~

Worker会重写Thread的run方法，然后会调用runWorker方法。在一个while循环中，将会经常试图在本地工作队列中拉取task。如果有一个task需要被执行，`executeTask(task)`会被调用来执行它。

我们看一下executeTask（task）方法做了什么:
~~~Kotlin
internal inner class Worker private constructor() : Thread() {
    ...
    private fun executeTask(task: Task) {
        val taskMode = task.mode
        idleReset(taskMode)
        beforeTask(taskMode)
        runSafely(task)
        afterTask(taskMode)
    }
    
    ...
    
    fun runSafely(task: Task) {
        try {
            task.run()
        } 
        ...
    }
}

internal abstract class Task(
    @JvmField var submissionTime: Long,
    @JvmField var taskContext: TaskContext
) : Runnable {
    constructor() : this(0, NonBlockingContext)
    inline val mode: Int get() = taskContext.taskMode // TASK_XXX
}
~~~

在`runSafely()`方法中，我们调用`task.run()`。Task是一个Runnable且`Runnable#run()`实际上意味着我们的协程任务实际上正在执行。

DispatchedContinuation继承自DispatchedTask类，DispatchedTask继承自SchedulerTask，SchedulerTask实现了Runnable接口。我们看到DispatchedTask最终实现了Runable接口，所以我们DispatchedTask中`run()`的实现。

~~~Kotlin
internal abstract class DispatchedTask<in T>(
    @JvmField public var resumeMode: Int
) : SchedulerTask() {

   public final override fun run() {
     // ...
     try {
         val delegate = delegate as DispatchedContinuation<T>
         val continuation = delegate.continuation
         withContinuationContext(continuation, delegate.countOrElement) {
             // ...
             val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
             if (job != null && !job.isActive) {
                 val cause = job.getCancellationException()
                 cancelCompletedResult(state, cause)
                 continuation.resumeWithStackTrace(cause)
             } else {
                 if (exception != null) {
                     continuation.resumeWithException(exception)
                 } else {
                     continuation.resume(getSuccessfulResult(state))
                 }
             }
         }
     }
     // ...
   }
}
~~~

在run()方法中，原始的协程延续对象是通过DispatchedContinuation获取的，resumeWithStackTrace，resumeWithException和resume是扩展函数，它们会触发resumeWith()方法。

~~~Kotlin
/**
 * Resumes the execution of the corresponding coroutine passing [value] as the return value of the last suspension point.
 */
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

/**
 * Resumes the execution of the corresponding coroutine so that the [exception] is re-thrown right after the
 * last suspension point.
 */
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable): Unit =
    resumeWith(Result.failure(exception))

@Suppress("NOTHING_TO_INLINE")
internal inline fun Continuation<*>.resumeWithStackTrace(exception: Throwable) {
    resumeWith(Result.failure(recoverStackTrace(exception, this)))
}
~~~

### 3.1.5 resumeWith

![](https://miro.medium.com/max/1400/1*t1cvbpjMSAalpOp8t1-IXw.jpeg)

最终的resumeWith()实现在BaseContinuationImpl类中：

~~~Kotlin
public final override fun resumeWith(result: Result<Any?>) {
    // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
    var current = this
    var param = result
    while (true) {
        // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
        // can precisely track what part of suspended callstack was already resumed
        probeCoroutineResumed(current)
        with(current) {
            val completion = completion!! // fail fast when trying to resume continuation without completion
            val outcome: Result<Any?> =
                try {
                    val outcome = invokeSuspend(param)
                    if (outcome === COROUTINE_SUSPENDED) return
                    Result.success(outcome)
                } catch (exception: Throwable) {
                    Result.failure(exception)
                }
            releaseIntercepted() // this state machine instance is terminating
            if (completion is BaseContinuationImpl) {
                // unrolling recursion via loop
                current = completion
                param = outcome
            } else {
                // top-level completion reached -- invoke and return
                completion.resumeWith(outcome)
                return
            }
        }
    }
}

protected abstract fun invokeSuspend(result: Result<Any?>): Any?
~~~

让我们注意`invokeSuspend()`方法

### 3.1.6 invokeSuspend

![](https://miro.medium.com/max/1400/1*HLjeocfvjdvBkTLTq7vETQ.jpeg)

协程体将会根据状态机顺序执行直到suspend函数被调用。当我们调用它时，方法将会返回COROUTINE_SUSPEND标志，并且直接返回退出循环以及Continuation体，在这种情况下，不会发生线程阻塞。

当一个方法需要被suspend时，状态机会存储上一次的结果作为延续体的一个变量。当suspend函数恢复，Continuation的resumeWith()方法会被调用，然后invokeSuspend()方法会被调用。这样，协程体后面的代码也可以继续执行。

![](https://miro.medium.com/max/1400/1*plj4er7LEFdVnzRj-2_kXQ.jpeg)

### 3.1.7 协程构建的总结

1、协程体(挂起lambda方法)在编译期会被编译成内部类，它继承SuspendLambda并实现Function2，特定的继承链是：
`SuspendLambda -> ContinuationImpl -> BaseContinuationImpl -> Continuation`

2、`CoroutineScope#launch()`创建一个协程，根据默认的启动模式CoroutineStart.DEFAULT，创建一个协程对象StandaloneCoroutine并启动`StandaloneCoroutine#start(start, coroutine, block)`

3、StandaloneCoroutine是AbstractCoroutine的一个子类，且StandaloneCoroutine#start()的实现可以在AbstractCoroutine (AbstractCoroutine#start())中找到。AbstractCoroutine#start() 会触发CoroutineStart#invoke()

4、由于我们例子中的调度器是Dispatchers.Default，我们调用协程体的startCoroutineCancellable()方法来执行逻辑CoroutineStart#invoke()

5、startCoroutineCancellable是一个链式调用：
`createCoroutineUnintercepted().intercepted().resumeCancellableWith()`

6、`createCoroutineUnintercepted`创建一个协程体对象

7、intercepted（）使用一个interceptor/scheduler来将协程体对象包装为DispatchedContinuation。DispatchedContinuation代表协程体类的一个延续对象，并包含调度器。

8、由于调度器是Dispatchers.Default且isDispatchNeeded() 方法返回true，DispatchedContinuation#resumeCancellableWith() 使用线程调度器的dispatcher#dispatch(context, this)来调度。

9、`Dispatchers.Default#dispatch()`调用SchedulerCoroutineDispatcher的dispatch()方法，它会调用CoroutineScheduler#dispatch()

10、CoroutineScheduler申请一个线程Worker并在Worker#run()方法中触发DispatchedContinuation的run方法(`DispatchedTask#run()`)

11、run方法会触发resumeWith()方法。协程体的调用实际上是一个resumeWith()方法的调用。

![](https://miro.medium.com/max/1400/1*j6NPAyhojgjP6u6FFkcuhg.jpeg)

## 3.2 字节码分析
让我们分析一下我们例子的字节码。使用Tools->Kotlin->Show Kotlin Bytecode，然后点击Decompile生成对应的反编译的Java代码。

~~~Java
BuildersKt.launch$default((CoroutineScope)LifecycleOwnerKt.getLifecycleScope(this), (CoroutineContext)null, (CoroutineStart)null, (Function2)(new Function2((Continuation)null) {
   int label;

   @Nullable
   public final Object invokeSuspend(@NotNull Object $result) {
      Object var10000;
      label17: {
         Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
         MainActivity var6;
         switch(this.label) {
         case 0:
            ResultKt.throwOnFailure($result);
            var6 = MainActivity.this;
            this.label = 1;
            var10000 = var6.getRandomNum(this);
            if (var10000 == var5) {
               return var5;
            }
            break;
         case 1:
            ResultKt.throwOnFailure($result);
            var10000 = $result;
            break;
         case 2:
            ResultKt.throwOnFailure($result);
            var10000 = $result;
            break label17;
         default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
         }

         int randomNum = ((Number)var10000).intValue();
         var6 = MainActivity.this;
         double var10001 = (double)randomNum;
         this.label = 2;
         var10000 = var6.getSqrt(var10001, this);
         if (var10000 == var5) {
            return var5;
         }
      }

      double sqrt = ((Number)var10000).doubleValue();
      MainActivity.this.log(String.valueOf(sqrt));
      return Unit.INSTANCE;
   }

   @NotNull
   public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
      Intrinsics.checkNotNullParameter(completion, "completion");
      Function2 var3 = new <anonymous constructor>(completion);
      return var3;
   }

   public final Object invoke(Object var1, Object var2) {
      return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
   }
}), 3, (Object)null);
~~~

我们看一下invokeSuspend方法：

1、15,16行，我们检查var10000是否为COROUTINE_SUSPENDED，如果是的话，意味着我们没有可用的返回值，并等待可用的返回值返回。在我们的例子中，getRandomNum()将会返回COROUTINE_SUSPENDED。在这之前我们把label设为1。

为什么getRandomNum()方法返回COROUTINE_SUSPENDED？

~~~Java
private final Object getRandomNum(Continuation var1) {
  Object $continuation;
  label20: {
     // ...
  }

  Object $result = ((<undefinedtype>)$continuation).result;
  Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
  switch(((<undefinedtype>)$continuation).label) {
  case 0:
     ResultKt.throwOnFailure($result);
     ((<undefinedtype>)$continuation).label = 1;
     if (DelayKt.delay(1000L, (Continuation)$continuation) == var5) {
        return var5;
     }
     break;
  case 1:
     ResultKt.throwOnFailure($result);
     break;
  default:
     throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
  }

  byte var2 = 1;
  return CollectionsKt.first(CollectionsKt.shuffled((Iterable)(new IntRange(var2, 1000))));
}
~~~

在13行，delay()方法被调用，我们看一下它的实现。

~~~Kotlin
public suspend fun delay(timeMillis: Long) {
    if (timeMillis <= 0) return // don't delay
    return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->
        // if timeMillis == Long.MAX_VALUE then just wait forever like awaitCancellation, don't schedule.
        if (timeMillis < Long.MAX_VALUE) {
            cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)
        }
    }
}

override fun scheduleResumeAfterDelay(timeMillis: Long, continuation: CancellableContinuation<Unit>) {
    postDelayed(Runnable {
        with(continuation) { resumeUndispatched(Unit) }
    }, timeMillis)
}
~~~

suspendCancellableCoroutine的返回是COROUTINE_SUSPENDED。需要suspend并等待结果返回。可以看到这里的delay的逻辑与Handler机制相似(Handler.postDelayed)。当执行完毕后，`continuation.resume() -> BaseContinuationImpl.resumeWith()-> SuspendLambda.invokeSuspend()`将被调用来恢复。

当getRandomNum()方法执行结束完毕且可用的返回值被返回后，invokeSuspend()会被调用。此时label=1。首先我们调用`throwOnFailure`，如果结果失败的话会抛出异常。如果结果成功，我们赋值给var10000，然后break当前的执行逻辑(32-39行)。

2、15,16行（译注：这里应该是36-37行），getSqrt()的执行过程与getRandomNum()类似。此时label=2。当getRandomNum()（译注：应该是sqrt()）的结果返回，invokeSuspend将会被再次调用，同样调用throwOnFailure，如果结果失败会抛出异常。如果结果成功，调用break label17，跳转到101行执行后续的逻辑。

![](https://miro.medium.com/max/1400/1*QJf2_Xbb6fIycaz8C00ZMA.jpeg)

现在你可以探索其他的协程构建器，并看它们之间有什么具体的区别。如果有你有什么疑问的话，你可以通过[LinkedIn](https://www.linkedin.com/in/aleksandarzekovic/)写给我。

希望你喜欢这篇文章，请继续关注更多！