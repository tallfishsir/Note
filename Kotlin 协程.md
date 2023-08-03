# Kotlin 协程

在 Android 中，协程是一个线程管理框架，用于简化异步编程。

- 创建或者获取 CoroutineScope 对象
- 创建 CoroutineScope 对象，需要配置 CoroutineContext 
- 启动 CoroutineScope 创建协程，并在内部使用 suspend 函数处理异步操作

AndroidStudio 完成以下设置，执行「Thread.currentThread().name」就会打印协程名称

```
Debug Configuration → VM options
-Dkotlinx.coroutines.debug=on
```

## CoroutineScope

Android 协程中，CoroutineScope 是一个用于定义协程作用域的接口，内部包含一个 CoroutineContext 对象，用于配置协程的执行环境。CoroutineScope 具有生命周期的概念，当被取消时，会释放资源，有助于避免内存泄漏等问题。

### 常见 CoroutineScope

#### GlobalScope

GlobalScope 是一个全局协程作用域，具有全局生命周期，可以用于创建顶级协程。在 Android 中，如果需要创建一个与 Application 生命周期相同的协程作用域，可以考虑使用。

需要注意避免在具有有限生命周期的组件 (Activity等) 中使用 GlobalScope，会导致发生内存泄漏。

```kotlin
public object GlobalScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```

#### ViewModelScope

viewModelScope 是一个与 Android ViewModel 对象生命周期绑定的协程作用域，在这个作用域中启动的协程将在 ViewModel 被清除时自动取消。

```kotlin
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immedi
        )
    }
```

#### LifecycleScope

LifecycleScope 是一个与 Android Lifecycle 对象生命周期绑定的协程作用域，在这个作用域中启动的协程将在 关联的生命周期组件销毁时自动取消。

#### supervisorScope 

supervisorScope 是一个顶层函数，适用于需要并发执行多个子任务，其中一个子任务失败不应影响其他子任务的场景。

supervisorScope具有一个特殊的 SupervisorJob。常规的 Job，一个子协程抛出的未捕获异常会导致父协程以及其它子协程被取消。而在 SupervisorJob 中，一个子协程抛出的未捕获异常仅会导致该子协程被取消，不会影响父协程和其它子协程。

使用 supervisorScope 需要注意自主处理子协程的异常，通过 join 方法等待子协程完成，然后检查 Job 的状态，并采取相应的措施。

```kotlin
supervisorScope {
	val childJob1 = launch {
		// ... do something
		throw RuntimeException("Error in childJob1")
	}

	val childJob2 = launch {
		// ... do something
	}

	// 等待子协程完成
	childJob1.join()
	childJob2.join()
}
```

#### CoroutineScope

除了上面直接获取 CoroutineScope 对象，还可以使用 CoroutineScope 构造函数创建自定义 CoroutineScope。创建 CoroutineScope 实例需要传入 CoroutineContext 对象配置协程环境，CoroutineContext 通常包括一个 Job 和一个调度器（Dispatcher）

```kotlin
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())
   
internal class ContextScope(context: CoroutineContext) : CoroutineScope {
    override val coroutineContext: CoroutineContext = context
    override fun toString(): String = "CoroutineScope(coroutineContext=$coroutineContext)"
}


val myJob = Job()
val myDispatcher = Dispatchers.Default
val myScope = CoroutineScope(myJob + myDispatcher)
```

一般以下几种情况，才会有创建自定义 CoroutineScope 的需求

- 在一个特定的功能模块或组件内共享多个协程，且这些协程共享相同的上下文（如 Job 和调度器）
- 在一个没有预定义协程作用域内要使用协程功能

需要注意的是，如果在具有生命周期的组件中使用自定义 CoroutineScope，在组件生命周期结束时，调用 Job.cancel() 取消其关联的 Job。

### CoroutineScope 启动方式

Kotlin 提供了几种不同的协程启动方式：runblocking、launch、async

#### runblocking

runblocking 启动协程会阻塞当前线程，直到协程内所有的内容和子协程执行完毕才会结束，阻塞具有两方面的的含义：

- 阻塞线程，线程会进入等待状态，等待 runblocking 执行完成
- runblocking 会等待内部启动的协程执行完，再结束

```
runBlocking {
    
}

public actual fun <T> runBlocking(context: CoroutineContext, block: suspend CoroutineScope.() -> T): T {
    ...
    val currentThread = Thread.currentThread()
    val contextInterceptor = context[ContinuationInterceptor]
    val eventLoop: EventLoop?
    val newContext: CoroutineContext
    ...
    val coroutine = BlockingCoroutine<T>(newContext, currentThread, eventLoop)
    coroutine.start(CoroutineStart.DEFAULT, coroutine, block)
    return coroutine.joinBlocking()
}
```

runblocking 的返回值是带接收者的函数类型参数 block 的返回值，一般用于测试和调试协程。

#### launch

