#  Android Activity

## 跨进程通信

### Binder 机制

Binder 是 Android 系统中一种跨进程通信（IPC, Inter-Process Communication）机制。它允许不同进程之间相互通信、调用和共享数据。Binder 机制基于客户端-服务端（Client-Server）架构，允许一个进程（客户端）请求另一个进程（服务端）提供的服务。以下是 Binder 机制的原理和关键组成部分：

- Binder 驱动程序：
  Binder 机制的底层实现是基于 Linux 内核中的 Binder 驱动程序。Binder 驱动程序负责管理进程之间的通信，确保数据在进程间正确传输。Android 应用程序通过打开 /dev/binder 设备文件来与 Binder 驱动程序交互。

- Binder 代理（Proxy）：
  Binder 代理是客户端进程中的一个 Java 对象，它实现了服务端提供的接口。客户端通过调用 Binder 代理的方法来实现对服务端的远程调用。Binder 代理会将调用的方法和参数封装成一个 Parcel 对象，并通过 Binder 驱动程序将其发送给服务端进程。

- Binder 对象：
  Binder 对象位于服务端进程中，它也实现了服务端提供的接口。当服务端接收到客户端的远程调用请求时，Binder 对象会根据请求中的方法和参数执行相应的操作，并将结果返回给客户端。

- ServiceManager：
  ServiceManager 是 Android 系统中的一个核心服务，负责管理所有的系统服务和自定义服务。当服务端进程启动时，它会将自己的 Binder 对象注册到 ServiceManager。客户端进程在向服务端发起远程调用前，需要先从 ServiceManager 获取服务端的 Binder 代理对象。

- AIDL（Android Interface Definition Language）：
  AIDL 是一种定义 Android IPC 接口的语言。通过 AIDL 文件，开发者可以定义跨进程通信时所需的接口和数据类型。AIDL 编译器会根据 AIDL 文件生成对应的 Java 接口文件和实现文件，供客户端和服务端使用。

Binder IPC 是通过内存映射 (mmap) 实现的，一次完整的 Binder IPC 通信过程是这样的：

- Binder 驱动在内核空间创建一个「数据接收缓冲区」和一个「内核缓冲区」
- 建立「数据接收缓冲区」和「内核缓冲区」的内存映射关系
- 建立「数据接收缓冲区」和「接收进程用户空间地址」的内存映射关系
- 发送方进程调用 copyfromuser() 将数据拷贝到「内核缓冲区」

### AIDL 使用过程

Android 中实现进程间通信方式最多的就是 AIDL，当我们定义好 AIDL 文件，IDE 会在编译期间生成 IPC 的 Java 文件，这个 Java 文件包含了一个 Stub 静态的抽象类和一个 Proxy 静态类，Proxy 是 Stub 的静态内部类。实际上，可以手动编码实现跨进程通信。

手动编码实现之前，需要先了解一些类或者接口的含义：

- IInterface：代表 Server 进程对象能够提供什么方法，对应的就是 AIDL 文件中定义的接口
- IBinder：代表一种跨进程通信能力的接口，只要实现了这个接口，这个对象就能跨进程传输
- Binder：Java 层的 Binder 类，代表 Binder 本地对象，继承自 IBinder

首先定义 Server 进程能够提供的方法

```java
public interface IBookManager extends IInterface {
    // Binder 的唯一标识，一般用当前类名表示
    static final String DESCRIPTOR = "com.tallfish.demo.binder.book.IBookManager";
    // Server 提供的方法和方法 id
    static final int TRANSACTION_getBookList = IBinder.FIRST_CALL_TRANSACTION + 0;
    static final int TRANSACTION_addBook = IBinder.FIRST_CALL_TRANSACTION + 1;
    public List<Book> getBookList() throws RemoteException;
    public void addBook(Book book) throws RemoteException;
}
```

然后实现 Binder 类

