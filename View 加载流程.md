# View 加载流程

## LayoutInflater

在[Activity 启动过程学习 ](Activity 启动过程学习.md)中提到在 Activity.setContentView 通过 mLayoutInflater 加载传入的布局文件

```java
//PhoneWindow.java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        //初始化DecorView和mContentParent
    	installDecor();
	} else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
	    mContentParent.removeAllViews();
	}
    ...
    //加载资源文件，创建view树装载到mContentParent
    mLayoutInflater.inflate(layoutResID, mContentParent);
}

//mLayoutInflater通过IPC获取
mLayoutInflater = LayoutInflater.from(context);
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

Android 加载布局流程主要分为加载解析 xml 文件和填充 View 树两个部分

```java
//LayoutInflater.java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    //使用pull来解析加载解析xml文件，将xml读取到内存，是一个IO操作
    XmlResourceParser parser = res.getLayout(resource);
    try {
        //填充View树
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}

public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        View result = root;
        try {
            if (TAG_MERGE.equals(name)) {
                 //merge标签只能是布局文件的根节点
              	 if (root == null || !attachToRoot) {
                      throw new InflateException("<merge /> can be used only with a valid "
                + "ViewGroup root and attachToRoot=true");
                 }
				//merge标签，循环遍历子View的加载过程
				rInflate(parser, root, inflaterContext, attrs, false);
			} else {
				//根据解析结果parse，创建View对象
				final View temp = createViewFromTag(root, name, inflaterContext, attrs);
				ViewGroup.LayoutParams params = null;
				if (root != null) {
					params = root.generateLayoutParams(attrs);
					if (!attachToRoot) {
						temp.setLayoutParams(params);
					}
				}
				//循环遍历子View的加载过程
				rInflateChildren(parser, temp, attrs, true);
				if (root != null && attachToRoot) {
					//将创建的View对象，添加作为root的子view
					root.addView(temp, params);
				}
			}
        }
        return result;
    }
}
```

createViewFromTag() 会先尝试通过 Factory 创建 View，如果没有设置额外的 Factory，才会调用框架的 onCreateView 来创建原生 View 或者调用 createView 来创建自定义 View。

Factory 和 Factory2 都是接口，Factory2 继承 Factory，扩展出一个参数该节点的父 View，设置 Factory 和 Factory2 需要在 Activity.onCreate() 中调用 setFactory() 或 setFactory2()， 且 Activity 的 mFactory 和 mFactory2 是互斥的，只能设置一个。

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    ...
    try {
        //通过Factory创建View
        View view = tryCreateView(parent, name, context, attrs);
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    //创建原生View
                    view = onCreateView(context, parent, name, attrs);
                } else {
                    //闯将自定义View
                    view = createView(context, name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }
        return view;
    }
    ...
}

//选择Factory或者Factory2或者mPrivateFactory来创建View
public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    View view;
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }
    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }
    return view;
}

public interface Factory {
    public View onCreateView(String name, Context context, AttributeSet attrs);
}

public interface Factory2 extends Factory {
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
}
```

onCreateView() 创建原生 View 的过程，只是添加了 View 的前缀包名，最终依然调用到了 createView()。

createView 会通过传入的 name 参数，利用 ClassLoader 创建 Class 对象，然后获取 Class 对象的 mConstructorSignature 形式的构造函数，最终 newInstance() 创建 View 对象。

mConstructorSignature 数组对应的正是 View(Context context, AttributeSet attrs) 构造函数。

```java
public View onCreateView(Context viewContext, View parent, String name, AttributeSet attrs) {
    return onCreateView(parent, name, attrs);
}

public View onCreateView(Context viewContext, View parent, String name) {
    return onCreateView(parent, name);
}

public View onCreateView(Context viewContext, String name) {
    return createView(name, "android.view.", attrs);
}

public final View createView(String name, String prefix, AttributeSet attrs) {
    //从缓存中获取构造器，如果对应的Class文件加载过，会被缓存起来  
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    Class<? extends View> clazz = null;

    try {
    	//第一次加载
        if (constructor == null) {
            //通过前缀+name的方式获取Class对象，并获得构造器
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            //添加到缓存中
            sConstructorMap.put(name, constructor);
        }
        ...
    }
    Object[] args = mConstructorArgs;
    final View view = constructor.newInstance(args);
    return view
 }

static final Class<?>[] mConstructorSignature = new Class[] {Context.class, AttributeSet.class};
```