launch 是一个扩展函数，用于启动一个不需要返回结果的协程，它会返回一个 Job 对象，Job 对象用于管理协程的生命周期。launch 在协程内是异步并发的，不会阻塞当前作用域。

```kotlin
val scope = CoroutineScope(Dispatchers.Default)
val job = scope.launch {
    // 协程代码
}

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
```

第一个参数 CoroutineContext 是协程的执行环境配置，默认值为 EmptyCoroutineContext

第二个参数 CoroutineStart 是协程的启动模式，有两种模式：

- CoroutineStart.DEFAULT：立即执行协程
- CoroutineStart.LAZY：调用 CoroutineScope.start/join/await() 后执行协程

第三个参数带接收者的函数类型参数就是协程内部执行的内容

#### async

async 是一个扩展函数，用于启动一个需要返回结果的协程，它会返回一个 Deferred 对象。Deferred 是 Job 子类，表示一个延迟计算的结果，可以使用 Deferred.await() 获取协程的返回值。async 在协程内是异步并发的，不会阻塞当前作用域。

```kotlin
val scope = CoroutineScope(Dispatchers.Default)
val deferredResult = scope.async {
    // 协程代码，返回结果
}
val result = deferredResult.await()

public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

需要注意的是，async 开启后就会立刻执行其中代码，不会阻塞。但调用 await 时，如果 async 的结果还没有返回，会阻塞，等待运行结束。

#### withContext

withContext 在指定的 CoroutineContext 中执行代码块并返回结果。launch 在协程内是同步的，会阻塞当前作用域。

```
withContext(context: CoroutineContext, block: suspend CoroutineScope.() -> T): T
```

#### withTime

withTime 在指定超时时间内执行代码块，如果超时就会抛出异常 TimeoutCancellationException。在协程内是同步的，会阻塞当前作用域。

```kotlin
withTimeout(timeoutMillis: Long, block: suspend CoroutineScope.() -> T): T
```

#### withTimeOrNull

withTimeOrNull 在指定超时时间内执行代码块，如果超时则返回 null。在协程内是同步的，会阻塞当前作用域。

```kotlin
withTimeoutOrNull(timeoutMillis: Long, block: suspend CoroutineScope.() -> T): T?
```

## CoroutineContext

CoroutineContext 是一个用于定义协程运行环境的接口，主要作用是为协程提供执行环境配置和生命周期管理，主要由以下四个 Element 组成：

- Job：协程的唯一标识，用来控制协程生命周期
- CoroutineDispatcher：指定协程运行在哪个线程或者线程池中
- CoroutineName：用于设置协程的名称
- CoroutineExceptionHandler：指定协程的异常处理器，用来处理协程内未捕获的异常



![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt4xh9LlagnhjmwzvYj1dGAz5NgzgfzPjnHdlJKLSS1w482IRdguMK9LyDdG4QJgT5mMicAVTpwiag1w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

以上就是 CoroutineContext 相关类的关系图，在创建 CoroutineScope 的时候可以使用 + 运算符将多个 Element 合并为一个 CoroutineContext 对象。

```kotlin
val scope = CoroutineScope(CoroutineName("Coroutine-Name") + Dispatchers.IO)
val job = scope.launch(start = CoroutineStart.DEFAULT){
    // 协程代码
}
```

CoroutineContext 内部设计允许组合，修改，扩展协程的能力，且不会发生同一种配置重复冲突的现象：

- get(key: Key)：根据给定的 Key，从 CoroutineContext 中获取对应的 Element，如果不存在返回 null
- plus(context: CoroutineContext)：将两个 CoroutineContext  合并为一个新的 CoroutineContext 
- CoroutineContext.Element：让每个 Element 的子类类型作为 Key，相应的对象实例作为 Value，保证不重复

```
public interface CoroutineContext {
    public operator fun <E : Element> get(key: Key<E>): E?
    public operator fun plus(context: CoroutineContext): CoroutineContext

    public interface Key<E : Element>

    public interface Element : CoroutineContext {
     
        public val key: Key<*>

        public override operator fun <E : Element> get(key: Key<E>): E? =
            @Suppress("UNCHECKED_CAST")
            if (this.key == key) this as E else null

        public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
            operation(initial, this)

