# Jetpack Lifecycle

Lifecycle 用于辅助管理页面生命周期的工具，可以监控 Activity 和 Fragment 各个生命周期回调，获取页面当前生命周期状态。

## 使用方法

### 基本使用

生命周期拥有者通过使用 `getLifecycle()`获取 Lifecycle 实例，然后使用`addObserver`添加观察者

观察者实现 LIfeObserver 接口，在自定义方法上使用 OnLIfecycleEvent 注解关注对应的生命周期，生命周期触发时就会执行对应的方法。

观察者自定义的方法可以接收一个类型是 LIfecycleOwner。

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

    private val TAG = "LifecycleObserver"

    @OnLifecycleEvent(value = Lifecycle.Event.ON_RESUME)
    fun connect(lifecycleOwner: LifecycleOwner) {
        if (lifecycleOwner.lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
            Log.d(TAG,"connect")
        }
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_PAUSE)
    fun disconect() {
        Log.d(TAG,"disconect")
    }
    
    ...
}
```

CustomLifecycleObserver 实现了 LifecycleObserver 接口，LIfecycleObserver 用于标记一个类是生命周期观察者，然后在自定义方法加上了 @LifecycleEvent 注解，使对应方法在对应生命周期时触发执行。

### MVP 架构中使用

在 MVP 架构中，可以将 Presenter 作为 LifecycleObserver，

```kotlin
class LifecycleTestActivity  :  IView, AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_kotlin)

        lifecycle.addObserver(CustomLifecycleObserver())
    }
    ...
}

class CustomPresenter(val mView : IView) : IPresenter,LifecycleObserver {
    private val TAG = "CustomPresenter"

    @OnLifecycleEvent(value = Lifecycle.Event.ON_RESUME)
    fun connect(lifecycleOwner: LifecycleOwner) {
        if (lifecycleOwner.lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
            Log.d(TAG,"connect")
        }
    }

    @OnLifecycleEvent(value = Lifecycle.Event.ON_PAUSE)
    fun disconect() {
        Log.d(TAG,"disconect")
    }
}
```

Presenter 实现 LifeObserver 接口，并注解自定义方法，然后在 Activity 中作为观察者添加到 Lifecycle中。当 Activity 生命周期发生变化时，Presenter就可以感知执行方法了，不需要在 Activity 中各个生命周期中调用 Presenter 方法。

所有的方法调用操作都是由组件本身管理，如果需要在其他 Activity/Fragment 中使用这个 Presenter，只需要添加 Presenter 为观察者即可

### 非 androidx 包中使用

以上在 Activity 中调用 `getLifecycle()`就能获取 Lifecycle 实例，是因为 androidx 中的 Activity 和 Fragment 都已实现 LifecycleOwner 接口，如果有一个自定义类希望成为 LifecycleOwner，可以使用 LifecycleRegistry 类，它是 LifecycleOwner 接口的实现类。

```kotlin
class CustomLifecycleActivity  :  LIfecycleOwner, Activity() {
	
	private val mLifecycleRegistry = LifecycleRegistry(this)
	
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_kotlin)

        getLifecycle().addObserver(CustomLifecycleObserver())
        
		mLifecycleRegistry.setCurrentState(Lifecycle.State.CREATED)
    }
    
    override fun onStart(){
    	mLifecycleRegistry.setCurrentState(Lifecycle.State.STARTED)
    }
    
   override fun onStop(){
    	mLifecycleRegistry.setCurrentState(Lifecycle.State.CREATED)
    }
    
   override fun getLifecycle(): Lifecycle{
    	return mLifecycleRegistry
    }
    ...
}
```

Activity 实现 LifecycleOwner 接口，`getLifecycle()`返回 LifecycleRegistry 实例，LifecycleRegistry 实例在 Activity 的 onCreate 中创建，并在各个生命周期内调用`setCurrentState`方法完成生命周期事件的传递。

在 Activity/Fragment 各个生命周期中 setCurrentState 方法具体传值见：源码解析-Lifecycle 类

### Application 中使用

之前对 App 进入前后台的判断是通过 registerActivityLifecycCallbacks 来完成的，然后在 callback 中利用一个全局变量做计数，在`onActivityStarted`中计数加 1 ，在`onActivityStopped`中计算减 1,从而判断前后台切换。

使用 ProcessLifecycleOwner 可以直接获取应用前后台切换状态，使用方式与 Activity 中类似，只不过要使用 `ProcessLifecycleOwner.get()`来获取 ProcessLifecycleOwner

```kotlin
class CustomApplication : Application {
	override fun onCreate() {
		 super.onCreate();
		 
		 // 册App生命周期观察者
        ProcessLifecycleOwner.get().getLifecycle().addObserver(new ApplicationLifecycleObserver());
	}
}