```java
public class BookManagerImplBinder extends Binder implements IBookManager {
    public BookManagerImpl() {
        this.attachInterface(this, DESCRIPTOR);
    }
    // 用于将服务端的 Binder 对象转换成客户端所需要的 AIDL 接口类型的对象
    // 这种转换过程区分进程
    // 客户端服务端同一进程，此方法返回的 BookManagerImplBinder 对象本身
    // 客户端服务端不在同一进程，此方法返回的是系统封装后的 BookManagerImplBinder.Proxy 对象
    public static IBookManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        android.os.IInterface iInterface = obj.queryLocalInterface(DESCRIPTOR);
        if (iInterface != null && iInterface instanceof IBookManager) {
            return (IBookManager) iInterface;
        }
        return new BookManagerImplBinder.Proxy(obj);
    }
    // 返回当前的 Binder 对象
    @Override
    public IBinder asBinder() {
        return this;
    }
    // 方法运行在服务端中的 Binder 线程池，当客户端跨进程请求时，请求会被系统底层封装后交此方法处理
    // 服务端根据 code 确定客户端所请求的目标方法是什么
    // 从 data 中取出目标方法需要的参数
    // 目标方法执行后，向 reply 写入返回值
    // 如果方法返回 true 请求成功，false 请求失败
    @Override
    protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags) throws RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION:
                reply.writeString(DESCRIPTOR);
                break;
            case TRANSACTION_getBookList:
                data.enforceInterface(DESCRIPTOR);
                List<Book> list = this.getBookList();
                reply.writeNoException();
                reply.writeTypedList(list);
                return true;
            case TRANSACTION_addBook:
                data.enforceInterface(DESCRIPTOR);
                Book book;
                if (0 != data.readInt()) {
                    book = (Book) Book.CREATOR.createFromParcel(data);
                } else {
                    book = null;
                }
                this.addBook(book);
                reply.writeNoException();
                return true;
            default:
        }
        return super.onTransact(code, data, reply, flags);
    }
    // Server 真正实现功能的方法
    @Override
    public List<Book> getBookList() throws RemoteException {
        return null;
    }
    // Server 真正实现功能的方法
    @Override
    public void addBook(Book book) throws RemoteException {
    }
    // Client 获取到的 Binder
    private static class Proxy implements IBookManager{
        private IBinder mRemote;
        public Proxy(IBinder mRemote) {
            this.mRemote = mRemote;
        }
        @Override
        public IBinder asBinder() {
            return mRemote;
        }
        public java.lang.String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }
        // 方法运行在客户端，当客户端远程调用此方法时，
        // 将 参数data 出参reply 返回值 创建
        // 然后调用 transact 方法（此方法会调用到 onTransact） 同时线程挂起直到远程调用结束
        @Override
        public List<Book> getBookList() throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            List<Book> list;
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                mRemote.transact(TRANSACTION_getBookList, data, reply, 0);
                reply.readException();
                list = reply.createTypedArrayList(Book.CREATOR);
            } finally {
                data.recycle();
                reply.recycle();
            }
            return list;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                if (book != null) {
                    data.writeInt(1);
                    book.writeToParcel(data, 0);
                } else {
                    data.writeInt(0);
                }
                mRemote.transact(TRANSACTION_addBook, data, reply, 0);
                reply.readException();
            } finally {
                data.recycle();
                reply.recycle();
            }
        }
    }
}
```

## 系统启动流程

当按下电源开机后，ROM 中的 BootLoader 会被加载到内存中，Bootloader 初始化软硬件环境后，启动内核。

Linux 内核启动过程中会创建 init 进程，init 进程是用户空间的第一个进程 。它创建和挂载启动所需要的文件目录，然后初始化并启动属性服务，最后解析 init.rc 配置文件，启动 zygote 进程。

Zygote 进程启动了 JVM 虚拟机，然后创建一个类加载器 PathClassLoader 用于后续加载 Java 类，最后调用 forkSystemServer 函数 fork 出 system_server 进程。

system_server 进程承载了 framework 层的核心业务，启动过程中启动了 Binder 线程池，创建了 SystemServiceManager，用于对系统服务进行创建、启动和生命周期管理，最后启动了引导服务、核心服务、其他服务，比如 AMS、WMS、PMS 等。

AMS 的启动过程 systemReady() 中，会通过 ActivityTaskManagerService 获取控制器，启动 Launcher。

![image-20221019220055901](C:\Users\24594\AppData\Roaming\Typora\typora-user-images\image-20221019220055901.png)

## Activity 跳转源码