        public override fun minusKey(key: Key<*>): CoroutineContext =
            if (this.key == key) EmptyCoroutineContext else this
    }
}
```

### Job

CoroutineScope 启动协程后会返回 Job 对象，它具有以下功能：

- 控制协程的生命周期
- 管理父子协程关系
- 跟踪协程的执行状态

#### 控制协程的生命周期

Job 提供了一种简单的方式来管理协程的生命周期，可以调用以下方法：

- start()：如果协程启动模式是 LAZY，需要调用此方法启动协程
- cancel()：如果需要停止协程，需要调用此方法，协程会尽可能的取消
- join()：挂起父协程，等待 Job 管理的当前协程执行完成

#### 管理父子协程关系

Job 支持创建父子协程关系，这意味着当一个父协程被取消时，它的所有子协程也会被取消。这有助于管理协程的层次结构和生命周期。

这种功能是借助于 CancellationException，当调用 cancel() 后，协程并不会立即结束，需要在协程内部通过 suspend 函数或者 isActive 检查。如果此时协程内容正在调用另一个 suspend 函数，在 suspend 函数恢复时，会检查协程的取消状态，然后抛出 CancellationException 异常。常见的用于检查取消状态的 suspend 函数有：

- yield()：
- delay()
- withContext()

#### 跟踪协程的执行状态

Job 可以获取对应属性，跟踪协程的执行状态：

- isActive：判断 Job 是否处于活跃状态

  执行 start() 后 Job 会进入此状态，父 job 没有被取消或异常和等待子 job 完成的时间内都被看成活跃状态

- isCancelled：判断 Job 是否处于已经取消状态

  执行 cancel() 后 Job 会进入此状态，父 Job 或者子 Job 取消也会进入此状态

- isCompleted：判断 Job 是否处于已完成状态

  Job 在任何原因下结束都会进入此状态，包括被取消、发生错误和执行完成。父 job 也会等到所有的子 job 都是完成状态时，才会变成完成状态

### Dispatchers

Dispatchers 是一个用于指定协程运行在哪个线程或线程池上的调度器，常用的 Dispatchers 有以下几种：

- Dispatchers.Main：用于在主线程上更新 UI
- Dispatchers.Default：用于CPU密集型任务的线程池，线程个数一般与机器CPU核心数量保持一致
- Dispatchers.IO：用于IO密集型任务的线程池，内部线程数量较多，一般为64个

除了内置的 Dispatcher，还可以自定义 Dispatcher，来指定线程或者线程池。

```kotlin
val job = Job()
val mySingleDispatcher = Executors.newSingleThreadExecutor {
    Thread(it,"MySingleThread").apply { isDaemon = true }
}.asCoroutineDispatcher()
val scope = CoroutineScope(job + mySingleDispatcher)
```

### CoroutineName

CoroutineName 用于给协程命名

```kotlin
val job = Job()
val name = CoroutineName("tallfish")
val dispatcher = Dispatchers.IO
val scope = CoroutineScope(job + name + dispatcher)
scope.launch {
    println("${Thread.currentThread().name}")
}
```

### CoroutineExceptionHandler

CoroutineExceptionHandler 用于处理协程内部为捕获的异常，协程的异常分为两类：取消异常(CancellationException) 和其他异常，这两种异常的处理方式并不一样。

取消异常用于父子协程功能，最好不要捕获异常，否则会破坏结构化。

其他异常有以下几种常见处理方式：

- CoroutineExceptionHandler

  创建  CoroutineExceptionHandler 接口的对象，并且必须放入顶层协程的 CoroutineContext 中并说明 launch 启动。

  ```kotlin
  val job = Job()
  val exceptionHandler  = CoroutineExceptionHandler { coroutineContext, throwable ->
  }
  val scope = CoroutineScope(job + exceptionHandler)
  scope.launch{}
  ```

- try-catch

  单个协程中的异常，使用 try-catch 块捕获异常，需要注意的是，try-catch 块只能捕获当前协程中的异常。

  ```
  kotlinCopy codeGlobalScope.launch {
      try {
          // 协程代码
      } catch (e: Exception) {
          // 处理异常
      }
  }
  ```

- supervisorScope 

  supervisorScope 允许子协程中的异常独立处理，而不会导致父协程和其他子协程的取消。

  ```kotlin
  kotlinCopy codeGlobalScope.launch {
      supervisorScope {
          launch {
              // 子协程 1
          }
  
          launch {
              // 子协程 2
          }
      }
  }
  ```

- SupervisorJob

  SupervisorJob 是一个特殊的 Job，用于创建具有“监督”特性的协程作用域。与普通的协程作用域不同，使用 SupervisorJob 创建的协程作用域允许子协程中的异常独立处理，而不会导致父协程和其他子协程的取消。

  ```kotlin
  val supervisorJob = SupervisorJob()
  val handler = CoroutineExceptionHandler{coroutineContext, throwable ->
  }
  launch {
      launch(supervisorJob+handler) {
          // 子协程 1
      }
      launch {
     	 // 子协程 2
  	}
  }
  ```

## suspend 函数

suspend 函数可以让协程在线程的执行过程中挂起和恢复，这里的挂起是非阻塞式的，不会阻塞当前执行的线程。主要作用是简化异步操作和非阻塞代码的编写，能够以类似于同步代码的方式编写异步操作。

从本质上讲，挂起是将协程与它当前所在的线程脱钩，然后换到 suspend 函数指定的线程上执行，而恢复则是让协程中 suspend 函数后面的代码重新回到原来的线程上执行。

因此， suspend 函数只能在协程或者其他挂起函数中运行，换句话说就是挂起函数必须直接或者间接地在协程中执行。suspend 关键字起到了两个作用：

- 语言层面：标志函数是一个耗时操作，需要放在协程中执行
- 编译层面：将 suspend 函数转为 Continuation 返回值的函数

在编译阶段，suspend 函数会被编译生成一个带有 Continuation 对象的函数。Continuation 接口包含两个部分：

- context：协程的执行配置环境，保存着名称，DIspatcher 等信息，用于管理协程的生命周期和行为
- resumeWith：用于挂起操作完成后恢复协程的执行，参数 Result 是一个密封类，表示成功（Success）或失败（Failure）

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

从 suspend 函数转换为 Continuation 函数的过程叫做 CPS 转换 (Continuation-Passing-style Transform)

```
suspend fun testCoroutine(){
    val user = getUserInfo()
    val friendList = getFriendList(user)
    val feedList = getFeedList(user,friendList)
    println(feedList)
}

