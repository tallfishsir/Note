#  Activity 启动过程学习 

## 跳转页面代码分析

### 开始跳转

Activity 的 startActivity 有多个重载方法，最终会调用 startActivityForResult 方法，在 startActivityForResult 方法内部会调用 Instrumentation 的 execStartActivity 方法，实现与 AMS 的通信来执行页面跳转逻辑。

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    try {
        ...
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getBasePackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, op
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}

public static void checkStartActivityResult(int res, Object intent) {
        if (!ActivityManager.isStartResultFatalError(res)) {
            return;
        }

        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);
            case ActivityManager.START_PERMISSION_DENIED:
                throw new SecurityException("Not allowed to start activity "
                        + intent);
            case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
                throw new AndroidRuntimeException(
                        "FORWARD_RESULT_FLAG used while also requesting a result");
            case ActivityManager.START_NOT_ACTIVITY:
                throw new IllegalArgumentException(
                        "PendingIntent is not an activity");
            case ActivityManager.START_NOT_VOICE_COMPATIBLE:
                throw new SecurityException(
                        "Starting under voice control not allowed for: " + intent);
            case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                        "Session calling startVoiceActivity does not match active session");
            case ActivityManager.START_VOICE_HIDDEN_SESSION:
                throw new IllegalStateException(
                        "Cannot start voice activity on a hidden session");
            case ActivityManager.START_ASSISTANT_NOT_ACTIVE_SESSION:
                throw new IllegalStateException(
                        "Session calling startAssistantActivity does not match active session");
            case ActivityManager.START_ASSISTANT_HIDDEN_SESSION:
                throw new IllegalStateException(
                        "Cannot start assistant activity on a hidden session");
            case ActivityManager.START_CANCELED:
                throw new AndroidRuntimeException("Activity could not be started for "
                        + intent);
            default:
                throw new AndroidRuntimeException("Unknown error code "
                        + res + " when starting " + intent);
        }
    }
```

checkStartActivityResult 方法用于检查页面跳转结果的检验，通常遇到的页面跳转 Exception 都是在这里产生。

### 启动进程

AMS 检查下一个 Activity 所属的进程是否存在，如果不存在会创建并启动对应的进程，然后调用 ActivityThread 的 handleBindApplication 创建目标进程的 Application 对象

```java
private void handleBindApplication(AppBindData data) {
    ...
    mInstrumentation = new Instrumentation();
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
}

public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    String appClass = mApplicationInfo.className;
    final java.lang.ClassLoader cl = getClassLoader();
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
    ...
    instrumentation.callApplicationOnCreate(app);
}
```

创建 ContextImpl 和 Instrumentation 对象，然后通过 ClassLoad 创建 Application 对象并调用 Application 的 attach 方法，然后通过 Instrumentation 的 callApplicationOnCreate 方法调用 Application 的 onCreate 方法。

### 启动页面

AMS 调用 ActivityThread 的 handleLaunchActivity 启动对应的 Activity，在 performLaunchActivity 方法中完成 Activity 对象的创建及其重要成员变量的初始化，在 Activity 的 attach 方法中完成 Activity 和 ContextImpl、Window 等重要数据绑定关系。

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
    final Activity a = performLaunchActivity(r, customIntent);
}

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    // 从 ActivityClientRecord 中的组件信息，创建 ContextImpl 对象
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        // 使用类加载器创建Activity对象
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        ...
    }
    ...
    // 绑定 Activity 与 ContextImpl、Window 等重要数据
    activity.attach(appContext, this, getInstrumentation(), r.token,
        r.ident, app, r.intent, r.activityInfo, title, r.parent,
        r.embeddedID, r.lastNonConfigurationInstances, config,
        r.referrer, r.voiceInteractor, window, r.configCallback,
        r.assistToken);
    ...
    // 调用 Activity onCreate 方法
    mInstrumentation.callActivityOnCreate(activity, r.state);
}

final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
        IBinder shareableActivityToken) {
    // 将 ActivityThread 中的 ContextImpl 对象与 Activity 绑定
    attachBaseContext(context);
    ...
    // 创建 Activity 对应的 PhoneWindow 对象
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    // 设置 onAttachedToWindow/onDetachedFromWindow/dispatchTouchEvent 等回调接口
    mWindow.setCallback(this);
    ...
    // 将 ActivityThread 中的若干重要数据对象与 Activity 绑定
    mMainThread = aThread;
    mInstrumentation = instr;
    mApplication = application;
    mActivityInfo = info;
    ...
    // 绑定 WindowManager 对象
    mWindow.setWindowManager((WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                             mToken, mComponent.flattenToString(),
                             (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    mWindowManager = mWindow.getWindowManager();
}
```

在 Activity.attach() 方法中，创建了 PhoneWindow 对象并绑定了 WindowManager。

### 页面初始化

Activity 的 onCreate 方法中会使用 setContentView 来渲染 DecorView 加载页面布局。

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}

// PhoneWindow.java
public void setContentView(int layoutResID) {
    // 创建 DecorView
    installDecor();
    // ID_ANDROID_CONTENT 的布局作为 Activity 的布局父布局
    mLayoutInflater.inflate(layoutResID, mContentParent);
    // 利用在 Activity.attach 方法中设置的回调通知 Activity
    getCallback().onContentChanged(); 
}

