# Jetpack LiveData

LiveData 是一个基于 Lifecycle 的可观察的数据存储容器，它具有感知生命周期的能力，确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。

使用 LiveData 具有以下优势：

- 确保界面符合数据状态，当生命周期状态发生变化时，LiveData 通知 Observer，observer 可以在生命周期状态更改时刷新界面，而不是每次数据变化时才刷新
- 不会发生内存泄漏，observer 会在 LifecycleOwner 状态变为 DESTROYED 后自动 remove
- 不会因 Activity 停止而导致崩溃，如果 LifecycleOwner 生命周期处于非活跃状态，则它不会接收任何 LiveData 事件
- 不需要手动解除观察，LiveData 能感知生命周期状态变化，会自动管理对 LiveData 的从观察
- 数据始终保持在最新状态，数据更新时，如果 LifecycleOwner 为非活跃状态，那么就会在变为活跃时接收最新的数据

## 使用方法

### 基本使用

1. 创建 LiveData 实例，通过泛型指定源数据类型
2. 创建 Observer 实例，实现 onChanged() 方法，用于接收源数据变化
3. LiveData 实例使用 observe() 方法添加 Observer 实例观察者，并传入 LifecycleOwner
4. LiveData 实例使用 setValue()/postValue() 更新源数据

```java
public class LiveDataTestActivity extends AppCompatActivity{

   private MutableLiveData<String> mLiveData;
   
   @Override
   protected void onCreate(Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);
       setContentView(R.layout.activity_lifecycle_test);
       
       //liveData基本使用
       mLiveData = new MutableLiveData<>();
       mLiveData.observe(this, new Observer<String>() {
           @Override
           public void onChanged(String s) {
               Log.i(TAG, "onChanged: "+s);
           }
       });
       Log.i(TAG, "onCreate: ");
       mLiveData.setValue("onCreate");//activity是非活跃状态，不会回调onChanged。变为活跃时，value被onStart中的value覆盖
   }
   @Override
   protected void onStart() {
       super.onStart();
       Log.i(TAG, "onStart: ");
       mLiveData.setValue("onStart");//活跃状态，会回调onChanged。并且value会覆盖onCreate、onStop中设置的value
   }
   @Override
   protected void onResume() {
       super.onResume();
       Log.i(TAG, "onResume: ");
       mLiveData.setValue("onResume");//活跃状态，回调onChanged
   }
   @Override
   protected void onPause() {
       super.onPause();
       Log.i(TAG, "onPause: ");
       mLiveData.setValue("onPause");//活跃状态，回调onChanged
   }
   @Override
   protected void onStop() {
       super.onStop();
       Log.i(TAG, "onStop: ");
       mLiveData.setValue("onStop");//非活跃状态，不会回调onChanged。后面变为活跃时，value被onStart中的value覆盖
   }
   @Override
   protected void onDestroy() {
       super.onDestroy();
       Log.i(TAG, "onDestroy: ");
       mLiveData.setValue("onDestroy");//非活跃状态，且此时Observer已被移除，不会回调onChanged
   }
}
```

除了使用observe()方法添加观察者，也可以使用 observeForever(Observer) 方法来注册未关联 LifecycleOwner的观察者。在这种情况下，观察者会被视为始终处于活跃状态。

### 进阶使用

#### onActive()/onInactive()

自定义 LiveData，覆写以上两个回调方法的回调时机：

- onActive()调用时机为：活跃的观察者（LifecycleOwner）数量从 0 变为 1 时
- onInactive()调用时机为：活跃的观察者（LifecycleOwner）数量从 1 变为 0 时

#### 数据修改 Transformations.map

Transformations.map()方法 对liveData1的数据进行的修改 生成了新的liveDataMap，liveDataMap添加观察者，最后liveData1设置数据

```java
        //Integer类型的liveData1
        MutableLiveData<Integer> liveData1 = new MutableLiveData<>();
        //转换成String类型的liveDataMap
        LiveData<String> liveDataMap = Transformations.map(liveData1, new Function<Integer, String>() {
            @Override
            public String apply(Integer input) {
                String s = input + " + Transformations.map";
                Log.i(TAG, "apply: " + s);
                return s;
            }
        });
        liveDataMap.observe(this, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.i(TAG, "onChanged1: "+s);
            }
        });

        liveData1.setValue(100);
```

#### 数据切换 Transformations.switchMap

想要根据某个值 切换观察不同LiveData数据，则可以使用Transformations.switchMap()方法。

```java
        MediatorLiveData<String> mediatorLiveData = new MediatorLiveData<>();

        MutableLiveData<String> liveData5 = new MutableLiveData<>();
        MutableLiveData<String> liveData6 = new MutableLiveData<>();

	//添加 源 LiveData
        mediatorLiveData.addSource(liveData5, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.i(TAG, "onChanged3: " + s);
                mediatorLiveData.setValue(s);
            }
        });
	//添加 源 LiveData
        mediatorLiveData.addSource(liveData6, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.i(TAG, "onChanged4: " + s);
                mediatorLiveData.setValue(s);
            }
        });

	//添加观察
        mediatorLiveData.observe(this, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.i(TAG, "onChanged5: "+s);
                //无论liveData5、liveData6更新，都可以接收到
            }
        });
        
        liveData5.setValue("liveData5");
        //liveData6.setValue("liveData6");

```