suspend fun getUserInfo(): String{
    withContext(Dispatchers.IO){
        delay(1000)
    }
    return "Coder"
}

suspend fun getFriendList(user: String): String{
    withContext(Dispatchers.IO){
        delay(1000)
    }
    return "Tom,Jack"
}
```

上面的代码，经过编译后变成下面这样，Continuation 的作用如下：

- 保存协程的状态：当一个协程挂起时，Continuation 会保存协程的当前状态，包括局部变量、执行位置等。这样，协程可以在恢复时从中断点继续执行。
- 管理协程的恢复：Continuation 提供了 resumeWith 方法，用于在挂起操作完成时恢复协程的执行。当异步操作完成时，可以调用 resumeWith 来传递结果并继续协程的执行。
- 支持协程的上下文：Continuation 提供了对协程上下文的访问，允许在 suspend 函数中访问和修改协程的行为和状态。

```
fun testCoroutine(completion: Continuation<Any?>): Any? {
	//3个变量，对应原函数的3个变量
	lateinit var user: String
	lateinit var friendList: String
	lateinit var feedList: String
	
	//result接收挂起函数的运行结果
	var result = continuation.result
	
	//suspendReturn表示挂起函数的返回值
	var suspendReturn: Any? = null

	//该flag表示当前函数被挂起了
	val sFlag = CoroutineSingles.CORTOUINE_SUSPEND

	val continuation = if (completion is TestContinuation){
		completion
	}else{
		//作为参数
		TestContinuation(completion)
	}
    
    when(continuation.label){
		0 -> {
			//检测异常
			throwOnFailure(result)
			//将label设置为1，准备进入下一个状态
			continuation.label = 1
			//执行getUserInfo
			suspendReturn = getUserInfo(continuation)
			//判断是否挂起
			if (suspendReturn == sFlag){
				return suspendReturn
			}else{
				result = suspendReturn
			}
		}
	
		1 -> {        
			throwOnFailure(result)
			// 获取 user 值        
			user = result as String        
			// 将协程结果存到 continuation 里        
			continuation.mUser = user        
			// 准备进入下一个状态        
			continuation.label = 2        
			// 执行 getFriendList        
			suspendReturn = getFriendList(user, continuation)        
			// 判断是否挂起       
			if (suspendReturn == sFlag) {            
				return suspendReturn
			}  else {            
				result = suspendReturn
			}           
		} 
		
		2 -> {
			throwOnFailure(result)
			user = continuation.mUser as String
			//获取friendList的值
			friendList = result as String
			//将挂起函数结果保存到continuation中
			continuation.mUser = user
			continuation.mFriendList = friendList
			//准备进入下一个阶段
			continuation.label = 3
			//执行获取feedList
			suspendReturn = getFeedList(user,friendList,continuation) 
			//判断是否挂起
			if (suspendReturn == sFlag){
				return suspendReturn
			}else{
				result = suspendReturn
			}
		}
		
		3 -> {
			throwOnFailure(result)
			user = continuation.mUser as String        
			friendList = continuation.mFriendList as String        
			feedList = continuation.result as String        
			loop = false
		}
	}

	class TestContinuation(completion: Continuation<Any?>) : ContinuationImpl(completion){
		//表示状态机的状态
		var label: Int = 0
		//当前挂起函数执行的结果
		var result: Any? = null
		//用于保存挂起计算的结果，中间值
		var mUser: Any? = null
		var mFriendList: Any? = null
		
		/**
		* 状态机入口，该类是ContinuationImpl中的抽象方法，同时ContinuationImpl
		* 又是继承至[Continuation]，所以[Continuation]中的resumeWith方法会回调
		* 该invokeSuspend方法。
		* 
		* 即每当调用挂起函数返回时，该方法都会被调用，在方法内部，先通过result记录
		* 挂起函数的执行结果，再切换labeal，最后再调用testCoroutine方法
		* */
		override fun invokeSuspend(_result: Result<Any?>): Any?{
			result = _result
			label = label or Int.Companion.MIN_VALUE
			return testCoroutine(this)
		}
	}
}
```

Continuation 的作用如下：

保存协程的状态：当一个协程挂起时，Continuation 会保存协程的当前状态，包括局部变量、执行位置等。这样，协程可以在恢复时从中断点继续执行。
管理协程的恢复：Continuation 提供了 resumeWith 方法，用于在挂起操作完成时恢复协程的执行。当异步操作完成时，可以调用 resumeWith 来传递结果并继续协程的执行。
支持协程的上下文：Continuation 提供了对协程上下文的访问，允许在 suspend 函数中访问和修改协程的行为和状态。

## 协程间通信

### Channel

在 Android 协程中，Channel 是一种用于在协程之间传输数据的通信原语。Channel 允许一个或多个生产者协程将数据发送到 Channel，一个或多个消费者协程从 Channel 接收数据。Channel 是热数据流，不管有没有消费者订阅，都会发射数据。

```kotlin
val channel = Channel<Int>()
launch {
    channel.send(1)
    channel.send(2)
    channel.send(3)
    
    channel.close()
}
launch {
    for (i in channel){
        println("Receive: $i")
    }
}
```

#### 创建

 Channel 提供了多种创建方法，以下是其中常用的几种：

Channel() 像是构造函数，其实它却是一个顶层函数，注意：如果把顶层函数当作构造函数来用的时候，函数首字母需要大写。

```
fun <E> Channel(
    capacity: Int = RENDEZVOUS,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND,
    onUndeliveredElement: ((E) -> Unit)? = null
): Channel<E>
```

- capacity：Channel 的容量
  - RENDEZVOUS：容量是0
  - CONFLATED：容量是1，而且新的数据会替代旧的数据
  - BUFFRED：容量默认是 64
  - UNLIMITED：无限容量
- onBufferOverflow：Channel 容量满了以后的策略
  - SUSPEND：挂起当前的 send() 方法，等管道容量有空闲位置以后再恢复
  - DROP_OLDEST：丢弃掉最旧的那个数据
  - DROP_LATEST：丢掉最新的数据
- onUndeliveredElement：异常处理回调，当管道种某些数据没有被成功接收的时候，这个回调就会被调用。这里其实也需要注意，是数据已经发送了，但是没有被接收，才会触发回调

ConflatedChannel() 创建一个 conflated 通道。在这种类型的通道中，如果发送者产生数据的速度快于接收者处理数据的速度，那么缓冲区中的数据将被新数据替换。

```kotlin
val conflatedChannel = ConflatedChannel<Int>()
```

BufferedChannel() 创建一个具有指定缓冲区容量的通道。当缓冲区满时，发送者将被挂起，直到有空间可用。

```kotlin
val bufferedChannel = Channel<Int>(capacity = 5)
```

#### 发送数据

Channel 常用的发送数据的方法有 send() 和 offer() 两种。

send(element: E)：将元素发送到 Channel，如果没有可用缓冲区空间，该方法会挂起协程，直到有空间可用。

offer(element: E)：尝试将元素发送到 Channel，如果没有可用缓冲区空间，则立即返回 false，不会挂起协程。

需要注意的是：

- 使用 send 方法时，确保接收端有协程处理数据，否则发送端可能会永久挂起
- 使用 offer 方法时，需要处理发送失败的情况，例如重试或放弃发送
- 使用 Channel 时，确保在不再需要时调用 channel.close() 关闭通道以避免内存泄漏

#### 接收数据

Channel 常用的接收数据的方法有以下几种。

receive()：从通道中接收一个元素。如果通道为空，调用该方法的协程将挂起，直到通道中有元素可用。如果 Channel 被关闭且没有更多数据可用，它将抛出 ClosedReceiveChannelException。

```kotlin
try {
    val value = channel.receive()
    println("Received: $value")
} catch (e: ClosedReceiveChannelException) {
    println("Channel is closed")
}
```

receiveOrNull()：从通道中接收一个元素，如果通道为空，调用该方法的协程将挂起，直到通道中有元素可用。如果 Channel 被关闭且没有更多数据可用，它会返回 null。

```kotlin
val value = channel.receiveOrNull()
if (value == null) {
    println("Channel is closed")
} else {
    println("Received: $value")
}
```

除了上面两种方法，还可以是用 for 循环遍历 Channel，当 Channel 关闭且没有数据时，循环会自动结束。

```kotlin
for (value in channel) {
    println("Received: $value")
}
println("Channel is closed")
```

### Flow

在 Android 协程中，Flow 是基于协程的响应式流，可以将数据从生产者协程发射到消费者协程。Flow 是冷数据流，只在有消费者订阅时开始发射数据，并且支持背压，会自动处理生产者和消费者之间速度的差异。

```kotlin
flow {
    emit(1)
}.collect {
    println("Received value: $it")
}
```

#### 使用方式

##### 创建发送

创建 Flow 的方法最常用的方式是通过 flow{} 创建 Flow，在内部调用 emit() 向 Flow 发送数据。

```kotlin
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> 
    = SafeFlow(block)

