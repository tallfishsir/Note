# Android Jetpack

## Lifecycle

Lifecycle 组件的作用是存储具有生命周期的组件的生命周期状态，并且作为一个被观察者，允许其他组件保持对生命周期状态的观察。主要目的是将生命周期感知逻辑和 UI 界面解耦

### 基本使用

在 AppCompatActivity 中获取 Lifecycle 对象然后添加 LIfeObserver 实现类。在 LIfeObserver 实现类中使用OnLIfecycleEvent 注解监测对应的生命周期。

```kotlin
class LifecycleTestActivity  : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_kotlin)

        lifecycle.addObserver(CustomLifecycleObserver())
    }
    ...
}

class CustomLifecycleObserver : LifecycleObserver {
    @OnLifecycleEvent(value = Lifecycle.Event.ON_RESUME)
    fun connect(lifecycleOwner: LifecycleOwner) {
        if (lifecycleOwner.lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
        }
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_PAUSE)
    fun disconect() {
    }
    ...
}
```

### 事件流转源码分析

Lifecycle 组件中有三个重要角色，分解扮演生命周期事件的生产者，管理者和消费者。LifecycleOwner 负责发出生命周期事件，Lifecycle 负责存储和调度，LifecycleObserver 负责监听和消费生命周期事件。

- LifecycleOwner：单一属性接口，表示实现该接口的类具有生命周期，生命周期发生变化时，会通知 Lifecycle
- Lifecycle：同时持有 LifecycleOwner 的软引用和 LifecycleObserver 的强引用，保存了 LifecycleOwner 的生命周期，当发生变化街道 LifecycleOwner 通知后，会通知 LifecycleObserver 
- LifecycleObserver：生命周期的观察者

#### LifecycleOwner

LifecycleOwner 是一个单一属性的接口，用来标记实现类持有 Lifecycle 对象，可以获取一个 Lifecycle 对象。并且提供了一个扩展属性 lifecycleScope，用于获取生命周期的协程

```kotlin
public interface LifecycleOwner {
    public val lifecycle: Lifecycle
}

public val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope
    

public val Lifecycle.coroutineScope: LifecycleCoroutineScope
    get() {
        while (true) {
            val existing = internalScopeRef.get() as LifecycleCoroutineScopeImpl?
            if (existing != null) {
                return existing
            }
            val newScope = LifecycleCoroutineScopeImpl(
                this,
                SupervisorJob() + Dispatchers.Main.immediate
            )
            if (internalScopeRef.compareAndSet(null, newScope)) {
                newScope.register()
                return newScope
            }
        }
    }
```

#### Lifecycle

Lifecycle 是一个抽象类，定义了两个抽象方法用于添加和删除 LifecycleObserver，还定义了两个表示生命周期的枚举类。

```kotlin
public abstract class Lifecycle {
    //添加LifecycleObserver
    @MainThread
    public abstract fun addObserver(observer: LifecycleObserver)

    //移除LifecycleObserver
    @MainThread
    public abstract fun removeObserver(observer: LifecycleObserver)

    //返回当前LifecycleOwner生命周期状态
    @get:MainThread
    public abstract val currentState: State

    public enum class Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY;

        public val targetState: State
            get() {
                when (this) {
                    ON_CREATE, ON_STOP -> return State.CREATED
                    ON_START, ON_PAUSE -> return State.STARTED
                    ON_RESUME -> return State.RESUMED
                    ON_DESTROY -> return State.DESTROYED
                    ON_ANY -> {}
                }
                throw IllegalArgumentException("$this has no target state")
            }
        }
    }
    
    public enum class State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        public fun isAtLeast(state: State): Boolean {
            return compareTo(state) >= 0
        }
    }
}
```

Event 枚举类用于表示生命周期事件，State 枚举类表示生命周期状态，两者之间的关系可以用下图总结：