### View 构造函数

View 的常见构造函数有四个，分别对应不同的使用场景

```java
//一般用于在代码中动态创建View对象
View(Context context) {}

//一般用于LayoutInflter解析xml后，反射创建View对象
View(Context context, AttributeSet attrs) {
	this(context, attrs, 0);
}

//需要主要调用，传入defStyleAttr
View(Context context, AttributeSet attrs, int defStyleAttr) {
	this(context, attrs, defStyleAttr, 0);
}

View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
}
```

#### AttributeSet

当使用自定义属性时，需要先在 attrs.xml 中编写 styleable 和 item 等标签元素

```xml
attrs.xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="test">
        <attr name="text" format="string" />
        <attr name="testAttr" format="integer" />
    </declare-styleable>
</resources>

布局.xml
<com.example.test.MyTextView
        android:layout_width="100dp"
        android:layout_height="@dimen/dp200"
        cus:testAttr="520"
        cus:text="@string/hello_world" />
```

在构造函数中 AttributeSet 参数保存的是该 View 在 xml 中所有的属性。所有的引用都会变成@+数字的字符串。为了简化读取数据的工作，就增加了 TypeArray 类进一步获取引用所指向的数据值。

obtainStyledAttributes() 的第二个参数 R.styleable.test，实际上是编译期间根据 declare-styleable 自动生成的一个数组

```java
View(Context context, AttributeSet attrs) {
    for (int i = 0; i < count; i++) {
        String attrName = attrs.getAttributeName(i);
        String attrVal = attrs.getAttributeValue(i);
        Log.e(TAG, "attrName = " + attrName + " , attrVal = " + attrVal);
	}
//attrName = layout_width , attrVal = 100
//attrName = layout_height , attrVal = @2131165235
//attrName = text , attrVal = 520
//attrName = testAttr , attrVal = @2131361809
    
	TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.test);
	String text = ta.getString(R.styleable.test_testAttr);
	int textAttr = ta.getInteger(R.styleable.test_text, -1);
}

public static final class styleable {
     public static final int test_android_text = 0;
     public static final int test_testAttr = 1;
     public static final int[] test = {
		0x0101014f, 0x7f0100a9
	};
}
```

#### defStyleAttr/defStyleRes

在 xml 文件中为自定义属性赋值有四种方法：

- 直接在布局文件中使用属性

  ```xml
  <com.example.customstyle.CustomTextView
  	android:layout_width="wrap_content"
  	android:layout_height="wrap_content"
  	ad:attr_one="attr one in xml"/>
  ```

- 设置 style 并在 style 中设置属性

  ```xml
  <com.example.customstyle.CustomTextView
  	android:layout_width="wrap_content"
  	android:layout_height="wrap_content"
  	style="@style/ThroughStyle"/>
  	
  style.xml
  <style name="ThroughStyle">
      <item name="attr_one">attr one from style</item>
  </style>
  ```

- 在 Application 或 Activity 的 theme 中设置属性

  ```xml
  <style name="AppTheme" parent="AppBaseTheme">
      <item name="attr_one">attr one from theme</item>
  </style>
  ```

- 在 Application 或 Activity 的 theme 中设置 style，然后在 style 中设置属性

  ```xml
  <style name="AppTheme" parent="AppBaseTheme">
      <item name="CustomizeStyle">@style/CustomizeStyleInTheme</item>
  </style>
  
  <style name="CustomizeStyleInTheme">
      <item name="attr_one">attr one from theme reference</item>
  </style>
  ```

在 View 的构造函数中，通过 Context.obtainStyledAttributes(AttributeSet set, int[] attrs, int defStyleAttr, int defStyleRes) 来获取。

- defStyleAttr：在当前 Application 或 Activity 的 Theme 中设置的 Style 中查找相应的属性值。如果这个参数传入0表示不向 Theme 中搜索默认值
- defStyleRes：在 defStyleAttr为0 或 defStyleAttr不为0但在Theme中没有设置defStyleAttr起作用