public fun interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}

val myFlow = flow {
    for (i in 1..3) {
        delay(100) // 模拟异步操作
        emit(i) // 发射值
    }
}
```

flowOf() 创建包含固定值的 Flow并发送其中的数据，适用于已知的一组值。

```kotliin
public fun <T> flowOf(vararg elements: T): Flow<T> = flow {
    for (element in elements) {
        emit(element)
    }
}

val myFlow = flowOf(1, 2, 3)
```

asFlow()：将集合或序列转换为 Flow。适用于需要将现有集合或序列转换为 Flow 的场景。

```kotlin
public fun <T> Iterable<T>.asFlow(): Flow<T> = flow {
    forEach { value ->
        emit(value)
    }
}

val myList = listOf(1, 2, 3)
val myFlow = myList.asFlow()
```

callbackFlow { }：创建可以与回调 API 集成的 Flow。可以在其 lambda 表达式内部使用 offer() 发送数据。

```kotlin
public fun <T> callbackFlow(@BuilderInference block: suspend ProducerScope<T>.() -> Unit): Flow<T> = CallbackFlowBuilder(block)

val myFlow = callbackFlow {
    val callback = object : MyCallback {
        override fun onDataReceived(data: Int) {
            offer(data)
        }
    }
    registerCallback(callback)
    awaitClose { unregisterCallback(callback) }
}
```

zip() 将两个 Flow 中的两个元素按顺序组合生成新的 Flow，如果两个长度不一致，取少的长度作为新 Flow 长度

```
public fun <T1, T2, R> Flow<T1>.zip(other: Flow<T2>, transform: suspend (T1, T2) -> R): Flow<R> = zipImpl(this, other, transform)

