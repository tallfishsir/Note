# Jetpack ViewBinding

ViewBinding 用于轻量化地实现视图与变量的绑定，它的实现原理是 Android Gradle 插件为每一个 XML 布局文件生成一个 View 绑定类，绑定类中包含了布局文件每个定义了 android:id 属性的 View 引用。

Gradle 生成的绑定类提供了三个静态方法用于生成 ViewBinding 实例：

```kotlin
fun <T> inflate(inflater : LayoutInflater) : T {
    return inflate(inflater, null, false);
}

fun <T> inflate(inflater : LayoutInflater, parent : ViewGroup?, attachToParent : Boolean) : T{
    View root = inflater.inflate(R.layout.xxx, parent, false);
    if (attachToParent) {
      parent.addView(root);
    }
    return bind(root);
}

fun <T> bind(rootView : View) : T
```

可以看到：前两个方法都是传入 LayoutInflater ，然后生成 rootView 后，再传给 bind 方法最后生成 ViewBinding 实例的。

## 基本使用

### 在 Activity 中使用

```kotlin
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

### 在 Fragment 中使用

```kotlin
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

总结来说，就是如果已经有了 XML 布局对应的 View 的根布局，就直接使用 bind 方法获取 ViewBinding 实例，如果没有XML 布局对应的 View 的根布局，就使用 inflate 先生成根布局，再获取 ViewBinding 实例。

## 源码分析

生成的 ViewBinding 绑定类源码如下，以 布局为 R.layout.activity_test，生成的 ActivityTestBinding 为例：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/tv_Display"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

```kotlin
public final class ActivityTestBinding implements ViewBinding {
    private final ConstraintLayout rootView;
    public final TextView tvDisplay;
  
    private ActivityTestBinding (ConstraintLayout paramConstraintLayout1, TextView paramTextView)
        this.rootView = paramConstraintLayout1;
        this.tvDisplay = paramTextView;
    }
  
	// 传入 inflate 生成的 rootView，然后通过 findViewById 绑定视图
    public static ActivityTestBinding bind(View paramView) {
        // 2131165363 是 android:id="@+id/tv_Display" 的值
        TextView localTextView = (TextView)paramView.findViewById(2131165363);
        if (localTextView != null) {
            // 最后调用 ViewBinding 的构造函数，生成实例
            return new ActivityMainBinding((ConstraintLayout)paramView, localTextView);
        }else {
          paramView = "tvDisplay";
        }
        throw new NullPointerException("Missing required view with ID: ".concat(paramView));
    }
  
    public static ActivityMainBinding inflate(LayoutInflater paramLayoutInflater) {
        return inflate(paramLayoutInflater, null, false);
    }
  
    public static ActivityMainBinding inflate(LayoutInflater paramLayoutInflater, ViewGroup paramViewGroup, boolean paramBoolean) {
        // 2131361821 是 R.layout.activity_test 编译后的值
        paramLayoutInflater = paramLayoutInflater.inflate(2131361821, paramViewGroup, false);
        if (paramBoolean) {
            paramViewGroup.addView(paramLayoutInflater);
        }
        return bind(paramLayoutInflater);
    }
  
    public ConstraintLayout getRoot() {
        return this.rootView;
    }
}
```

## 委托封装

在 Activity/Fragment 中使用 View Binding 需要编写样板代码，特别是 Fragment 中 ViewBinding 的实例是可空可变的，使用起来不方便

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

### 在 Activity 中使用

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