![image-20210831222230669](https://pic.imgdb.cn/item/612e3b2544eaada7390e1a01.png)

#### LifecycleRegister

LifecycleRegister 是 Lifecycle 的唯一子类，它主要实现了记录组件生命周期状态，保存和通知 LifecycleObserver 的功能。

- LifecycleRegistry 并不会直接持有 LifecycleObserver，而是通过将 LifecycleObserver 作为 Key 值，使用ObserverWithState 存储在 FastSafeIterableMap中。ObserverWithState 持有 LifecycleObserver，并会持有一个生命周期状态，表示LifecycleObserver当前所收到的状态，用此状态和组件的生命周期状态进行比对，最终判断状态时升级还是降级或者维持不变
- 状态有两种改变方式，但是都是沿着顺序改变(DESTROYED;INITIALIZED;CREATED;CREATED;STARTED;RESUMED)一种是通过forwardPass进行升状态，只处理向RESUMED方向转化的状态，一种是backwardPass进行降状态，只处理向DESTROYED方向转化的状态。前者采用正序通知观察者回调，后者逆序
- 状态时逐级分发调用的，不会跨级分发。状态改变后（或者添加监听时），相差多少级就会通过补发Event的方式触发多次分发以及回调

```kotlin
open class LifecycleRegistry private constructor(
    provider: LifecycleOwner,
    private val enforceMainThread: Boolean
) : Lifecycle() {
    //保存绑定的LifecycleObserve对象
    private var observerMap = FastSafeIterableMap<LifecycleObserver, ObserverWithState>()

    //LifecycleOwner当前的生命周期状态
    private var state: State = State.INITIALIZED
    //通过弱引用的方式持有LifecycleOwner
    private val lifecycleOwner: WeakReference<LifecycleOwner>
    
    //正在进行绑定的观察者数量
    private var addingObserverCounter = 0
    //是否正在处理生命周期事件
    private var handlingEvent = false
    //标记有新事件发生
    private var newEventOccurred = false

    constructor(provider: LifecycleOwner) : this(provider, true)
    init {
        lifecycleOwner = WeakReference(provider)
    }
    override var currentState: State
        get() = state
        set(state) {
            enforceMainThreadIfNeeded("setCurrentState")
            moveToState(state)
        }
    open fun handleLifecycleEvent(event: Event) {
        enforceMainThreadIfNeeded("handleLifecycleEvent")
        moveToState(event.targetState)
    }

    //根据当前的生命周期事件，设置当前的状态并通知观察者
    private fun moveToState(next: State) {
        //如果当前状态和接收到的状态一致，那么直接返回不做任何处理
        if (state == next) {
            return
        }
        //修改当前的生命周期状态
        state = next
        //如果正在分发状态或者有观察者正在添加，则标记有新事件发生
        if (handlingEvent || addingObserverCounter != 0) {
            newEventOccurred = true
            return
        }
        //标记正在处理事件
        handlingEvent = true
        //开始同步生命周期状态
        sync()
        //还原标记
        handlingEvent = false
        if (state == State.DESTROYED) {
            observerMap = FastSafeIterableMap()
        }
    }
    
    //添加LifecycleObserver
    override fun addObserver(observer: LifecycleObserver) {
        enforceMainThreadIfNeeded("addObserver")
        //如果当前组件不处于DESTROYED状态，那么标记LifecycleObserver的初始状态为INITIALIZED
        val initialState = if (state == State.DESTROYED) State.DESTROYED else State.INITIALIZED
        //创建ObserverWithState，为LifecycleObserver赋予状态
        val statefulObserver = ObserverWithState(observer, initialState)
        //添加到容器mObserverMap中
        val previous = observerMap.putIfAbsent(observer, statefulObserver)
        //如果previous不为null，表示重复添加，直接结束
        if (previous != null) {
            return
        }
        
        //判断LifecycleOwner是否已经被销毁
        val lifecycleOwner = lifecycleOwner.get()
            ?: // it is null we should be destroyed. Fallback quickly
            return
        
        //判断是否是重入状态
        val isReentrance = addingObserverCounter != 0 || handlingEvent
        //计算状态
        var targetState = calculateTargetState(observer)
         //标记正在添加的LifecycleObserver的数量
        addingObserverCounter++
        //通过一个while循环来将LifecycleObserver的状态设置为最新
        while (statefulObserver.state < targetState && observerMap.contains(observer)
        ) {
            pushParentState(statefulObserver.state)
            val event = Event.upFrom(statefulObserver.state)
                ?: throw IllegalStateException("no event up from ${statefulObserver.state}")
            statefulObserver.dispatchEvent(lifecycleOwner, event)
            popParentState()
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer)
        }
        if (!isReentrance) {
            sync()
        }
        addingObserverCounter--
    }

    //移除观察者LifecycleObserver
    override fun removeObserver(observer: LifecycleObserver) {
        enforceMainThreadIfNeeded("removeObserver")
        observerMap.remove(observer)
    }
    
    private fun sync() {
        val lifecycleOwner = lifecycleOwner.get()
            ?: throw IllegalStateException(
                "LifecycleOwner of this LifecycleRegistry is already " +
                    "garbage collected. It is too late to change lifecycle state."
            )
        //是否同步完成
        while (!isSynced) {
            //第一种是当前的状态值小于之前的状态，则执行回退状态的操作（如CREATED->RESUMED）
            newEventOccurred = false
            if (state < observerMap.eldest()!!.value.state) {
                backwardPass(lifecycleOwner)
            }
            //第二种是当前的状态值大于之前的状态，则执行前进状态的操作（如>RESUMED->CREATED）
            val newest = observerMap.newest()
            if (!newEventOccurred && newest != null && state > newest.value.state) {
                forwardPass(lifecycleOwner)
            }
        }
        newEventOccurred = false
    }
    
    //判断是否同步完成
    //通过判断mObserverMap的最eldest和newest的状态是否和当前状态一致
    //来判断是否是否将状态同步到了所有观察者
    private val isSynced: Boolean
        get() {
            if (observerMap.size() == 0) {
                return true
            }
            val eldestObserverState = observerMap.eldest()!!.value.state
            val newestObserverState = observerMap.newest()!!.value.state
            return eldestObserverState == newestObserverState && state == newestObserverState
        }

    private fun forwardPass(lifecycleOwner: LifecycleOwner) {
        @Suppress()
        val ascendingIterator: Iterator<Map.Entry<LifecycleObserver, ObserverWithState>> =
            observerMap.iteratorWithAdditions()
        while (ascendingIterator.hasNext() && !newEventOccurred) {
            val (key, observer) = ascendingIterator.next()
            //如果观察者所处的状态和组件的最新状态跨等级，则一步一步的升级而不会跨状态提升
            while (observer.state < state && !newEventOccurred && observerMap.contains(key)
            ) {
                pushParentState(observer.state)
                val event = Event.upFrom(observer.state)
                    ?: throw IllegalStateException("no event up from ${observer.state}")
                //此处的observer是一个ObserverWithState类型，它是LifecycleRegistry的静态内部类
                observer.dispatchEvent(lifecycleOwner, event)
                popParentState()
            }
        }
    }

    private fun backwardPass(lifecycleOwner: LifecycleOwner) {
        val descendingIterator = observerMap.descendingIterator()
        while (descendingIterator.hasNext() && !newEventOccurred) {
            val (key, observer) = descendingIterator.next()
            while (observer.state > state && !newEventOccurred && observerMap.contains(key)
            ) {
                val event = Event.downFrom(observer.state)
                    ?: throw IllegalStateException("no event down from ${observer.state}")
                pushParentState(event.targetState)
                observer.dispatchEvent(lifecycleOwner, event)
                popParentState()
            }
        }
    }
}
```

#### LifecycleObserver

LifecycleObserver 是一个接口，起到类型或者身份标记的作用，本身是一个空接口，所有的任务基本都靠它的具体实现类完成。

```kotlin
public interface LifecycleObserver

public fun interface LifecycleEventObserver : LifecycleObserver {
    public fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event)
}
```

### 事件产生源码分析

常见的生命周期事件产生者有 Activity 和 Fragment。比如 ComponentActivity 实现 LifecycleOwner 接口，是它没有直接在自己的生命周期方法里调用 handleLifecycleEvent() 修改生命周期状态，而是通过 ReportFragment 进行的。

```kotlin
public class ComponentActivity extends androidx.core.app.ComponentActivity implements LifecycleOwner{
    ...
    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    ...
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mSavedStateRegistryController.performRestore(savedInstanceState);
        ReportFragment.injectIfNeededIn(this); //使用ReportFragment分发生命周期事件
    }
    ...
}
```

因为 Fragment 的生命周期都是依附 Activity 的，所以 ReportFragment  是没有布局的，它的作用就是获取生命周期。在 ReportFragment 内部不同的生命周期最后会在 dispatch() 内调用 LifecycleRegistry.handleLifecycleEvent() 分发事件。

```java
public class ReportFragment extends Fragment {
    
    public static void injectIfNeededIn(Activity activity) {
        if (Build.VERSION.SDK_INT >= 29) {
            //在API 29及以上，可以直接注册回调 获取生命周期
            activity.registerActivityLifecycleCallbacks(
                    new LifecycleCallbacks());
        }
        //API29以前，使用fragment 获取生命周期
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
    }

    @SuppressWarnings("deprecation")
    static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
        if (activity instanceof LifecycleRegistryOwner) {//这里废弃了，不用看
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);//使用LifecycleRegistry的handleLifecycleEvent方法处理事件
            }
        }
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatch(Lifecycle.Event.ON_CREATE);
    }
    @Override
    public void onStart() {
        super.onStart();
        dispatch(Lifecycle.Event.ON_START);
    }
    @Override
    public void onResume() {
        super.onResume();
        dispatch(Lifecycle.Event.ON_RESUME);
    }
    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }
    ...省略onStop、onDestroy
    
    //在API 29及以上，使用的生命周期回调
    static class LifecycleCallbacks implements Application.ActivityLifecycleCallbacks {
        ...
        @Override
        public void onActivityPostCreated(@NonNull Activity activity,@Nullable Bundle savedInstanceState) {
            dispatch(activity, Lifecycle.Event.ON_CREATE);
        }
        @Override
        public void onActivityPostStarted(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_START);
        }
        @Override
        public void onActivityPostResumed(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_RESUME);
        }
        @Override
        public void onActivityPrePaused(@NonNull Activity activity) {
            dispatch(activity, Lifecycle.Event.ON_PAUSE);
        }
        ...省略onStop、onDestroy
    }
}
```

## LiveData

LiveData 是一个不可变的数据持有者类，用于在 ViewModel 和 UI 组件之间传递数据。LiveData 对象可以被观察，但不能直接更改其值，可以通过 LiveData 的 observe 和 observeForever 方法来观察数据的变化，需要注意的是 LiveData 的观察始终是在主线程上进行的。

MutableLiveData 是 LiveData 的子类，它允许更改和更新数据。MutableLiveData 可以被观察，也可以修改其值。可以使用 setValue 和 postValue 方法来更新数据。

在实际应用中，通常会在 ViewModel 内部使用 MutableLiveData 来存储和管理数据，然后将其作为 LiveData 对象对外暴露给 UI 组件（如 Activity 和 Fragment）。这样可以确保数据的封装性和不可变性，防止外部组件意外地修改数据。

```kotlin
class MyViewModel : ViewModel() {
    private val _data = MutableLiveData<String>()
    val data: LiveData<String> get() = _data

    fun updateData(newValue: String) {
        _data.value = newValue
    }
}
```

### 基本使用

使用 LiveData 的过程分为以下几步：

- 创建 LiveData 实例，通过泛型指定源数据类型
- 创建 Observer 实例，实现 onChanged() 方法，用于接收源数据变化
- LiveData 实例使用 observe() 方法添加 Observer 实例观察者，并传入 LifecycleOwner
- LiveData 实例使用 setValue()/postValue() 更新源数据

```kotlin
class MyActivity : AppCompatActivity() {
    private lateinit var viewModel: MyViewModel
    private val lifecycleScope = LifecycleCoroutineScope(this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        viewModel = ViewModelProvider(this).get(MyViewModel::class.java)
        lifecycleScope.launchWhenStarted {
            viewModel.data.observe(this@MyActivity) { newData ->
		   }
        }
        
        //activity是非活跃状态，不会回调onChanged。变为活跃时，value被onStart中的value覆盖
        viewModel.data.value = "onCreate"
    }
    
    override fun onStart(){
        //活跃状态，会回调onChanged。并且value会覆盖onCreate、onStop中设置的value
        viewModel.data.value = "onStart"
    }
    
    override fun onResume(){
        //活跃状态，回调onChanged
        viewModel.data.value = "onResume"
    }
    
    override fun onPause(){
        //活跃状态，回调onChanged
        viewModel.data.value = "onPause"
    }
    
    override fun onStop(){
        //非活跃状态，不会回调onChanged。后面变为活跃时，value被onStart中的value覆盖
        viewModel.data.value = "onStop"
    }
    
    override fun onDestroy(){
        //非活跃状态，且此时Observer已被移除，不会回调onChanged
        viewModel.data.value = "onDestroy"
    }
}
```

当发送数据时，有以下两个方法可以使用：

- setValue(T): 用于在主线程中更新 LiveData 的值。更新的值将同步地发送给观察者
- postValue(T): 用于在非主线程中更新 LiveData 的值。更新的值将异步地发送给观察者

当接收数据时，有以下两个方法可以使用：

- observe(LifecycleOwner, Observer): 只有当 LifecycleOwner 处于活动状态，Observer 才会通知更新
- observeForever(Observer): 用于在非 LifecycleOwner 环境中观察 LiveData 对象。观察者会被视为始终处于活跃状态

### MediatorLiveData

MediatorLiveData 是 LiveData 的子类，允许合并多个 LiveData 对线的数据，当任意一个 LiveData 发生变化，MediatorLiveData 都会触发观察者。它通过以下两个方法添加和删除源 LiveData

- addSource(source: LiveData<S>, onChanged: (S) -> Unit)：将一个源 LiveData 添加到 MediatorLiveData
- removeSource(source: LiveData<S>)：从 MediatorLiveData 中移除先前添加的源 LiveData

```kotlin
val liveData1: LiveData<Int> = ...
val liveData2: LiveData<Int> = ...

val mediatorLiveData = MediatorLiveData<Int>()

// 添加 liveData1 作为源，并在 liveData1 发生变化时更新 mediatorLiveData
mediatorLiveData.addSource(liveData1) { value ->
    mediatorLiveData.value = value // 将 liveData1 的值设置为 mediatorLiveData 的值
}

// 添加 liveData2 作为源，并在 liveData2 发生变化时更新 mediatorLiveData
mediatorLiveData.addSource(liveData2) { value ->
    mediatorLiveData.value = value // 将 liveData2 的值设置为 mediatorLiveData 的值
}

// 观察 mediatorLiveData，当 liveData1 或 liveData2 发生变化时，UI 会自动更新
mediatorLiveData.observe(lifecycleOwner, Observer { value ->
    // 更新 UI，例如显示最新的值
})
```

### 源码分析

#### 添加 Observer

LiveData 添加 Observer 有两种方式，observe 添加 LifecycleOwner 的 Observer

```java
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    //检查 observe() 是否在主线程上调用
    assertMainThread("observe");
    //命周期已经处于 DESTROYED 状态，不再观察
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        return;
    }
    //LifecycleBoundObserver 封装了observer，用于处理生命周期相关的逻辑
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    //已经有相同的观察者存在，putIfAbsent() 方法会返回现有的观察者包装器
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    //Lifecycle添加观察者，当生命周期状态发生变化时，回调LifecycleBoundObserver.onStateChanged方法
    owner.getLifecycle().addObserver(wrapper);
}

//LifecycleBoundObserver.onStateChanged中会对DESTROYED移除监听
public void onStateChanged(@NonNull LifecycleOwner source,
        @NonNull Lifecycle.Event event) {
    Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
    if (currentState == DESTROYED) {
        removeObserver(mObserver);
        return;
    }
    Lifecycle.State prevState = null;
    while (prevState != currentState) {
        prevState = currentState;
        //状态不一致时，修改当前状态，并且如果在活动状态还会调用dispatchingValue执行observer回调
        activeStateChanged(shouldBeActive());
        currentState = mOwner.getLifecycle().getCurrentState();
    }
}
```

#### 更新数据

LivaData 数据更新可以使用 setValue()  和 postValue() ，postValue 实际是借助 Handler post 到主线程

```java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    //执行observer回调
    dispatchingValue(null);
}

protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    //内部创建DefaultTaskExecutor对象，保存了newFixedThreadPool线程池和mMainHandler
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        setValue((T) newValue);
    }
};
```

#### Observer 回调

当 LifecycleOwner 发生了生命周期状态改变或者 LiveData 数据发生改变，都会调用 dispatchingValue 来执行Observer 回调。

```java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        //如果当前正在分发 return
        mDispatchInvalidated = true;
        return;
    }
    //标记正在分发
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            //observerWrapper不为空，来自生命周期变化导致的分发，使用considerNotify()通知真正的观察者
            considerNotify(initiator);
            initiator = null;
        } else {
            // observerWrapper为空，来自源数据内容变化导致的分发，遍历通知所有的观察者
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        //观察者非活跃 return
        return;
    }
    //若当前observer对应owner非活跃，就会再调用activeStateChanged方法，并传入false，其内部会再次判断
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    ///回调真正的mObserver的onChanged方法
    observer.mObserver.onChanged((T) mData);
}
```

## ViewModel

ViewModel 是一个为界面准备数据的模型，它的生命周期长于 Activity，在 Activity 配置更改（如屏幕旋转）时 ViewModel 对象依然会被保留，可以避免因为屏幕旋转等操作导致的数据丢失和重新加载数据。只有 Activity 真正被 Finish 的时候，ViewModel 才会被清除。

### 基本使用

使用 LiveData 的过程分为以下几步：

- 创建 MyViewModel 类继承 ViewModel 
- 在 MyViewModel 中创建 LiveData 实例，并完成 LiveData 的取值和设值 setValue
- 在 Activity/Fragment 中使用 ViewModelProvider 获取 MyViewModel 实例
- 观察 MyViewModel 中的 LiveData 数据，并进行对应的 UI 更新

```kotlin
class UserViewMode : ViewModel() {
    val userLiveData: MutableLiveData<String> = MutableLiveData()
    val loadingLiveData: MutableLiveData<Boolean> = MutableLiveData()
    private val uiScope = CoroutineScope(Dispatchers.Main)

    fun getUserInfo() {
        loadingLiveData.value = true
        userLiveData.value = "result user"
        loadingLiveData.value = false
    }
}

class KotlinActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_kotlin)

        val viewModelProvider = ViewModelProvider(this, ViewModelProvider.NewInstanceFactory())
        val userViewMode: UserViewMode = viewModelProvider.get(UserViewMode::class.java)
        userViewMode.userLiveData.observe(this) { user ->
            // update ui
        }
        userViewMode.loadingLiveData.observe(this){ showLoading ->
            if (showLoading) {
                // show loading
            } else {
                // hide loading
            }
        }
        userViewMode.getUserInfo()
    }
}
```

需要注意的是，ViewModel 不应该持有 View 或者 Context 的引用，只能通过 LiveData 或者 Flow 更新数据内容，在页面设置数据观察者来监听改变。

创建 ViewModel 实例通过 ViewModelProvider 工具类。它的主要作用是确保在组件的生命周期内 ViewModel 实例保持唯一性和持久性。一般用于多 Fragment 在同一个 Activity 范围内共享 ViewModel 对象通信。

```kotlin
class SharedViewModel : ViewModel() {
    private val _data = MutableLiveData<String>()
    val data: LiveData<String>
        get() = _data

    fun updateData(newData: String) {
        _data.value = newData
    }
}

class MainActivity : AppCompatActivity() {
    private lateinit var viewModel: SharedViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        viewModel = ViewModelProvider(this).get(SharedViewModel::class.java)
    }
}

class FirstFragment : Fragment() {
    private lateinit var viewModel: SharedViewModel

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val view = inflater.inflate(R.layout.fragment_first, container, false)
        viewModel = ViewModelProvider(requireActivity()).get(SharedViewModel::class.java)
        return view
    }
}

class SecondFragment : Fragment() {
    private lateinit var viewModel: SharedViewModel

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val view = inflater.inflate(R.layout.fragment_second, container, false)
        viewModel = ViewModelProvider(requireActivity()).get(SharedViewModel::class.java)
        return view
    }
}
```

### 源码分析

ViewModel 通过 ViewModelProvider.get() 创建实例，ViewModelProvider 的构造函数有三个类型：

- ViewModelStore： 存储 ViewModel 实例的容器，负责处理 ViewModel 的生命周期。当宿主 Activity 或 Fragment 销毁时，ViewModelStore 会清除所有关联的 ViewModel 实例
- Factory：创建 ViewModel 实例的方式，默认的 ViewModelProvider.AndroidViewModelFactory 会实例化具有一个 Application 参数构造函数的 ViewModel
- CreationExtras ：用于传递额外的创建信息给 ViewModel 工厂，默认值为 CreationExtras.Empty

```
constructor(
    private val store: ViewModelStore,
    private val factory: Factory,
    private val defaultCreationExtras: CreationExtras = CreationExtras.Empty,
)
```

ViewModelStore 内部通过 HashMap 存放 ViewModel实例，HashMap 的 key 是通常是由 ViewModel 的类名生成的，value 是对应的 ViewModel 实例。

```kotlin
open class ViewModelStore {

    private val map = mutableMapOf<String, ViewModel>()
    
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    fun put(key: String, viewModel: ViewModel) {
        val oldViewModel = map.put(key, viewModel)
        oldViewModel?.onCleared()
    }
    
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    operator fun get(key: String): ViewModel? {
        return map[key]
    }
    
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    fun keys(): Set<String> {
        return HashSet(map.keys)
    }
    
    fun clear() {
        for (vm in map.values) {
            vm.clear()
        }
        map.clear()
    }
}
```

Activity 的 ViewModelStore 是通过 getViewModelStore() 获取，如果 ViewModelStore 还没有创建，有两个步骤赋值：

- 获取 NonConfigurationInstances 对象，查看它里面是否保存 ViewModelStore
- 如果上一步也不存在 ViewModelStore，就创建新的 ViewModelStore 实例

```java
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        //activity还没关联Application，即不能在onCreate之前去获取viewModel
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    ensureViewModelStore();
    return mViewModelStore;
}

void ensureViewModelStore() {
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
}
```

需要注意的是，Activity 在配置改变销毁重新创建前会调用 onRetainNonConfigurationInstance()，其中会把 ViewModelStore 保存在 NonConfigurationInstances 对象中。NonConfigurationInstances 并不会因为配置改变而销毁，所以当再次调用 getViewModelStore() 时会从 NonConfigurationInstances 获取到之前的 ViewModelStore。

这就是屏幕旋转等的配置改变不会销毁 ViewModel 的原因。

```java
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();
    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }
    }
    if (viewModelStore == null && custom == null) {
        return null;
    }
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    //保存ViewModelStore
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

当得到 ViewModelStore 对象后，ViewModelProvider 会检查 ViewModelStore 中是否已经存在一个指定类型的 ViewModel 实例。如果存在，ViewModelProvider 会直接返回这个实例；如果不存在，ViewModelProvider 会使用提供的工厂方法创建一个新的 ViewModel 实例，并将其存储在 ViewModelStore 中，然后返回这个新创建的实例。

当宿主（如 Activity 或 Fragment）被销毁时，ViewModelStore 会调用 clear() 方法来清除存储的所有 ViewModel 实例。在这个过程中，每个 ViewModel 的 onCleared() 方法都会被调用，这样就可以执行任何必要的资源释放和清理操作。

```kotlin
//ViewModelProvider.kt
public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {
    val viewModel = store[key]
    if (modelClass.isInstance(viewModel)) {
        //获取到ViewModel返回
        (factory as? OnRequeryFactory)?.onRequery(viewModel!!)
        return viewModel as T
    } else {
        ...
    }
    val extras = MutableCreationExtras(defaultCreationExtras)
    extras[VIEW_MODEL_KEY] = key
    return try {
        ///没有获取到，就使用Factory创建
        factory.create(modelClass, extras)
    } catch (e: AbstractMethodError) {
        factory.create(modelClass)
    }.also { store.put(key, it) } //存入ViewModelStore
}
```

## ViewBinding

Android Gradle 插件为每个 XML 布局文件生成了一个 ViewBinding 类，绑定类中包含了布局文件中定义的 android:id 属性的 View 引用。方便直接访问布局中的视图，无需使用 findViewById。开启 ViewBinding 需要在模块级别的 build.gradle 中启用。

```groovy
android {
    ...
    buildFeatures {
        viewBinding true
    }
}
```

### 基本使用

生成的 ViewBinding 类提供了三个静态方法用于创建 ViewBinding 实例，最终会生成 XML 中最外层的 RootView。

```java
public final class ActivityKotlinBinding implements ViewBinding {

  private final ConstraintLayout rootView;
  public final RoundRectImage test;

  private ActivityKotlinBinding(@NonNull ConstraintLayout rootView, @NonNull RoundRectImage test) {
    this.rootView = rootView;
    this.test = test;
  }

  @Override
  //XML布局的最顶层布局
  public ConstraintLayout getRoot() {
    return rootView;
  }

  //传入LayoutInflater，最终调用到bind()
  public static ActivityKotlinBinding inflate(@NonNull LayoutInflater inflater) {
    return inflate(inflater, null, false);
  }

  //传入LayoutInflater，最终调用到bind()
  public static ActivityKotlinBinding inflate(@NonNull LayoutInflater inflater,
      @Nullable ViewGroup parent, boolean attachToParent) {
    View root = inflater.inflate(R.layout.activity_kotlin, parent, false);
    if (attachToParent) {
      parent.addView(root);
    }
    return bind(root);
  }

  //自主完成findViewById，并返回ActivityKotlinBinding对象
  public static ActivityKotlinBinding bind(@NonNull View rootView) {
    int id;
    missingId: {
      id = R.id.test;
      RoundRectImage test = ViewBindings.findChildViewById(rootView, id);
      if (test == null) {
        break missingId;
      }
      return new ActivityKotlinBinding((ConstraintLayout) rootView, test);
    }
    String missingId = rootView.getResources().getResourceName(id);
    throw new NullPointerException("Missing required view with ID: ".concat(missingId));
  }
}
```

在 Activity 中，先使用 ViewBinding.inflate() 加载布局文件，再将 ViewBinding.rootView 传入 setContentView() 中

```java
class TestActivity: AppCompatActivity(R.layout.activity_test) {

    private lateinit var binding: ActivityTestBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = ActivityTestBinding.inflate(layoutInflater)
        setContentView(binding.root)
        binding.tvDisplay.text = "Hello World."
    }
}   
```

在 Fragment 中，调用 ViewBinding.bind() 完成 View 的绑定，在 onDestroyView() 中销毁。

```java
class TestFragment : Fragment(R.layout.fragment_test) {