val flowA = flowOf(1, 2)
val flowB = flowOf("A", "B", "C")
val zippedFlow = flowA.zip(flowB) { a, b -> a.toString() + b }
```

combine() 将两个 Flow 中的元素组合，当任一 Flow 发射新元素时，取另一个 Flow 的最新值进行组合。

```
public fun <T1, T2, R> Flow<T1>.combine(flow: Flow<T2>, transform: suspend (a: T1, b: T2) -> R): Flow<R> = flow {
    combineInternal(arrayOf(this@combine, flow), nullArrayFactory(), { emit(transform(it[0] as T1, it[1] as T2)) })
}

val flowA = flowOf(1, 2)
val flowB = flowOf("A", "B", "C")
val combinedFlow = flowA.combine(flowB) { a, b -> a.toString() + b }
```

##### 监听状态

onStart：监听 Flow 开始状态，与位置无关，在 Flow 开始传输数据之前执行操作。

onCompletion：监听 Flow 结束状态，与位置无关，在 Flow 全部数据接收后执行操作。如果发生了异常，可以使用 it 访问。

catch：捕获 Flow 发生的异常，只能捕获发生在它上面的异常。发生异常后将不再继续处理源数据的剩余数据，但可以在 catch 中 emit 新的数据。

##### 执行环境

flowOn：用于更改 Flow 的上下文，指定此方法上游的内容在某个特定的协程调度器上执行。

launchIn：配置一个 CoroutineScope 启动 Flow，可以用于在某个特定的协程调度器上执行收集操作。此方法是非阻塞式的，需要使用 Job.join() 配合使用

##### 操作符

操作符可以在 Flow 传递数据时，对数据做一些处理，将新的结果传入后续的流程中。常见的操作符有：

map：将 Flow 中的每个元素转换为另一种类型。

take：从原始 Flow 中获取指定数量的值。

filter：根据指定条件过滤 Flow 中的元素。

onEach：在 Flow 的每个元素上执行操作。

flatMapConcat：将原始 Flow 中的每个值转换为一个新的 Flow，并按顺序连接这些新 Flow 的值。

flatMapMerge：与 flatMapConcat 类似，但允许同时合并多个 Flow，而不是按顺序连接。它可以实现并发地合并多个 Flow，提高吞吐量。

##### 接收

Flow 提供了多种接收方法来手机 Flow 内的数据，当调用接收方法时，Flow 开始生产数据并传输。

collect：收集并处理 Flow 中的所有值。这是最基本的接收方法。

toList：收集 Flow 中的所有值，并将它们添加到一个 List 中。

toSet：收集 Flow 中的所有值，并将它们添加到一个 Set 中。

debounce：确保flow的各项数据之间存在一定的时间间隔

first：仅收集 Flow 中的第一个值。

single：如果 Flow 只有一个值，收集该值；否则抛出异常。

fold：提供初始值，对 Flow 中的元素执行累积操作，将操作结果存储在累积器中。

reduce：不需要提供初始值，对 Flow 中的元素执行累积操作，将操作结果存储在累积器中。

#### 原理分析

Flow 工作流程可以分为发送和接收两部分，通过 Flow{} 生成一个 Flow 对象，主要完成以下逻辑：

- 传入 FlowCollector 类型的扩展函数，这个扩展函数内部调用了 emit() 方法发送数据
- SafeFlow 的一个属性 val block(suspend FlowCollector<T>.() -> Unit) 保存了第一步传入的函数
- SafeFlow 重写 collectSafely() 方法，内部调用了 block() 方法

```kotlin
flow {
	emit(1)
}

