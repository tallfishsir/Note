# Jetpack ViewModel

ViewModel 是一个为界面准备数据的模型，为 UI 层提供数据。

ViewModel 的生命周期长于 Activity，在因为屏幕旋转而重新创建 Activity 后 ViewModel 对象依然会被保留，只有 Activity 真正被 Finish 的时候，ViewModel 才会被清除。

![ViewModel和Activity生命周期](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb123d4f25504ef292beedd51c1a503e~tplv-k3u1fbpfcp-watermark.image)

同时因为 ViewModel 的生命周期比 UI 层引用长，所以 ViewModel 不会持有 UI 层引用，避免了内存泄漏。ViewModel 是通过 LiveData 来完成对 UI 层的回调通知。

## 使用方法

### 基本使用

1. 创建 MyViewModel 类继承 ViewModel 
2. 在 MyViewModel 中创建 LiveData 实例，并编写获取数据的逻辑
3. 将获取到的数据利用 LiveData setValue 传递出来
4. 在 Activity/Fragment 中使用 ViewModelProvider 获取 MyViewModel 实例
5. 观察 MyViewModel 中的 LiveData 数据，并进行对应的 UI 更新

```kotlin
//
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

### Fragment 间数据共享

Activity 中的多个 Fragment 需要相互通信，可以让他们使用其 Activity 范围共享 ViewModel 处理。多个 Fragment 通过 ViewModelProvider 获取 ViewModel 实例时，ViewModelStoreOwner 传参是他们的宿主 Activity。具有以下优势：

1. Activity 不需要执行任何操作，不需要对通信有任何了解
2. 除了共享的数据约定外，Fragment 之间不需要相互了解，各自执行生命周期。不会相互影响

## 源码分析

### ViewModel 的存储和获取

ViewModel 实例是通过 ViewModelProvider 类的 get 方法获取的，ViewModelProvider 的构造函数：

```java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
    this(owner.getViewModelStore(), factory);
}

public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    mViewModelStore = store;
}
```

ViewModelProvider 的构造函数出现了三个类型的参数：ViewModelStoreOwner 、ViewModelStore 和 Factory

- ViewModelStoreOwner： ViewModel 存储容器拥有者
- ViewModelStore：ViewModel 存储容器
- Factory：ViewModel 实例的创建方式

ViewModelStore 是实际上存放 ViewModel 的数据结构：

```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     * 调用ViewModel的clear()方法，然后清除ViewModel
     * 如果ViewModelStore的拥有者（Activity/Fragment）销毁后不会重建，那么就需要调用此方法
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

ViewModelStore 实际上是让 ViewModel 作为 Key 存储在 HashMap 中。

看下 ViewModel 实例是如何通过 get 方法创建的：

```java
// ViewModelProvider
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    // 拿到Key，也即是 ViewModelStore 中的Map的用于存 ViewModel 的 Key
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    //从ViewModelStore获取ViewModel实例
    ViewModel viewModel = mViewModelStore.get(key);
    if (modelClass.isInstance(viewModel)) {
        //如果从ViewModelStore获取到，直接返回
        return (T) viewModel;
    } else {
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
    } else {
        //没有获取到，就使用Factory创建
        viewModel = (mFactory).create(modelClass);
    }
    //存入ViewModelStore 然后返回
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```

先尝试从 ViewModelStore 获取 ViewModel 实例，key是"androidx.lifecycle.ViewModelProvider.DefaultKey:xxx.SharedViewModel"，如果没有获取到，就使用 Factory 创建，然后存入 ViewModelStore。

### ViewModelStore 的存储和获取

ViewModelProvider 构造函数中 ViewModelStoreOwner 是一个接口，实现类有 Activity/Fragment，也就是说 Activity/Fragment 都是 ViewModel 存储器的拥有者。

```java
public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}

//ComponentActivity.java
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        //activity还没关联Application，即不能在onCreate之前去获取viewModel
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        //如果存储器是空，就先尝试 从lastNonConfigurationInstance从获取
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}

public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();
    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        // No one called getViewModelStore(), so see if there was an existing
        // ViewModelStore from our last NonConfigurationInstance
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
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

先从 NonConfigurationIntance 中获取 ViewModelStore 实例，如果 NonConfigurationInstance 也不存在  ViewModelStore 实例，就创建一个 ViewModelStore 实例。

onRetainNonConfigurationInstance 方法很重要，在 Activity 因为配置改变而正要销毁、新 Activity 立即创建时调用。NonConfigurationInstance 中的 ViewModelStore  实例就是在 Activity 的配置改变时，回调 onRetainNonConfigurationInstance保存起来。

```java
//ComponentActivity.java
static final class NonConfigurationInstances {
    Object custom;
    ViewModelStore viewModelStore;
}
```

NonConfigurationInstances 是 ComponentActivity 的静态内部类，所以屏幕旋转等的配置改变 不会影响到这个实例。

mLastNonConfigurationInstances 是在 Activity 的 attach 方法中赋值。 attach 方法是为 Activity 关联上下文环境，是在Activity 启动的核心流程——ActivityThread的performLaunchActivity方法中调用，这里的lastNonConfigurationInstances 是存在 ActivityClientRecord 中的一个组件信息。

ActivityThread 中的 ActivityClientRecord 是不受 activity 重建的影响，那么ActivityClientRecord中lastNonConfigurationInstances也不受影响，那么其中的Object activity也不受影响，那么 ComponentActivity 中的NonConfigurationInstances 的 ViewModelStore  不受影响，那么 ViewModel  也就不受影响了。

## 总结

Activity 的 onSaveInstaneState 调用时发生在系统”未经许可“销毁 Activity 时：

- 用户按下 Home 键时或长按 Home 切换其他应用
- 用户按下电源键，关闭屏幕
- 从 Activity 启动另一个 Activity
- 屏幕方向等配置更改时

使用 ViewModel 恢复数据则只有在因配置更改界面销毁重建的情况。