#### 观察多个数据 - MediatorLiveData

MediatorLiveData 是 LiveData 的子类，允许合并多个 LiveData 源。只要任何原始的 LiveData 源对象发生更改，就会触发 MediatorLiveData 对象的观察者。

```java
        MediatorLiveData<String> mediatorLiveData = new MediatorLiveData<>();

        MutableLiveData<String> liveData5 = new MutableLiveData<>();
        MutableLiveData<String> liveData6 = new MutableLiveData<>();

	//添加 源 LiveData
        mediatorLiveData.addSource(liveData5, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.i(TAG, "onChanged3: " + s);
                mediatorLiveData.setValue(s);
            }
        });
	//添加 源 LiveData
        mediatorLiveData.addSource(liveData6, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.i(TAG, "onChanged4: " + s);
                mediatorLiveData.setValue(s);
            }
        });

	//添加观察
        mediatorLiveData.observe(this, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.i(TAG, "onChanged5: "+s);
                //无论liveData5、liveData6更新，都可以接收到
            }
        });
        
        liveData5.setValue("liveData5");
        //liveData6.setValue("liveData6");
```

## 源码分析

### 添加观察者

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // LifecycleOwner是DESTROYED状态，直接忽略
        return;
    }
    //使用LifecycleOwner、observer 组装成LifecycleBoundObserver，添加到mObservers中
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
         //!existing.isAttachedTo(owner)说明已经添加到mObservers中的observer指定的owner不是传进来的owner
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    //这里说明已经添加到mObservers中,且owner就是传进来的owner
    if (existing != null) {
        return;
    }
    // LifecycleOwner 添加观察者，当生命周期状态发生变化时，回调 onStateChanged 方法
    owner.getLifecycle().addObserver(wrapper);
}

//和observe()类似，只不过会认为观察者一直时活跃状态
@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    assertMainThread("observeForever");
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```

### 事件回调

```java

// GenericLifecycleObserver 是 LifecycleEventObserver 的子类
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull
    final LifecycleOwner mOwner;
    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }
    @Override
    boolean shouldBeActive() {
        //至少是STARTED状态
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            //LifecycleOwner变成DESTROYED状态，则移除观察者
            removeObserver(mObserver);
            return;
        }
        // 至少是STARTED状态
        // activeStateChanged 是 ObserverWrapper 方法
        activeStateChanged(shouldBeActive());
    }
    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }
    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}

private abstract class ObserverWrapper {
    final Observer<? super T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;
    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }
    abstract boolean shouldBeActive();
    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }
    void detachObserver() {
    }
    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            //活跃状态 未发生变化时，不会处理。
            return;
        }
        mActive = newActive;
        //没有活跃的观察者
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        //mActive为true表示变为活跃
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        if (wasInactive && mActive) {
            //活跃的观察者数量 由0变为1
            onActive();
        }
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            //活跃的观察者数量 由1变为0
            onInactive();
        }
        if (mActive) {
            //观察者变为活跃，就进行数据分发
            dispatchingValue(this);
        }
    }
}

void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        //如果当前正在分发，则分发无效，return
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

### 数据更新

LivaData数据更新可以使用setValue(value)、postValue(value)，区别在于postValue(value)用于 子线程

```java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
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
        //noinspection unchecked
        setValue((T) newValue);
    }
};
```

### Transformations 原理

```java
 @MainThread
 public static <X, Y> LiveData<Y> map(
         @NonNull LiveData<X> source,
         @NonNull final Function<X, Y> mapFunction) {
     final MediatorLiveData<Y> result = new MediatorLiveData<>();
     result.addSource(source, new Observer<X>() {
         @Override
         public void onChanged(@Nullable X x) {
             result.setValue(mapFunction.apply(x));
         }
     });
     return result;
 }
 
 @MainThread
public static <X, Y> LiveData<Y> switchMap(
        @NonNull LiveData<X> source,
        @NonNull final Function<X, LiveData<Y>> switchMapFunction) {
    final MediatorLiveData<Y> result = new MediatorLiveData<>();
    result.addSource(source, new Observer<X>() {
        LiveData<Y> mSource;
        @Override
        public void onChanged(@Nullable X x) {
            LiveData<Y> newLiveData = switchMapFunction.apply(x);
            if (mSource == newLiveData) {
                return;
            }
            if (mSource != null) {
                result.removeSource(mSource);
            }
            mSource = newLiveData;
            if (mSource != null) {
                result.addSource(mSource, new Observer<Y>() {
                    @Override
                    public void onChanged(@Nullable Y y) {
                        result.setValue(y);
                    }
                });
            }
        }
    });
    return result;
}
```