public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
```

Flow 创建流程结束后，现在存在一个 SafeFlow 对象，对象内部包含了一个 FlowCollector 的扩展函数。

FlowCollector 是一个接口，内部只有一个 emit() 方法，所以根据 SAP ，以下两种写法是等价的；

```kotlin
public fun interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}
fun collect(collector: FlowCollector){}

//调用函数两种写法
collect {}

collect(object: FlowCollector(){
	override suspend fun emit(value: Int) {
	}
})
```

SafeFlow 是 AbstractFlow 的子类，调用 AbstractFlow.collect() 进入 Flow 接收部分。主要完成以下逻辑：

- collect 传入一个 FlowCollector 对象，并将  FlowCollector 对象封装为 SafeCollector 对象
- 调用 AbstractFlow.collectSafely() 就是调用创建 Flow 流程中生成的 SafeFlow.collectSafely()
- 由此知道 Flow{} 中调用的 emit() 方法，实际的调用对象式是 collect() 传入的 FlowCollector 封装而成的SafeCollector 对象。
- SafeCollector.emit() 调用执行 collect{} 内部的操作

```kotlin
.collect {
    println("Received value: $it")
}

public abstract class AbstractFlow<T> : Flow<T>, CancellableFlow<T> {
    public final override suspend fun collect(collector: FlowCollector<T>) {
        val safeCollector = SafeCollector(collector, coroutineContext)
        try {
            //调用Flow{}中传入的block
            collectSafely(safeCollector)
        } finally {
            safeCollector.releaseIntercepted()
        }
    }
    
    public abstract suspend fun collectSafely(collector: FlowCollector<T>)
}