上面四个为属性赋值的优先级是：直接在布局文件定义>style 定义>由 defStyleAttr 和 defStyleRes 指定的默认值>直接在Theme中指定的值

### View 异步加载

AsyncLayoutInflater 就是把 View 的一些初始化加载过程，放在子线程，结束后，在将结果回调到主线程。

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    new AsyncLayoutInflater(this).inflate(R.layout.activity_main,null, new AsyncLayoutInflater.OnInflateFinishedListener(){
        @Override
        public void onInflateFinished(View view, int resid, ViewGroup parent) {
            setContentView(view);
            rv = findViewById(R.id.tv_right);
            rv.setLayoutManager(new V7LinearLayoutManager(MainActivity.this));
            rv.setAdapter(new RightRvAdapter(MainActivity.this));
        }
    });
}
```

AsyncLayoutInflater 的构造方法中创建了三个对象：

- BasicInflater：LayoutInflater 的子类，重写了 onCreateView，优先加载"android.widget.”、 "android.webkit."、"android.app." 这三个前缀的 Layout
- Handler：
- InflateThread：创建一个子线程，把 inflate 请求添加到阻塞队列中，按顺序执行，inflate 成功或失败，都将request 发送到主线程去处理

```java
public final class AsyncLayoutInflater {
    private static final String TAG = "AsyncLayoutInflater";

    LayoutInflater mInflater;
    Handler mHandler;
    InflateThread mInflateThread;

    public AsyncLayoutInflater(@NonNull Context context) {
        mInflater = new BasicInflater(context);
        mHandler = new Handler(mHandlerCallback);
        mInflateThread = InflateThread.getInstance();
    }

    @UiThread
    public void inflate(@LayoutRes int resid, @Nullable ViewGroup parent,
            @NonNull OnInflateFinishedListener callback) {
        if (callback == null) {
            throw new NullPointerException("callback argument may not be null!");
        }
        InflateRequest request = mInflateThread.obtainRequest();
        request.inflater = this;
        request.resid = resid;
        request.parent = parent;
        request.callback = callback;
        mInflateThread.enqueue(request);
    }
        ....

```

InflateThread 的作用是创建一个子线程，将 inflate 请求添加到阻塞队列中，并按顺序执行 BasicInflater.inflate 操作，不管 infalte 成功或失败后，都会将 request 消息发送给主线程做处理。

```java
private static class InflateThread extends Thread {
    private static final InflateThread sInstance;
    static {
        sInstance = new InflateThread();
        sInstance.start();
    }

    public static InflateThread getInstance() {
        return sInstance;
    }
    //生产者-消费者模型，阻塞队列
    private ArrayBlockingQueue<InflateRequest> mQueue = new ArrayBlockingQueue<>(10);
    //使用了对象池来缓存InflateThread对象，减少对象重复多次创建，避免内存抖动
    private SynchronizedPool<InflateRequest> mRequestPool = new SynchronizedPool<>(10);

    public void runInner() {
        InflateRequest request;
        try {
            //从队列中取出一条请求，如果没有则阻塞
            request = mQueue.take();
        } catch (InterruptedException ex) {
            // Odd, just continue
            Log.w(TAG, ex);
            return;
        }

        try {
            //inflate操作（通过调用BasicInflater类）
            request.view = request.inflater.mInflater.inflate(
                    request.resid, request.parent, false);
        } catch (RuntimeException ex) {
            // 回退机制：如果inflate失败，回到主线程去inflate
            Log.w(TAG, "Failed to inflate resource in the background! Retrying on the UI"
                    + " thread", ex);
        }
        //inflate成功或失败，都将request发送到主线程去处理
        Message.obtain(request.inflater.mHandler, 0, request)
                .sendToTarget();
    }

    @Override
    public void run() {
        //死循环（实际不会一直执行，内部是会阻塞等待的）
        while (true) {
            runInner();
        }
    }

    //从对象池缓存中取出一个InflateThread对象
    public InflateRequest obtainRequest() {
        InflateRequest obj = mRequestPool.acquire();
        if (obj == null) {
            obj = new InflateRequest();
        }
        return obj;
    }