class ApplicationLifecycleObserver : LIfecycleOwner {
    
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
	private void onAppForeground() {
		Log.w(TAG, "ApplicationObserver: app moved to foreground");
	}

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    private void onAppBackground() {
        Log.w(TAG, "ApplicationObserver: app moved to background");
    }
}
```

## 源码分析

Lifecycle 的源码大概逻辑是：LifecycleOwner 在生命周期状态改变时，遍历观察者，获取每个观察者的方法上的注解，如果注解是 @LifecycleEvent 且 value 是和生命周期状态一致，那么就执行这个方法。

### Lifecycle 类

```java
public abstract class Lifecycle {
    //添加观察者
    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);
    //移除观察者
    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    //获取当前状态
    public abstract State getCurrentState();

//生命周期事件，对应Activity生命周期方法
    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY  //可以响应任意一个事件
    }
    
    //生命周期状态. （Event是进入这种状态的事件）
    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        //判断至少是某一状态
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
```

Lifecycle 使用两种枚举跟踪组件的生命周期状态：

- Event：生命周期事件，这些事件对应 Activity/Fragment 生命周期方法
- State：生命周期状态，通过 Event 事件，进入某种状态

两者之间的关系可以用下图总结：

![image-20210831222230669](https://pic.imgdb.cn/item/612e3b2544eaada7390e1a01.png)

### LifecycleRegistry 类

在非 androidx 包中使用 Lifecycle 提到了 LifecycleRegistry，它是 Lifecycle 的子类，实现了 addObserver/removeObserver 方法，完成了 Lifecycle.Event 和 Lifecycle.State 之间的转化和同步状态到所有观察者的功能。

```java
//LifecycleRegistry.java

//系统自定义的保存Observer的map，可在遍历中增删
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap = new FastSafeIterableMap<>();

//当前生命周期状态
private State mState;

//有新的 Event 发生，但没有处理
private boolean mNewEventOccurred = false;

//是否正在同步状态给各个观察者
private boolean mHandlingEvent = false;

//正在执行 addObserver 的观察者的数量
private int mAddingObserverCounter = 0;

public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    //带状态的观察者，这个状态的作用：新的事件触发后 遍历通知所有观察者时，判断是否已经通知这个观察者了
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState)
    //observer作为key，ObserverWithState作为value，存到mObserverMap
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
    
    //如果已经添加过，不处理
    if (previous != null) {
        return;
    }
    
    // mLifecycleOwner 是 WeakReference
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    // lifecycleOwner退出了，不处理
    if (lifecycleOwner == null) {
        return;
    }
    
    // 正在添加观察者或者正在同步状态
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    //计算目标状态
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    //通过while循环，把新的观察者的状态 连续地 同步到最新状态mState
    //虽然可能添加的晚，但把之前的事件一个个分发给你(upEvent方法)，即粘性
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState
        popParentState();
        targetState = calculateTargetState(observer);
    }
    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}

public void removeObserver(@NonNull LifecycleObserver observer) {
    mObserverMap.remove(observer);
}
                                                               
//设置当前生命周期状态
public void setCurrentState(@NonNull State state) {
    moveToState(state);
}

private void moveToState(State next) {
    // 如果当前状态已经是需要设置的状态，不执行后续逻辑
    if (mState == next) {
        return;
    }
    // 赋值新状态
    mState = next;
     // 如果正在同步观察者当前状态 或者 正在添加观察者，修改 mNewEventOccurred 标志位，不执行后续逻辑
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        return;
    }
    mHandlingEvent = true;
    //把生命周期状态同步给所有观察者
    sync();
    mHandlingEvent = false;
}