internal actual class SafeCollector<T> actual constructor(
    @JvmField internal actual val collector: FlowCollector<T>,
    @JvmField internal actual val collectContext: CoroutineContext
) : FlowCollector<T>, ContinuationImpl(NoOpContinuation, EmptyCoroutineContext), CoroutineStackFrame {
    ...
    ////Flow{}调用emit()，实际运行的方法
    override suspend fun emit(value: T) {
        return suspendCoroutineUninterceptedOrReturn sc@{ uCont ->
            try {
                //发射数据
                emit(uCont, value)
            } catch (e: Throwable) {
                lastEmissionContext = DownstreamExceptionElement(e)
                throw e
            }
        }
    }

    //内部方法
    private fun emit(uCont: Continuation<Unit>, value: T): Any? {
        val currentContext = uCont.context
        currentContext.ensureActive()
        val previousContext = lastEmissionContext
        if (previousContext !== currentContext) {
            checkContext(currentContext, previousContext, value)
        }
        completion = uCont
        return emitFun(collector as FlowCollector<Any?>, value, this as Continuation<Unit>)
    }

    ...
}
```

总结以上，下游的传入的 FlowCollector 对象(collect {})，经过封装后，在上游(Flow{})内部调用 emit() 传入数据。对于添加了多层操作符的流程来说，调用中间操作符filter{}会创建出新的Flow对象，而且会对数据重新进行发射。

#### StateFlow 

StateFlow 是一个可观察数据可响应流，可以向其收集器发出当前状态更新。与 LiveData 的功能相似，与 Flow 相比，它多出了更新数据内容的功能。使用 StateFlow 一般分为三步：

- 创建 MutableStateFlow 对象并设置初始默认值
- 调用 collect() )收集数据流
- MutableStateFlow  调用 setValue() 修改数据，引起 collect 内的影响

```kotlin
runBlocking {
    val selected = MutableStateFlow<String>("init")
    launch {
        selected.collect{
            println("MutableStateFlow Received value: $it")
        }
    }
    delay(1000L)
    selected.value = "changed"
}
```

需要注意的是，StateFlow 不会自动处理生命周期，因此需要配合 LifecycleCOroutineScope 或launchWhenStarted、launchWhenResumed、launchWhenCreated 等扩展函数来确保在正确的生命周期范围内收集数据。

#### ShareFlow

ShareFlow 也是一个可观察数据可响应流，和 StateFlow 类似，都可以用来存储状态。而且它还可以将已经发送过的数据发送新的订阅者。使用 ShareFlow 一般分为三步：

- 创建一个 MutableSharedFlow 对象并配置参数
- 使用 emit() 或者 tryEmit() 发送数据
- 调用 collect() 数据流

```kotlin
val selected = MutableSharedFlow<Int>(3, 6, BufferOverflow.DROP_OLDEST)
launch {
    for (i in 0..10) {
        selected.tryEmit(i)
    }
}
selected.collect {
    println("MutableSharedFlow Received value: $it")
}
```

MutableSharedFlow 构造函数三个参数的含义：

- replay：当新的订阅者 collect 时，发送几个已经发送过的数据给它
- extraBufferCapacity：减去replay，MutableSharedFlow 还缓存多少数据
- onBufferOverflow：缓存策略，SUSPEND(丢弃最旧值并挂起)、DROP_OLDEST(丢弃最旧值)、DROP_LATEST(丢弃最新值)

```
public fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
): MutableSharedFlow<T> 
```

MutableSharedFlow 发送数据时，当缓存数据量超过阈值，以下两中发送方法处理方式会不一样：

- emit：当缓存策略为 BufferOverflow.SUSPEND 时，emit 方法会挂起，直到有新的缓存空间。
- tryEmit：tryEmit 会返回一个 Boolean 值，true 代表传递成功，false 代表会产生一个回调，让这次数据发射挂起，直到有新的缓存空间。

如果想要将 Flow 转为 SharedFlow，可以使用扩展方法 Flow.shareIn()，第三个参数启动方式 started  提供了三种策略：

- SharingStarted.WhileSubscribed()：存在订阅者时，将使上游提供方保持活跃状态
- SharingStarted.Eagerly：立即启动提供方
- SharingStarted.Lazily：在第一个订阅者出现后开始共享数据，并使数据流永远保持活跃状态

```kotlin
val latestNews: Flow<List<ArticleHeadline>> = flow {
    ...
}.shareIn(
    externalScope,
    replay = 1,
    started = SharingStarted.WhileSubscribed() // 启动政策
)
```





[揭秘Kotlin协程中的CoroutineContext (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650255071&idx=1&sn=03e3c9b1157b8f277d519e61b18a9946&chksm=88635fb0bf14d6a67f7901ffac8a7f544b752fb63adc6c9e5e042d610e9ca8d05ec3c2750bfe)

[Kotlin协程专栏 - 西瓜king的专栏 - 掘金 (juejin.cn)](https://juejin.cn/column/7090416754840567816)

[一看就会！协程原来是这样啊~ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650254892&idx=1&sn=e5cae8e0319596391a734d5f51116022&chksm=88635f43bf14d6558cc60edc3894692d4f503000c5ce7a87428971e4c914ced773c12eb621f7)

[协程到底是怎么切换线程的？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650257918&idx=1&sn=0bde010f58df22508907e213980eb1d6&chksm=88634891bf14c187ac0ad10562c26476a6caf531ded0a772e4bd29bfae23f2accbcebbd04668)

[Kotlin Flow啊，你将流向何方？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650847978&idx=1&sn=54e84aaaf36b4a9328943e80a1545b7a&chksm=80b76e74b7c0e762da77d2edead0a7136071d999865b3b3a50c8a2a8e8bf201be6881e9654e3&mpshare=1&scene=1&srcid=12278Qz5uCDTfQfTEW1Dknbc&sharer_sharetime=1672103282043&sharer_shareid=6abfe554f60ba3cca7d5d67848a573e9#rd)

[Kotlin协程中的并发问题解决方案 (qq.com)](https://mp.weixin.qq.com/s/6paEFQDD-lHYMjcWmwZdhw)

[Kotlin Flow响应式编程，操作符函数进阶 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650270522&idx=1&sn=b05ff0909b3454cfb5e17bbfd4a37f60&chksm=88631a55bf149343b7bcfefa54b563df66a3b5fb7848c296dc50b964a3e06770241c6aa42a15&mpshare=1&scene=1&srcid=1122li34Ak988iuHIGorB1Th&sharer_sharetime=1669080565266&sharer_shareid=6abfe554f60ba3cca7d5d67848a573e9#rd)

[Kotlin协程的取消机制 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650270262&idx=1&sn=2b511a7d84aa64e1adb92143757bb199&chksm=88631b59bf14924f8fea009dd8615beb7a407ced41061ecc8f412a501877016932c3927a1c83&mpshare=1&scene=1&srcid=1111XDlg1mzzIrL9UgMZQRsu&sharer_sharetime=1668125147027&sharer_shareid=6abfe554f60ba3cca7d5d67848a573e9#rd)

[使用协程，让网络世界更加美好 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650253993&idx=1&sn=a23168bc0d7489ce88e056692cb20ace&chksm=88635bc6bf14d2d045c13c6d024e348b74b4b286472afbe5442fab806b5853051ece1d0d16fc)