    //对象池缓存中的对象的数据清空，便于对象复用
    public void releaseRequest(InflateRequest obj) {
        obj.callback = null;
        obj.inflater = null;
        obj.parent = null;
        obj.resid = 0;
        obj.view = null;
        mRequestPool.release(obj);
    }

    //将inflate请求添加到ArrayBlockingQueue（阻塞队列）中
    public void enqueue(InflateRequest request) {
        try {
            mQueue.put(request);
        } catch (InterruptedException e) {
            throw new RuntimeException(
                    "Failed to enqueue async inflate request", e);
        }
    }
}
```

mHandlerCallback 的作用是当子线程中 inflate 失败后，会继续再主线程中进行 inflate 操作，最终通过OnInflateFinishedListener 接口将 view 回调到主线程。

```java
private Callback mHandlerCallback = new Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        InflateRequest request = (InflateRequest) msg.obj;
        if (request.view == null) {
            //view == null说明inflate失败
            //继续再主线程中进行inflate操作
            request.view = mInflater.inflate(
                    request.resid, request.parent, false);
        }
        //回调到主线程
        request.callback.onInflateFinished(
                request.view, request.resid, request.parent);
        mInflateThread.releaseRequest(request);
        return true;
    }
};
```

使用AsyncLayoutInflate主要有如下几个局限性：

1. 所有构建的 View 中不能直接使用 Handler 或者是调用 Looper.myLooper()，因为异步线程默认没有调用 Looper.prepare ()
2. 异步转换出来的 View 并没有被加到父 View 中，需要我们自己手动添加；
3. AsyncLayoutInflater 不支持设置 LayoutInflater.Factory 或者 LayoutInflater.Factory2
4. 同时缓存队列默认 10 的大小限制如果超过了10个则会导致主线程的等待
5. 使用单线程来做全部的 inflate 工作，如果一个界面中 layout 很多不一定能满足需求

### ViewStub/include-merge

#### include-merge

include-merge 标签用于防止在引用布局文件时产生多余的布局嵌套。在 LayoutInflater.inflate 中可以看到，面对 merge 标签的 xml 数据，会调用 rInflate 来解析子 View。

```xml
activity_layout.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="#000000">

    <include layout="@layout/titlebar_layout"></include>

    <TextView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:text="这是内容区域"
        android:gravity="center"
        android:textSize="25sp"
        android:textColor="#ffffff"/>
</LinearLayout>

titlebar_layout.xml
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    <Button
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="top"
        android:text="顶部Button" />

    <Button
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom"
        android:text="底部Button" />
</merge>
```

使用 include 标签需要注意：

- include 标签的 layout_* 属性会替换掉被 include 视图的根节点的对应属性
- include 标签的 id 属性会替换调被 include 视图的根节点 id
- 一个布局文件中支持 include 多个视图，这样

使用 merge 标签需要注意：

- merge 必须是布局文件的根节点

- merge 标签不是 View，当使用 Layout Inflater.inflate 渲染时，第二个参数必须指定一个父容器，且第三个参数必须是 true

- merge 标签不是 View，在 xml 中对其设置的所有属性都是无效的

- 自定义View 继承 ViewGroup，自定义 View 的布局文件根节点设置成 merge，可以减少节点

  ```xml
  public class MergeLayout extends LinearLayout {
      public MergeLayout(Context context) {
          super(context);
          LayoutInflater.from(context).inflate(R.layout.merge_activity, this, true);
      }
  }
  
  <?xml version="1.0" encoding="utf-8"?>
  <merge xmlns:android="http://schemas.android.com/apk/res/android" >
      <TextView
          android:layout_width="match_parent"
          android:layout_height="0dp"
          android:layout_weight="1.0"
          android:background="#000000"
          android:gravity="center"
          android:text="第一个TextView"
          android:textColor="#ffffff" />
  
      <TextView
          android:layout_width="match_parent"
          android:layout_height="0dp"
          android:layout_weight="1.0"
          android:background="#ffffff"
          android:gravity="center"
          android:text="第一个TextView"
          android:textColor="#000000" />
  </merge>
  ```

#### ViewStub

ViewStub 是一个没有大小，不占布局位置且不可见的 View，可以用来懒加载布局。当 ViewStub 可见时或被 inflate() 时，布局会被加载，在加载完成后，ViewStub 会被新的布局替换。

```java
<ViewStub
    android:id="@+id/view_stub_id"
    android:layout="@layout/view_stub"
    android:inflatedId="@+id/view_stub_id"
    android:layout_width="200dp"
    android:layout_height="50dp" />

