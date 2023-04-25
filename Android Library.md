# Android Library

## LeakCanary

LeakCanary 是一个用于检测 Android 应用内存泄漏的工具。

### 实现原理

LeakCanary 的实现原理是基于引用队列 (ReferenceQueue)。软引用、弱引用、虚引用 (Reference) 可以配合引用队列一起使用用于跟踪这些引用对象在垃圾回收过程中的状态。Reference 内存存在四种状态：

- Active：一般说来内存一开始被分配的状态
- Pending 快要放入队列（ReferenceQueue）的对象，也就是马上要回收的对象
- Enqueued 对象已经进入队列，已经被回收的对象。方便我们查询某个对象是否被回收
- Inactive 最终的状态，无法变成其他的状态

当创建软引用、弱引用、虚引用时可以设置引用队列，在 Reference 被回收的时候，Reference 会被添加到引用队列中。

```java
ReferenceQueue queue = new ReferenceQueue(); 
//创建弱引用，此时状态为Active，并且Reference.pending为空
WeakReference reference = new WeakReference(new Object(), queue); 
// 当GC执行后，由于是弱引用，所以回收该object对象，此时reference的状态为PENDING  
System.gc();  
//此时Reference状态为ENQUEUED，Reference.queue = ReferenceENQUEUED 
//当从queue里面取出该元素，则变为INACTIVE，Reference.queue = Reference.NULL  
Reference reference1 = queue.remove();
```

LeakCanary 监听 Activity 的生命周期，在 onDestory() 中创建对应 Activity 的 Reference 和 ReferenceQueue，启动后台线程检测。在经过一段时间后读取 ReferenceQueue，如果 Activity 的 Reference 没有出现在队列中，说明存在内存泄漏。

### 源码分析

在 build.gradle 中引入 LeakCanary 依赖

```groovy
implementation 'com.squareup.leakcanary:leakcanary-android:2.7'
```

在 ActivityThread.attach() 内调用 installContentProvider() 创建和初始化 ContentProvider，ContentProvider 的创建发生在 Application 对象创建之后，但在 Application 的 onCreate() 方法调用之前。

LeakCanary 中有一个 ContentProvider，在 onCreate() 方法中开启了内存泄漏监测。

```
internal sealed class AppWatcherInstaller : ContentProvider() {

  internal class MainProcess : AppWatcherInstaller()
  internal class LeakCanaryProcess : AppWatcherInstaller()

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }
}

fun manualInstall(
  application: Application,
  retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
  watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application
) {
  checkMainThread()
  if (isInstalled) {
    throw IllegalStateException(
      "AppWatcher already installed, see exception cause for prior install cal
    )
  }
  ...
  LeakCanaryDelegate.loadLeakCanary(application)
  //实际设置监听，来自appDefaultWatchers()
  watchersToInstall.forEach {
    it.install()
  }
}
```

在 AppWatcherInstaller() 中设置了 Activity，Fragment，RootView，Service 的内存泄漏监听。

```kotlin
fun appDefaultWatchers(
  application: Application,
  reachabilityWatcher: ReachabilityWatcher = objectWatcher
): List<InstallableWatcher> {
  return listOf(
    ActivityWatcher(application, reachabilityWatcher),
    FragmentAndViewModelWatcher(application, reachabilityWatcher),
    RootViewWatcher(reachabilityWatcher),
    ServiceWatcher(reachabilityWatcher)
  )
}

class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        //Activity销毁时执行
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}

val objectWatcher = ObjectWatcher(
  clock = { SystemClock.uptimeMillis() },
  checkRetainedExecutor = {
    check(isInstalled) {
      "AppWatcher not installed"
    }
    //把runnable post 5s后执行
    mainHandler.postDelayed(it, retainedDelayMillis)
  },
  isEnabled = { true }
)

@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
  if (!isEnabled()) {
    return
  }
  removeWeaklyReachableObjects()
  val key = UUID.randomUUID()
    .toString()
  val watchUptimeMillis = clock.uptimeMillis()
  //创建KeyedWeakReference，KeyedWeakReference继承于WeakReference。
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  // 根据key加入map
  watchedObjects[key] = reference
  //会延迟5s
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}
```

## BlockCanary

BlockCanary 是监测 UI 线程阻塞的工具。

### 实现原理

BlockCanary 的实现原理是基于 Looper.loop() 中的 Printer 对象。在 dispatchMessage() 之前之后都有打印操作。

- Looper 设置 MessageLogging 对象 mainLooperPrinter
- mainLooperPrinter 中判断 start 和 end，来获取主线程 dispatch 该 message 的开始和结束时间
- 当时间超过阈值，通过 Thread.getStackTrace() 获取堆栈信息

```java
public static void loop() {
    ...
    for (;;) {
        ...
        //dispatch前打印
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        msg.target.dispatchMessage(msg);
        //dispatch后打印
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        ...
    }
}
```