    private var binding: FragmentTestBinding? = null

    override fun onViewCreated(root: View, savedInstanceState: Bundle?) {
        binding = FragmentTestBinding.bind(root)
        binding.tvDisplay.text = "Hello World."
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        binding = null
    }
}
```

### 委托封装

在 Activity/Fragment 中使用 ViewBinding 需要编写样板代码，特别是 Fragment 中 ViewBinding 的实例是可空可变的，使用起来不方便，所以可以使用委托机制封装起来。

```kotlin
inline fun <Component : Any, V : ViewBinding> viewBindings(
    crossinline viewProvider: (Component) -> View = ::getBindView,
    crossinline viewBinder: (View) -> V
): ViewBindingProperty<Component, V> =
    ActivityViewBindingProperty { component :Component ->
        viewBinder(viewProvider(component))
    }

interface ViewBindingProperty<in R : Any, out V : ViewBinding> : ReadOnlyProperty<R, V> {
    @MainThread
    fun clear()
}

class ActivityViewBindingProperty<A : Any, V : ViewBinding>(
    viewBinder: (A) -> V
) : LifecycleViewBindingProperty<A, V>(viewBinder) {
    override fun getLifecycleOwner(thisRef: A): LifecycleOwner? {
        if (thisRef is LifecycleOwner) {
            return thisRef
        }
        return null
    }
}