public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context);
    final TypedArray a = context.obtainStyledAttributes(attrs,
            R.styleable.ViewStub, defStyleAttr, defStyleRes);
    saveAttributeDataForStyleable(context, R.styleable.ViewStub, attrs, a, defStyleAttr,
            defStyleRes);
    //懒加载的布局id
    mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
    // 要被加载的布局
    mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
    // ViewStub 的 Id
    mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
    a.recycle();
    // 初始状态为 GONE
    setVisibility(GONE);
    // 设置为不会绘制
    setWillNotDraw(true);
}

public void setVisibility(int visibility) {
    // mInflatedViewRef 是对布局的弱引用
    if (mInflatedViewRef != null) {
        // 如果不为 null,就拿到懒加载的 View
        View view = mInflatedViewRef.get();
        if (view != null) {
            // 然后就直接对 View 进行 setVisibility 操作
            view.setVisibility(visibility);
        } else {
            // 如果为 null，就抛出异常
            throw new IllegalStateException("setVisibility called on un-referenced view");
        }
    } else {
        super.setVisibility(visibility);
        // 之前说过，setVisibility(int) 也可以进行加载布局
        if (visibility == VISIBLE || visibility == INVISIBLE) {
            // 因为在这里调用了 inflate()
            inflate();
        }
    }
}
```

ViewStub 只能被 inflate 一次，inflate 之后 ViewStub 对象就被置空。即某个被ViewStub指定的布局被Inflate后，就不能够再通过ViewStub来控制它了。

```java
public View inflate() {
    // 获取父视图
    final ViewParent viewParent = getParent();

    if (viewParent != null && viewParent instanceof ViewGroup) {
        // 如果没有指定布局，就会抛出异常
        if (mLayoutResource != 0) {
            // viewParent 需为 ViewGroup
            final ViewGroup parent = (ViewGroup) viewParent;
            final LayoutInflater factory;
            if (mInflater != null) {
                factory = mInflater;
            } else {
                // 如果没有指定 LayoutInflater
                factory = LayoutInflater.from(mContext);
            }
            // 获取布局
            final View view = factory.inflate(mLayoutResource, parent, false);
            // 为 view 设置 Id
            if (mInflatedId != NO_ID) {
                view.setId(mInflatedId);
            }
            // 计算出 ViewStub 在 parent 中的位置
            final int index = parent.indexOfChild(this);
            // 把 ViewStub 从 parent 中移除
            parent.removeViewInLayout(this);

            // 接下来就是把 view 加到 parent 的 index 位置中
            final ViewGroup.LayoutParams layoutParams = getLayoutParams();
            if (layoutParams != null) {
                // 如果 ViewStub 的 layoutParams 不为空
                // 就设置给 view
                parent.addView(view, index, layoutParams);
            } else {
                parent.addView(view, index);
            }
            // mInflatedViewRef 就是在这里对 view 进行了弱引用
            mInflatedViewRef = new WeakReference<View>(view);
            if (mInflateListener != null) {
                mInflateListener.onInflate(this, view);
            }
            return view;
        } else {
            throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
        }
    } else {
        throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
    }
}
```

## addView/removeView

### addView

在 Layout Inflater.inflate() 中，当创建 View 后会调用 View Group.addView 添加子 View，它有三个重载方法。

```java
ViewGroup.java
public void addView(View child) {
    //默认<0，插入数组尾部
    addView(child, -1);
}

public void addView(View child, int index) {
    if (child == null) {
        throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
        //布局参数是空，生成默认值，最终的布局参数一定要存在
        params = generateDefaultLayoutParams();
    }
    addView(child, index, params);
}

protected LayoutParams generateDefaultLayoutParams() {
    return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}

public void addView(View child, int index, LayoutParams params) {
    requestLayout();
    invalidate(true);
    addViewInner(child, index, params, false);
}