![](https://upload-images.jianshu.io/upload_images/1869462-882b8e0470adf85a.jpg)

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

AMS 检查下一个 Activity 所属的进程是否存在，如果不存在就会通过调用 startProcessLocked 函数向 Zygote 进程发送请求创建对应进程。

```java
private final void startProcessLocked(ProcessRecord app, String hostingType, String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    int uid = app.uid;//获取应用进程的id
    ...
    if (entryPoint == null) entryPoint = "android.app.ActivityThread";
    //应用程序进程是通过Zygote进程fock产生的
    Process.ProcessStartResult startResult = Process.start(entryPoint,
                  app.processName, uid, uid, gids, debugFlags, mountExternal,
                  app.info.targetSdkVersion, app.info.seinfo, requiredAbi,
                  instructionSet, app.info.dataDir, entryPointArgs);
}

//RuntimeInit.java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) throws ZygoteInit.MethodAndArgsCaller {
    ...
    invokeStaticMain(args.startClass, args.startArgs, classLoader);//1
}

//RuntimeInit.java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader) throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl;
    try {
        cl = Class.forName(className, true, classLoader);//android.app.ActivityThread类
    } catch (ClassNotFoundException ex) {
    }
    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
    }
    ...
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);//3
}
```

创建应用进程的过程中，会 ActivityThread 的 main 方法。

```java
final ApplicationThread mAppThread = new ApplicationThread();
final H mH = new H();

public static void main(String[] args) {
    ...
    //创建当前主线程的Looper对象
    Looper.prepareMainLooper();
    ...
    //创建ActivityThread对象
    ActivityThread thread = new ActivityThread();
    //发送创建Application消息
    thread.attach(false, startSeq);
    if (sMainThreadHandler == null) {
        //Handler类型的H对象
        sMainThreadHandler = thread.getHandler();
    }
    ...
    //开启looper循环
    Looper.loop();
}

private void attach(boolean system, long startSeq) {
    ...
    //获得ActivityManagerService实例
    final IActivityManager mgr = ActivityManager.getService();
    try {
        //ApplicationThread作为IApplicationThread的一个实例与AMS绑定
        //承担起最后发送Activity生命周期、及其它一些消息的任务
        mgr.attachApplication(mAppThread);
    } catch (RemoteException ex) {
    }
    ...
}

//ActivityManagerService.java
public void attachApplication(IApplicationThread app){
  ...
  //在AMS中调用ApplicationThread的bindApplication方法初始化Application
  thread.bindApplication();
  ...
}

//ApplicationThread
public final void bindApplication(String processName, 
    ApplicationInfo appInfo,...){
    ...
    sendMessage(H.BIND_APPLICATION, data);
}

//在 H 对象的中处理Message
public void handleMessage(Message msg) {
    ...
    switch (msg.what) {
        case BIND_APPLICATION:
            AppBindData data = (AppBindData)msg.obj;
            handleBindApplication(data);
            break;
    }
}
```

ActivityThread 的 handleBindApplication 创建目标进程的 Application 对象。

```java
private void handleBindApplication(AppBindData data) {
    ...
    //通过反射初始化Instrumentation对象
    mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
}

public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    String appClass = mApplicationInfo.className;
    final java.lang.ClassLoader cl = getClassLoader();
    //初始化ContextImpl对象
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    //创建Application实例
    app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
    ...
    //调用Application的onCreate()方法
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

## 序列化

Parcelable 和 Serializable 都是用于在 Android 中传递对象的方法。它们之间的主要区别在于性能和实现方式。以下是它们之间的一些区别：

Parcelable 在性能上优于 Serializable，尤其是在内存使用和处理速度方面。Parcelable 是 Android 专门为传递数据对象而设计的接口，它的实现方式直接将数据对象的属性写入字节流，避免了创建额外的对象。而 Serializable 是 Java 标准序列化接口，它使用反射机制来序列化和反序列化对象，可能会创建额外的临时对象，从而导致更高的内存开销和更慢的处理速度。

实现 Parcelable 接口相对复杂一些。需要手动实现对象序列化和反序列化的过程，包括编写 writeToParcel() 方法和一个特殊的构造函数。此外，还需要实现一个名为 CREATOR 的静态变量，用于创建新对象和数组。

实现 Serializable 接口相对简单，只需要让类实现 Serializable 接口即可。如果类中的所有属性都是可序列化的，那么不需要编写任何额外的代码。但是，如果类中包含不可序列化的属性，需要在类中声明 transient 关键字来标记这些属性，使其在序列化过程中被忽略。

在 Android 中，Parcelable 通常用于在不同组件（如 Activity、Service 等）之间传递数据对象。由于其优秀的性能表现，它是 Android 推荐的对象传递方式。

Serializable 更适用于跨平台的对象序列化，如将对象存储到磁盘或通过网络发送到其他平台。但由于其性能较差，不推荐在 Android 中频繁使用。



[3分钟看懂Activity启动流程 - 简书 (jianshu.com)](https://www.jianshu.com/p/9ecea420eb52)

[Android深入理解Context（一）Context关联类和Application Context创建过程 | BATcoder - 刘望舒 (liuwangshu.cn)](http://liuwangshu.cn/framework/context/1-application-context.html)