## EventBus

EventBus 是一个 Android 和 Java 的发布/订阅（Publish/Subscribe）事件总线库，用于在组件之间实现解耦的通信。

### 实现原理

EventBus 的实例获取可以使用默认的全局单例模式，也可以使用 Builder 模式进行配置特定行为。

当注册时，通过反射获取当前类带有 @Subscribe 注解的方法，并将当前类和方法添加到订阅者列表中。当调用 post 方法发布事件时，EventBus 会遍历订阅者列表，找到所有订阅了该事件的方法，并根据 @Subscribe 注解中的 threadMode 属性确定事件的处理线程。然后通过反射调用方法

EventBus 支持多种线程模式（如：MAIN, POSTING, BACKGROUND, ASYNC），可以让订阅者在不同的线程中处理事件。EventBus 通过线程池和 Android 的 Handler 机制实现线程切换。

### 源码分析

在注册时主要完成查找注解方法和保存 subscriber 与注解方法的对应关系

```java
 public void register(Object subscriber) {
     Class<?> subscriberClass = subscriber.getClass();
     //反射找到@Subscribe注解的方法
     List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
     synchronized (this) {
         for (SubscriberMethod subscriberMethod : subscriberMethods) {
             subscribe(subscriber, subscriberMethod);
         }
     }
 }
 
 private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    //封装subscriber对象和注解方法subscriberMethod
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    ...
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            //保存Subscription对象
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    ...
}
```

当发送事件的是主要完成了查找对应注解方法并获取 subscriber 实例，然后在不同的线程内反射调用方法。

```java
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    ...
    while (!eventQueue.isEmpty()) {
   	    postSingleEvent(eventQueue.remove(0), postingState);
    }
}

private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    ...
    subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    ....
}

private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    ...
    postToSubscription(subscription, event, postingState.isMainThread);
    ....
}

private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

## OkHttp

OkHttp 需要先创建一个 OkHttpClient 对象，主要用于发送网络请求和处理响应。OkHttpClient 具有很高的可配置性，可以根据需要对其进行定制。以下是一些常见的 OkHttpClient 配置：

- 连接超时（connectTimeout）： 设置建立连接的超时时间。如果在指定时间内无法建立连接，则请求会失败
- 读超时（readTimeout）： 设置读取数据的超时时间。如果在指定时间内无法读取服务器返回的数据，则请求会失败
- 写超时（writeTimeout）： 设置向服务器发送数据的超时时间。如果在指定时间内无法将数据写入服务器，则请求会失败
- 拦截器（Interceptor）： 可以添加多个拦截器，用于处理请求或响应的数据。例如，可以添加一个日志拦截器以记录请求和响应的详细信息
- 证书引擎（sslSocketFactory 和 hostnameVerifier）： 可以定制 SSL 握手和主机名验证的处理，用于处理自签名证书、证书链、证书颁发机构等
- 连接池（connectionPool）： 可以配置连接池的大小和连接的存活时间，以实现连接的复用，减少连接建立和释放的开销
- Cookie 管理（cookieJar）： 可以设置一个 CookieJar 用于自动处理请求和响应中的 Cookie，实现会话保持等功能
- 代理（proxy）： 可以设置一个代理服务器，所有的请求都将通过这个代理服务器进行

```java
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(15, TimeUnit.SECONDS)
    .addInterceptor(new LoggingInterceptor())
    .sslSocketFactory(sslSocketFactory, trustManager)
    .hostnameVerifier(hostnameVerifier)
    .connectionPool(new ConnectionPool(5, 5, TimeUnit.MINUTES))
    .cookieJar(new MyCookieJar())
    .proxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxy.example.com", 8080)))
    .retryOnConnectionFailure(true)
    .build();
```

Client 调用 newCall() 会创建一个 RealCall 对象，该对象提供同步和异步两种请求方式，完成对请求数量等限制。然后异步方式通过 OkHttpClient 的 dispatcher 来完成线程的切换。

```java
internal fun enqueue(call: AsyncCall) {
  synchronized(this) {
    readyAsyncCalls.add(call)
  }
  promoteAndExecute()
}

private fun promoteAndExecute(): Boolean {
  this.assertThreadDoesntHoldLock()
  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    val i = readyAsyncCalls.iterator()
    while (i.hasNext()) {
      val asyncCall = i.next()
      //数量限制
      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.
      i.remove()
      asyncCall.callsPerHost.incrementAndGet()
      //遍历enqueue中添加的AsyncCall对象添加到executableCalls
      executableCalls.add(asyncCall)
      runningAsyncCalls.add(asyncCall)
    }
    isRunning = runningCallsCount() > 0
  }
  for (i in 0 until executableCalls.size) {
    val asyncCall = executableCalls[i]
    //遍历executableCalls中的asyncCall在executorService线程池中执行
    asyncCall.executeOn(executorService)
  }
  return isRunning
}
```

在线程池中执行，会调用 RealCall 封装起来的 AsyncCall.run()，构建出 RealInterceptorChain 链表，开启请求过程。

```java
override fun run() {
  threadName("OkHttp ${redactedUrl()}") {
    var signalledCallback = false
    timeout.enter()
    try {
      val response = getResponseWithInterceptorChain()
      signalledCallback = true
      responseCallback.onResponse(this@RealCall, response)
    } catch (e: IOException) {
    } catch (t: Throwable) {
    } finally {
      client.dispatcher.finished(this)
    }
  }
}

internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = mutableListOf<Interceptor>()
  interceptors += client.interceptors
  interceptors += RetryAndFollowUpInterceptor(client)
  interceptors += BridgeInterceptor(client.cookieJar)
  interceptors += CacheInterceptor(client.cache)
  interceptors += ConnectInterceptor
  if (!forWebSocket) {
    interceptors += client.networkInterceptors
  }
  interceptors += CallServerInterceptor(forWebSocket)
  val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
  )
  var calledNoMoreExchanges = false
  try {
    //开启请求流程
    val response = chain.proceed(originalRequest)
    return response
  } catch (e: IOException) {
  } finally {
  }
}
```

OkHttp 默认的五个拦截器作用是

- RetryAndFollowUpInterceptor： 重试和跟踪拦截器，处理请求失败的重试和处理重定向。当网络请求失败时，该拦截器会根据配置的重试次数和策略自动重试请求
- BridgeInterceptor： 桥接拦截器，负责处理请求头和响应头的一些通用操作
- CacheInterceptor： 缓存拦截器，负责处理 HTTP 缓存逻辑。根据请求的缓存策略和服务器返回的缓存控制头部，该拦截器会决定是否使用缓存的响应，还是发起新的网络请求。同时，它还会负责将服务器返回的响应写入缓存
- ConnectInterceptor： 连接拦截器，负责建立底层的网络连接。它会根据请求的 URL 和 OkHttpClient 配置的连接池策略来获取一个可用的连接，然后将连接传递给后续的拦截器
- CallServerInterceptor： 调用服务器拦截器，负责将处理后的请求发送到服务器，并接收服务器返回的响应

## Glide

Glide 是一个用于 Android 的开源图像加载和缓存库，它可以绑定生命周期，免了在生命周期变化时可能导致的内存泄漏、图片请求无法取消等问题，并且拥有高效的缓存策略。

 Glide 通过 RequestManager 来管理图片请求。RequestManager 与 Activity、Fragment 的生命周期关联，以便在生命周期变化时自动处理请求的暂停和恢复。

- 对于 Activity： Glide 使用一个透明的 Fragment（RequestManagerFragment）附加到 Activity。这个 Fragment 会监听 Activity 的生命周期事件，如 onStart、onStop 和 onDestroy。当这些事件发生时，RequestManagerFragment 会相应地通知 RequestManager。

- 对于 Fragment： Glide 使用一个子 Fragment（RequestManagerFragment）附加到宿主 Fragment。这个子 Fragment 会监听宿主 Fragment 的生命周期事件，如 onStart、onStop 和 onDestroy。当这些事件发生时，RequestManagerFragment 会相应地通知 RequestManager。

```
public static RequestManager with(@NonNull Activity activity) {
  return getRetriever(activity).get(activity);
}

public RequestManager get(@NonNull Activity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else if (activity instanceof FragmentActivity) {
    return get((FragmentActivity) activity);
  } else {
    assertNotDestroyed(activity);
    frameWaiter.registerSelf(activity);
    android.app.FragmentManager fm = activity.getFragmentManager();
    return fragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}

public RequestManager get(@NonNull FragmentActivity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    frameWaiter.registerSelf(activity);
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}
```

Glide 的缓存策略分为两个层次：内存缓存和磁盘缓存。这两个缓存层次协同工作，以提高图片加载速度，减少网络请求和解码开销，并优化内存占用。

内存缓存主要用于缓存解码后的 Bitmap 对象，以便快速地在 UI 中显示。当需要加载一张图片时，Glide 首先会检查内存缓存中是否存在该图片。如果存在，它将直接从内存缓存中获取并显示，避免了重新解码的开销。内存缓存是 LRU（Least Recently Used，最近最少使用）策略，当缓存达到设定的最大容量时，最近最少使用的图片将被移除以释放空间。

磁盘缓存用于缓存原始的图片文件（如从网络下载的 JPEG 或 PNG 文件）。当内存缓存中没有找到所需的图片时，Glide 会检查磁盘缓存。如果磁盘缓存中存在该图片，Glide 会从磁盘缓存中读取并解码，然后将解码后的 Bitmap 对象缓存在内存中。磁盘缓存同样采用 LRU 策略，当缓存达到最大容量时，最近最少使用的图片文件将被移除。