private void addViewInner(View child, int index, LayoutParams params,
        boolean preventRequestLayout) {
    //child的mParent存在，抛出异常，一个视图不能加到两个父视图节点
    if (child.getParent() != null) {
        throw new IllegalStateException("The specified child already has a parent. " +
                "You must call removeView() on the child's parent first.");
    }
    //检查生成LayoutParams
    if (!checkLayoutParams(params)) {
        params = generateLayoutParams(params);
    }
    //设置LayoutParams
    if (preventRequestLayout) {
        child.mLayoutParams = params;
    } else {
        child.setLayoutParams(params);
    }
    //index小于0时，插入点在mChildrenCount，即子View数组的尾部
    if (index < 0) {
        index = mChildrenCount;
    }
    //插入数组
    addInArray(child, index);
    //为child设置mParent
    if (preventRequestLayout) {
        child.assignParent(this);
    } else {
        child.mParent = this;
    }
    //子view调用dispatchAttachedToWindow，初始化AttatchInfo
    child.dispatchAttachedToWindow(mAttachInfo, (mViewFlags&VISIBILITY_MASK));
}
```

### removeView

removeView() 和 addView() 是一对，前者是删除视图，后者是增加视图。

```JAVA
public void removeViewAt(int index) {
    removeViewInternal(index, getChildAt(index));
    requestLayout();
    invalidate(true);
}

private void removeViewInternal(int index, View view) {
    if (view.getAnimation() != null ||
                (mTransitioningViews != null && mTransitioningViews.contains(view))) {
        addDisappearingView(view);
    } else if (view.mAttachInfo != null) {
        //回调View onDetachedFromWindow 方法
        view.dispatchDetachedFromWindow();
    }
    ...
    //从数组中删除
    removeFromArray(index);      
    dispatchViewRemoved(view);    
    ...
}

```

### 相似方法

|    add相似方法     |                           方法内容                           |    remove相似方法    |                           方法内容                           |
| :----------------: | :----------------------------------------------------------: | :------------------: | :----------------------------------------------------------: |
|      addView       | requestLayout()<br/>invalidate()<br/>addInArray()<br/>onAttachedToWindow() |      removeView      | onDetachedFromWindow()<br/>removeFromArray()<br/>requestLayout()<br/>invalidate() |
|  addViewInLayout   |            addInArray()<br/>onAttachedToWindow()             |  removeViewInLayou   |         onDetachedFromWindow()<br/>removeFromArray()         |
| attachViewToParent |                         addInArray()                         | detachViewFromParent |                      removeFromArray()                       |
|                    |                                                              |                      |                                                              |

addView/removeView：触发视图重绘制，操作容器内的子视图数组，触发子视图attach和detached窗体回调。
addViewInLayout/removeViewInLayou：操作容器内的子视图数组，触发子视图attach和detached窗体回调。
attachViewToParent/detachViewFromParent：操作容器内的子视图数组。

## View 视图重绘

View 视图重绘分为两个方法：requestLayot() 和 invalidate()，前者最终会触发 Measure、Layout 流程，后者会触发 Draw 流程。

### requestLayout

View.requestLayout 会向上遍历父布局，给每个布局设置 PFLAG_FORCE_LAYOU T和 PFLAG_INVALIDATED 标记，直到调用到 ViewRootImpl.requestLayout()。

ViewRootImpl.requestLayout() 中会检查当前线程是否是主线程，然后调用 scheduleTraversals() 开启刷新流程。

```java
//View.java
public void requestLayout() {
    //清空测量缓存
    if (mMeasureCache != null) mMeasureCache.clear();
    ...
    //PFLAG_FORCE_LAYOUT，该标记会在measure阶段用于判断
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    //添加重绘标记
    mPrivateFlags |= PFLAG_INVALIDATED;
    //如果布局没有结束就不会布局 
    if (mParent != null && !mParent.isLayoutRequested()) {
        //如果上次的layout 请求已经完成
        //父布局继续调用requestLayout
        mParent.requestLayout();
    }
    ...
}