private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    //isSynced()意思是 所有观察者都同步完了
    while (!isSynced()) {
        mNewEventOccurred = false;
        // 设置的状态在保存的状态之前
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        // 设置的状态在保存的状态之后
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
                                                               
private boolean isSynced() {
    if (mObserverMap.size() == 0) {
        return true;
    }
    State eldestObserverState = mObserverMap.eldest().getValue().mState;
    State newestObserverState = mObserverMap.newest().getValue().mState;
    //最老的和最新的观察者的状态一致，都是ower的当前状态，说明已经同步完了
    return eldestObserverState == newestObserverState && mState == newestObserverState;
}
                                                               
private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    //反向遍历，从大到小
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            // 根据状态，获取反向的（DESTROYED< [ON_DESTROY] < CREATED < [ON_STOP] < STARTED < [ON_PAUSE] < RESUMED）此状态将要执行的 Event
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
                                                               
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    //正向遍历，从小到大
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
             // 根据状态，获取反向的（DESTROYED > [ON_CREATE] > CREATED > [ON_START] > STARTED > [ON_RESUME] > RESUMED）此状态将要执行的 Event
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}

public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}
 ...
```

### LifecycleOwner 类

在 androidx 包中的 ComponentActivity 实现了 LifecycleOwner 接口，所以可以直接使用 getLifecycle()

```java
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}

//androidx.activity.ComponentActivity，这里忽略了一些其他代码，我们只看Lifecycle相关
public class ComponentActivity extends androidx.core.app.ComponentActivity implements LifecycleOwner{
    ...
   
    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    ...
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mSavedStateRegistryController.performRestore(savedInstanceState);
        ReportFragment.injectIfNeededIn(this); //使用ReportFragment分发生命周期事件
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }
    @CallSuper
    @Override
    protected void onSaveInstanceState(@NonNull Bundle outState) {
        Lifecycle lifecycle = getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).setCurrentState(Lifecycle.State.CREATED);
        }
        super.onSaveInstanceState(outState);
        mSavedStateRegistryController.performSave(outState);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```

ComponentActivity实现了接口LifecycleOwner，并在getLifecycle()返回了LifecycleRegistry实例。然后在onSaveInstanceState()中设置mLifecycleRegistry的状态为State.CREATED，其他生命周期状态设置都是通过 `ReportFragment.injectIfNeededIn(this);` 来实现的

### 生命周期事件分发-ReportFragment

```java
//专门用于分发生命周期事件的Fragment
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

首先injectIfNeededIn()内进行了版本区分：在API 29及以上 直接使用activity的registerActivityLifecycleCallbacks 直接注册了生命周期回调，然后给当前activity添加了ReportFragment，注意这个fragment是没有布局的。

无论LifecycleCallbacks、还是fragment的生命周期方法 最后都走到了 dispatch(Activity activity, Lifecycle.Event event)方法，其内部使用LifecycleRegistry的handleLifecycleEvent方法处理事件。

而ReportFragment的作用就是获取生命周期而已，因为fragment生命周期是依附Activity的。好处就是把这部分逻辑抽离出来，实现activity的无侵入。如果你对图片加载库Glide比较熟，就会知道它也是使用透明Fragment获取生命周期的。

## 总结

Lifecycle 的子类 LifecycleRegistry 是实际实现监控逻辑的文件，在 addObserver 方法创建了 ObserverWithState 类，并添加到 FastSafeIterableMap 类型的数据容器中缓存，供组件的生命周期发生变化时，寻找对应的方法并调用。

当组件的生命周期发生变化时，通过调用 LifecycleRegistry 的 setCurrentState 或 handleLifecycleEvent，开始同步观察者最新生命周期状态 sync 方法。

ObserverWithState 中包含了 LifecycleEventObserver 接口类型的实例，当组件的生命周期发生变化时，sync  方法会调用 ObserverWithState  的 dispatchEvent 方法，此方法会调用 LifecycleEventObserver 的 onStateChanged 方法。

在 onStateChanged  中会通过解析注解寻找对应的方法，并调用

文章参考学习：

[Lifecycle 完全掌握]: https://juejin.cn/post/6893870636733890574#heading-12