private void installDecor() {
    // DecorView 是 FrameLayout 子类，返回一个新创建 DecorView 对象
    mDecor = generateDecor(-1);
    // 找到资源id是 ID_ANDROID_CONTENT 的子布局
    mContentParent = generateLayout(mDecor);
}

// 根据配置的主题等信息，修改 DecorView 中的内容
protected ViewGroup generateLayout(DecorView decor) {
    ...
    // 找到资源id是 ID_ANDROID_CONTENT 的子布局作为 Activity 的布局根布局
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    return contentParent
}
```

在 onCreate 中创建 DecorView 并初始化布局文件，但此时 DecorView 还没有被 WindowManager 加到 PhoneWindow 中，所以是不会显示在界面上的。

### 显示页面

AMS 调用 ActivityThread 的 handleResumeActivity 显示页面内容

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
                                 String reason) {
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    final Activity a = r.activity;
    r.window = r.activity.getWindow();
}

public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest, 
                                                  String reason) {
    final ActivityClientRecord r = mActivities.get(token);
    // 调用 Activity onResume 方法
    r.activity.performResume(r.startsNotResumed, reason);
    ...
    // DecorView 被 WindowManager 加到 PhoneWindow 中
    View decor = r.window.getDecorView();
    ViewManager wm = a.getWindowManager();
    wm.addView(decor, l);
    ...
    // 创建 ViewRootImpl 对象，开始刷新界面
    ViewRootImpl root = new ViewRootImpl(view.getContext(), display);
    root.setView(view, wparams, panelParentView, userId);
}

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
                    int userId) {
    // ViewRootImpl 的成员变量 mView 赋值为 DecorView
    mView = view;
    // 刷新界面
    requestLayout();
    // 将 DecorView 的 mParent 指向 ViewRootImpl 对象
    view.assignParent(this);
}

public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        // 检查当前线程
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        // mHandler 发送同步屏障信息，优先处理异步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // mTraversalRunnable 是 Runnable 的子类，会往主线程的Handler发送一条异步消息
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

void doTraversal() {
    if (mTraversalScheduled) {
        ...
        performTraversals();
        ...
    }
}
```

在 onResume 中 DecorView 被 WindowManager 添加到 PhoneWindow 中，显示在页面上。并创建了 ViewRootImpl 对象，将 DecorView 的 parent 指向 ViewRootImpl 。

在 ViewRootImpl 构造函数中，初始化了 View.AttachInfo 对象和 ViewRootHandler 对象。

View.AttachInfo 这个类存储一些 View attach 到 Window 时候的一些信息，并保存了 ViewRootHandler 对象引用。

ViewRootHandler  类是 Handler 的子类，内部负责处理一些 View 布局绘制时的逻辑。

### 刷新页面

performTraversals 方法开始执行 View 体系的测量布局绘制流程

```java
private void performTraversals() {
    // DecorView 赋值为局部变量 host
    final View host = mView;
    ...
    // 执行 measure 逻辑
    measureHierarchy(host, lp, res, desiredWindowWidth, desiredWindowHeight);
    ...
    // 执行 layout 逻辑
    performLayout(lp, mWidth, mHeight);
    ...
    // 通知布局结束
    mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
    ...
    // 执行 draw 逻辑
    performDraw();
    ...
}

private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
        final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    ...
    // 构建最初的宽度高度 MeasureSpec 数据
    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
    childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
    // 开启 View 体系的测量流程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
    }
}

private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth
        int desiredWindowHeight) {
    final View host = mView;
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
}

private void performDraw() {
    final boolean fullRedrawNeeded =
                mFullRedrawNeeded || mReportNextDraw || mNextDrawUseBlastSync;
    // 选择硬件绘制还是软件绘制
    draw(fullRedrawNeeded);
}
```

## Activity 跳转流程相关类和对象

### Instrumentation

ActivityThread 的 mInstrumentation 对象是在启动进程过程的 handleBindApplication() 方法中被创建出来的。

在启动页面过程的 performLaunchActivity() 方法中，通过 Activity.attach() 方法中传给 Activity。

它完成了 Activity 对象的类加载器创建，启动页面，调用 Application/Activity 生命周期方法。

### ViewRootImpl

ViewRootImpl 是在通过 WindowManager 调用 addView() 添加 Window（View树）时创建的。然后被添加的 View 树的 parant 指向了 ViewRootImpl。后续完成与 WindowManagerService 的通信以及 View 体系的测量布局绘制流程。

### ContextImpl

Context 体系的继承关系：

- Context 是抽象类，提供了应用环境全局信息的接口
- ContextWrapper 是上下文功能的封装类
- ContextImpl 是上下文功能的实现类

Applicaiton 和 Service 直接继承自 ContextWrapper 类，Activity 继承自 ContextThemeWrapper 类，因为 Activity 提供 UI 显示，需要有主题。应用的 Context 数量 = Activity 数量 + Service 数量 + 1（Application 数量）

ContextWrapper 包含一个真正的 ContextImpl 的引用 mBase，然后就是 ContextImpl 的装饰者模式。

Application 的 ContextImpl 对象是在 performLaunchActivity() 方法中创建的。

Activity 的 ContextImpl 对象是在 Activity.attach() 方法中，由 Application 的 ContextImpl 对象传入的。

[3分钟看懂Activity启动流程 - 简书 (jianshu.com)](https://www.jianshu.com/p/9ecea420eb52)