//PFLAG_FORCE_LAYOUT会在layout阶段结束后移除
public boolean isLayoutRequested() {
    return (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
}

//ViewRootImpl.java
public void requestLayout() {
    //是否正在进行layout过程
    if (!mHandlingLayoutInLayoutRequest) {
        //检查线程是否一致
        checkThread();
        //标记有一次layout的请求
        mLayoutRequested = true;
        //开启View 三大流程
        scheduleTraversals();
    }
}
```



### invalidate

View.invalidate() 表示当前绘制内容无效，需要重新绘制。向上遍历父布局，调用父布局的 invalidateChild() 方法，方法内区分了硬件加速绘制和软件绘制，最终调用到 ViewRootImpl 开启刷新流程。

```java
public void invalidate() {
    invalidate(true);
}

public void invalidate(boolean invalidateCache) {
    //invalidateCache 使绘制缓存失效
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}


void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache, boolean fullInvalidate) {
    ...
    //设置了跳过绘制标记
    if (skipInvalidate()) {
        return;
    }
    //PFLAG_DRAWN在draw()中被标识，表示此前该View已经绘制过
    //PFLAG_HAS_BOUNDS在setFrame()中被标识，表示该View已经layout过，确定过坐标了
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
       ...) {
        if (fullInvalidate) {
            //清除PFLAG_DRAWN标记
            mPrivateFlags &= ~PFLAG_DRAWN;
        }
        //需要绘制
        mPrivateFlags |= PFLAG_DIRTY;
        if (invalidateCache) {
            //1、加上绘制失效标记
            //2、清除绘制缓存有效标记
            //这两标记在硬件加速绘制分支用到
            mPrivateFlags |= PFLAG_INVALIDATED;
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }
        
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            //记录需要重新绘制的区域 damge，该区域为该View尺寸
            damage.set(l, t, r, b);
            //p 为该View的父布局，调用父布局的invalidateChild
            p.invalidateChild(this, damage);
        }
        ...
    }
}

public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null && attachInfo.mHardwareAccelerated) {
        //如果是支持硬件加速，则走该分支
        onDescendantInvalidated(child, child);
        return;
    }
    //软件绘制
    ViewParent parent = this;
    if (attachInfo != null) {
        ...
        do {
            View view = null;
            if (parent instanceof View) {
                view = (View) parent;
            }
            ...
            parent = parent.invalidateChildInParent(location, dirty);
        } while (parent != null);
    }
}
```

硬件加速 invalidate 流程：

- onDescendantInvalidated() 会向上遍历父布局
- 如果该View需要重绘，则加上PFLAG_INVALIDATED 标记
- 设置重绘区域为 Window 大小， ViewRootImpl.scheduleTraversals() 开启刷新流程

```java
//View.java
public void onDescendantInvalidated(@NonNull View child, @NonNull View target) {
    mPrivateFlags |= (target.mPrivateFlags & PFLAG_DRAW_ANIMATION);
    
    if ((target.mPrivateFlags & ~PFLAG_DIRTY_MASK) != 0) {
       //此处都会走
        mPrivateFlags = (mPrivateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DIRTY;
        //清除绘制缓存有效标记
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
    }
    
    if (mLayerType == LAYER_TYPE_SOFTWARE) {
        //如果是开启了软件绘制，则加上绘制失效标记
        mPrivateFlags |= PFLAG_INVALIDATED | PFLAG_DIRTY;
        //更改target指向
        target = this;
    }

    if (mParent != null) {
        //调用父布局的onDescendantInvalidated
        mParent.onDescendantInvalidated(this, target);
    }
}

//ViewRootImpl.java
public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
    if ((descendant.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
        mIsAnimating = true;
    }
    invalidate();
}

void invalidate() {
    //mDirty 为脏区域，也就是需要重绘的区域，mWidth，mHeight 为Window尺寸
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        //开启View 三大流程
        scheduleTraversals();
    }
}
```

软件绘制 invalidate() 流程：

- invalidateChildInParent() 会向上遍历父布局
- 不断修正需要重绘的区域 dirty
- 设置重绘区域 dirty 并集， ViewRootImpl.scheduleTraversals() 开启刷新流程

```java
//ViewGroup.java
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    //dirty 为失效的区域，也就是需要重绘的区域
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID)) != 0) {
        //该View绘制过或者绘制缓存有效
        if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE))
                != FLAG_OPTIMIZE_INVALIDATE) {
            //修正重绘的区域
            dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                    location[CHILD_TOP_INDEX] - mScrollY);
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                //如果允许子布局超过父布局区域展示
                //则该dirty 区域需要扩大
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }
            final int left = mLeft;
            final int top = mTop;
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                //默认会走这，如果不允许子布局超过父布局区域展示，则取相交区域
                if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                    dirty.setEmpty();
                }
            }
            //记录偏移，用以不断修正重绘区域，使之相对计算出相对屏幕的坐标
            location[CHILD_LEFT_INDEX] = left;
            location[CHILD_TOP_INDEX] = top;
        } else {
            ...
        }
        //标记缓存失效
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        if (mLayerType != LAYER_TYPE_NONE) {
            //如果设置了缓存类型，则标记该View需要重绘
            mPrivateFlags |= PFLAG_INVALIDATED;
        }
        //返回父布局
        return mParent;
    }
    return null;
}