abstract class LifecycleViewBindingProperty<in R : Any, out V : ViewBinding>(private val viewBinder: (R) -> V) :
    ViewBindingProperty<R, V> {
    private var viewBinding: V? = null

    protected abstract fun getLifecycleOwner(thisRef: R): LifecycleOwner?

    override fun getValue(thisRef: R, property: KProperty<*>): V {
        viewBinding?.let { return it }
        getLifecycleOwner(thisRef)?.lifecycle?.apply {
            if (currentState == Lifecycle.State.DESTROYED) {

            } else {
                addObserver(ClearOnDestroyLifecycleObserver(this@LifecycleViewBindingProperty))
            }
        }
        viewBinder(thisRef).apply {
            viewBinding = this
            return this
        }
    }

    @MainThread
    override fun clear() {
        viewBinding = null
    }

    private class ClearOnDestroyLifecycleObserver(val property: LifecycleViewBindingProperty<*, *>) :
        LifecycleObserver {
        private companion object {
            private val mainHandler = Handler(Looper.getMainLooper())
        }

        @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
        fun onDestroy() {
            mainHandler.post { property.clear() }
        }
    }
}

fun getBindView(component: Any): View {
    return when (component) {
        is Activity -> {
            val contentView = component.findViewById<ViewGroup>(android.R.id.content)
            contentView.getChildAt(0)
        }
        is Fragment -> {
            component.view ?: error("Fragment getView is return null")
        }
        else -> error("unsupport type $component")
    }
}
```

在 Activity 中使用

```kotlin
class TestActivity: AppCompatActivity(R.layout.activity_test) {

    val binding by viewBindings(ActivityTestBinding::bind)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContentView(binding.root)
        binding.tvDisplay.text = "Hello World."
    }
}
```



[Android JetPack LifeCycle源码分析 - 掘金 (juejin.cn)](https://juejin.cn/post/7073390714129743903)

[不做跟风党，LiveData，StateFlow，SharedFlow 使用场景对比 - 掘金 (juejin.cn)](https://juejin.cn/post/7007602776502960165)

[由浅入深，详解 LiveData 的那些事 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650271079&idx=1&sn=ca7ccaa157fd1732072f1f7d4c5981a7&chksm=88631c08bf14951ebad1368c02d88f265617d73ea096991585056d457ef056f59ea28e1f5990&mpshare=1&scene=1&srcid=1213Zf1idsLg2NjsPzqk6ugf&sharer_sharetime=1670890023071&sharer_shareid=6abfe554f60ba3cca7d5d67848a573e9#rd)

[由浅入深，详解 ViewModel 的那些事 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650272024&idx=1&sn=eb92295862d3fc8ee94a5bbcee916a7a&chksm=88630077bf14896150df4fd5d44cdace40eba361fa9a0cc45db0fc238a1c313e78cb27f52c11&mpshare=1&scene=1&srcid=0128nkUwGcPnjx6yUlBlN1xZ&sharer_sharetime=1674867867004&sharer_shareid=6abfe554f60ba3cca7d5d67848a573e9#rd)