//ViewRootImpl.java
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    checkThread();
    if (dirty == null) {
        //脏区域为空，则默认刷新整个窗口
        invalidate();
        return null;
    } else if (dirty.isEmpty() && !mIsAnimating) {
        return null;
    }
    ...
    //合并脏区域，重绘
    invalidateRectOnScreen(dirty);
    return null;
}

private void invalidateRectOnScreen(Rect dirty) {
    final Rect localDirty = mDirty;
    //合并脏区域，取并集
    localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
    ...
    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        //开启View的三大绘制流程
        scheduleTraversals();
    }
}
```

### postInvalidate

因为 ViewRootImpl 的 invalidateChildInParent() 中调用了 checkThread()，invalidate() 只能在主线程调用，所以在子线程中开启刷新流程，需要调用 postInvalidate()。

```java
//View.java
public void postInvalidate() {
    postInvalidateDelayed(0);
}

public void postInvalidateDelayed(long delayMilliseconds) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    }
}

//ViewRootImpl.java
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
    //此处Message.obj = view
    Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
    mHandler.sendMessageDelayed(msg, delayMilliseconds);
}

public void handleMessage(Message msg) {
    switch (msg.what) {
        case MSG_INVALIDATE:
            //obj 即为待刷新的View
            ((View) msg.obj).invalidate();
            break;
            ...
    }
}
```

对于单个 View 的视图重绘总结为：

- 需要重绘时调用 invalidate，当涉及View的尺寸、位置变化时使用 requestLayout

- invalidate 调用后只会出发 Draw 流程
- requestLayout 会触发 Measure、Layout 过程，如果尺寸发生改变，会调用 invalidate
- 如果不确定requestLayout 是否触发invalidate，可在requestLayout后继续调用invalidate



[Android布局优化（一）LayoutInflate — 从布局加载原理说起 - 简书 (jianshu.com)](https://www.jianshu.com/p/8ca35e86d476)

[Android布局优化（三）使用AsyncLayoutInflater异步加载布局 - 简书 (jianshu.com)](https://www.jianshu.com/p/8548db25a475)

[Android中自定义样式与View的构造函数中的第三个参数defStyle的意义 - AngelDevil - 博客园 (cnblogs.com)](https://www.cnblogs.com/angeldevil/p/3479431.html)

[ViewStub你肯定听过，但是这些细节了解吗？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650829913&idx=1&sn=4f6ed4bc4e232a1313170ce324dc50ca&chksm=80b7a0c7b7c029d19ef094e85b2d6c95c04859272b988af1c3b92b75b083cc923d47e6a455f1)

[Android 布局优化之include与merge - 简书 (jianshu.com)](https://www.jianshu.com/p/5f6f1f12fd07)

[addView方法分析 - 简书 (jianshu.com)](https://www.jianshu.com/p/ce2d6baabb36)

[Android invalidate/postInvalidate/requestLayout-彻底厘清 - 简书 (jianshu.com)](https://www.jianshu.com/p/02073c90ef98)

[requestLayout 这么问？谁能会？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650837018&idx=1&sn=7da933906bf6c709b7429aafeff12dc1&chksm=80b74484b7c0cd92fe2c993e9c35f351d7563ca9cfddc5a3464965f06ec9ee078d5349fd5a0e)
