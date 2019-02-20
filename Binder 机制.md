##### Q1：常见的跨进程方案有哪些，为什么还会有 Binder 这种跨进程方案的出现？

Android 系统是基于 Linux 内核的，Linux 已经提供了管道、共享内存、消息队列、Socket 等进程间通信方案，但他们各自都有不适应移动环境的问题。从性能角度来看，Socket 传输效率低，主要用于跨网路的进程间通信；消息队列和管道采用的是存储-转发方式，数据需要先从发送方缓存区拷贝到内核缓存区，再从内核缓存区拷贝到接收缓存区，整个过程一共进行了两次数据的拷贝操作；共享内容不需要进行拷贝数据，但它控制复杂。

Binder 进程间通信，通过操作系统的 mmap() 方法将用户空间的一块内存区域映射到内核空间，用户对这块内存区域的修改，可以直接反映到内核空间。反之，内核空间对这段区域的修改也能直接反应到用户空间，整个过程中数据只拷贝了一次，性能上仅次于共享内存方案。

##### Q2：Binder 通信模型是什么样？

Binder 的通信模型采用 client-server 架构，它的实现原理是：Binder 驱动在内核空间创建一个内核数据接收缓存区和内核缓存区，然后建立内核缓存区和内核数据接收缓存区之间的映射关系，内核数据接收缓存区和接收进程用户空间之间的映射关系。当发送方进程通过系统调用 copy_from_user 将数据 copy 到内核缓存区，由于之前的映射关系，数据会直接反应到接收方进程的用户空间。

整个 Binder 的进程间通信，存在四个角色：Client、Server、ServiceManager、Binder 驱动，其中 Client、Server、ServiceManager 运行在用户空间、Binder 驱动运行在内核空间。

- Binder 驱动：负责进程之间 Binder 通信的建立，Binder 在进程之间的传递、数据包在进程间的传递等。它是 Linux 的动态内核可加载模块，可以被独立编译，但不能独立运行。在运行时被链接到内核。
- ServiceManager：内部存在一张表用来记录当前系统 Binder 实体，使得 Client 可以通过 Binder 的名字获得一个 Binder 实体的引用。
- Server：Server 会创建一个自己的 BInder 实体，并向 ServiceManager 注册。
- Client：Server 向 ServerManager 注册 Binder 后，Client 就可以通过名字获取到 Binder 引用，然后调用 Server 提供的功能。

##### Q3：Binder 如何实现跨进程通信，简单描述一次 Binder 通信的过程？

1. 因为 Server 向 ServiceManager 注册也是进程间通信，所以 ServiceManager 的Binder 是 Binder 驱动自动创建的，ServiceManager 的实体引用在所有的 Client 中都固定是 0
2. Server  创建了 Binder，并为它起一个字符形式，可读易记得名字，然后通过 Binder 驱动向 ServiceManager 注册 Binder，表明可以对外提供服务。Binder 驱动为这个 Binder 创建位于内核中的实体节点以及 ServiceManager 对这个实体的引用。然后将名字和实体引用打包传给 ServiceManager，ServiceManager 将其填入查找表中。
3. Client 通过名字在 Binder 驱动下从 ServiceManager 中获取到 Binder 引用，利用这个 Binder引用实现进程间通信。

从上面的过程中可以发现，不同角度下的 Binder 有着不同的定义：

- 从进程间通信角度来看，Binder 是一种进程间通信的机制；
- 从 Server 进程角度看，Binder 指的是位于内核空间的 Binder 实体对象；
- 从 Client 进程角度看，Binder 指的是 Binder 的引用，是 Binder 实体对象的一个远程代理；
- 从传输角度看，Binder 是一个可以跨进程传输的对象，Binder 驱动会对这个对象处理，自动完成代理对象和本地对象的转换。

##### Q4：Android AIDL 和 Binder 之间的关系？

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

##### Q5：对象序列化 Serializable 和 Parcelable 比较？

```java
public class User implements Serializable {
    private static final long serialVersionUID = 512345678910L;
    public int userId;
    public User(int userId, String userName, boolean isMale) {
        this.userId = userId;
    }
}

```

Serializable 序列化对象只需要让类实现 Serializable 接口，反序列化会和序列化时候的 serialVersionUID 进行比较，如果不同，直接不进行反序列化了，就抛出异常。如果不手动写这个值，它会根据当前这个类结构去生成的hash值为值。所以当我们把这个类结构更改后，再去反序列化就报错了。

```java
public class Book implements Parcelable {
    public int bookId;
    public String bookName;
    public Book(int bookId, String bookNama) {
        this.bookId = bookId;
        this.bookName = bookNama;
    }
    public Book(Parcel source) {
        this.bookId = source.readInt();
        this.bookName = source.readString();
    }
    @Override
    public int describeContents() {
        return 0;
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }
    public static final Parcelable.Creator CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel source) {
            return new Book(source);
        }
        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };
}
```

##### Q6：从 Activity 启动过程谈谈 ActivityManagerService？

Activity 的启动过程是从 startActivity 方法开始的，startActivity 方法有好几个重载方法，但他们最后都会调用到 startActivityForResult 方法：

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        ...
        Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(this, mMainThread.getApplicationThread(), mToken, this, intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult( mToken, mEmbeddedID, requestCode, ar.getResultCode(), ar.getResultData());
        }
        if (requestCode >= 0) {
            mStartedActivity = true;
        }
        ...
    } else {
        ...
    }
}
```

以上代码有需要注意，启动 Activity 实际上是从 mInstrumentation.execStartActivity 开始的，方法的第二个ApplicationThread 类型的参数是 ActivityThread 的一个内部类。然后追踪下其源码。

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

从上面的代码可以看到，启动 Activity 的实现是由 ActivityManager.getService() 的 startActivity 方法完成的。而它的返回值会传给 checkStartActivityResult 来检查启动 Activity 的结果，如果无法正确启动 Activity 时，这个方法就会抛出异常。

说回到 ActivityManager.getService()，它的返回值实际上是实现了 IActivityManager 接口的 ActivityManagerService ，接着追踪其源码。

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int  requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
            userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, "startActivityAsUser");
}
```

Activity 的启动过程又转移到 ActivityStarter 的 startActivityMayWait 方法，最终会调用到 app.thread.scheduleLaunchActivity 方法，这个 app.thread 是 IApplicationThread 类型，它的实现是 ActivityThread 中的内部类 ApplicationThread。

```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
    updateProcessState(procState, false);
    ActivityClientRecord r = new ActivityClientRecord();
    r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.referrer = referrer;
    r.voiceInteractor = voiceInteractor;
    r.activityInfo = info;
    r.compatInfo = compatInfo;
    r.state = state;
    r.persistentState = persistentState;
    r.pendingResults = pendingResults;
    r.pendingIntents = pendingNewIntents;
    r.startsNotResumed = notResumed;
    r.isForward = isForward;
    r.profilerInfo = profilerInfo;
    r.overrideConfig = overrideConfig;
    updatePendingConfiguration(curConfig);
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

ApplicationThread.scheduleLaunchActivity 的内容就是发送一个启动 Activity 的消息交由 Handler 处理，这个 Handler 在 ActivityThread 创建时就初始化了，在 handleMessage 中处理 LAUNCH_ACTIVITY 的内容是：

```java
case LAUNCH_ACTIVITY: {
    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
    r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);
    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
} break;

private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    ...
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        ...
        handleResumeActivity(r.token, false, r.isForward,!r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
        if (!r.activity.mFinished && r.startsNotResumed) {
            performPauseActivityIfNeeded(r, reason);
            if (r.isPreHoneycomb()) {
                r.state = oldState;
            }
        }
    } else {
        try {
            ActivityManager.getService().finishActivity(r.token, Activity.RESULT_CANCELED, null,Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
}
```

从上面的源码可以看到，performLaunchActivity 方法最终完成了 Activity 对象的创建和启动过程，并且 ActivityThread 通过 handleResumeActivity方法来调用被启动 Activity 的 onResume 生命周期。performLaunchActivity  方法中主要做了以下几件事情：

1. 从 ActivityClientRecord 中获取待启动的 Activity 的组件信息
2. 通过 mInstrumenttation 的 newActivity 方法使用类加载器创建 Activity 对象
3. 通过 LoadedApk 的 makeApplication 方法尝试创建 Application 对象
4. 创建 ContextImpl 对象，并通过 Activity 的 attach 方法完成一些重要数据的初始化，比如 Window 的创建
5. mInstrumentation.callActivityOnCreate 调用 Activity 的 onCreate 方法
6. activity.performStart() 中调用 mInstrumentation.callActivityOnStart(this); 调用 Activity 的 onStart 方法

##### Q7：从 DecorView 的加载过程谈谈 WindowManagerService？

WindowManager 添加一个 Window 时，只需要调用 WindowManager.addView(View view, WindowManager.LayoutParams param)，第二个参数 WindowManager.LayoutParams 有两个成员变量比较重要：flags，type。

Flags 表示 Window 的属性，通过它可以控制 Window 的显示特性：

- FALG_NOT_FOCUSABLE

  表示 Window 不需要获取焦点，也不需要接收任何输入事件，此标记会同时启动 FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层的具有焦点的 Window。

- FLAG_NOT_TOUCH_MODAL

  表示当前 Window 区域以内的点击事件会自己处理，以外的点击事件会传递下一层的 Window。

- FLAG_SHOW_WHTN_LOCKED

  表示当前 Window可以显示在锁屏上

Type 表示 Window 的类型，Window 有三个类型：应用 Window、子 Window 和系统 Window。应用 Window 对应着一个 Activity，子 Window 不能单独存在，需要依附在特定的父 Window 之中，比如 Dialog，系统 Window 是需要声明权限才能创建的 Window，比如 Toast、系统状态栏。应用 Window 的层级范围是 1~99，子 Window 的层级范围是 1000~1999，系统 Window 的层级范围是 2000~2999，层级大的会覆盖在层级小的 Window 上面。一般系统层级可以指定 type = LayoutParams.TYPE_SYSTEM_OVERLAY。

我们需要先了解 WindowManger 对 View 的操作的实现，才能理解 Activity 和 Window 的绑定流程。WindowManager 继承 ViewManager 接口，只提供了三个方法：

```java
public interface ViewManager {
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

WindowManager 也是接口，它的实现是 WindowManagerImpl 类，上面的三个方法的实现是：

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
@Override
public void removeView(View view) {
    mGlobal.removeView(view, false);
}
@Override
public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.updateViewLayout(view, params);
}
```

mGlobal 是 WindowManagerGlobal 类型的单例对象，这个类中有四个成员变量：

```java
// 存储的是所有 Window 所对应的 View
private final ArrayList<View> mViews = new ArrayList<View>();
// 存储的是所有 Window 所对应的 ViewRootImpl
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
// 存储的是所有 Window 所对应的布局参数
private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>();
// 存储的是正在被删除的 View 对象，也就是已经调用 removeView 但还未完成的删除的 View
private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

WindowManagerGlobal 的添加操作会将入参 View、params 还有生成的 ViewRootImpl 对象存放在对应的容器中，最后通过 ViewRootImpl 的 setView() 更新界面并完成 Window的添加过程：

```java
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    ...
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        ...
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            ...
        }
    }
}
```

ViewRootImpl 的 setView 方法会调用 requestLayout 完成异步刷新请求，在 requestLayout 中最终会调用 scheduleTraversals() 这个 View 绘制的入口。setView() 请求异步刷新后，会调用 mWindowSession.addToDisplay() 来完成 Window 的添加过程。

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	...
    requestLayout();
	...	
    try {
    	mOrigWindowType = mWindowAttributes.type;
    	mAttachInfo.mRecomputeGlobalAttributes = true;
   		collectViewAttributes();
 	  	res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
            getHostVisibility(), mDisplay.getDisplayId(),
            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
            mAttachInfo.mOutsets, mInputChannel);
	}
    ...	
}
```

mWindowSession 是 IWindowSession 类型的对象，是一个 Binder 类，真正的实现类是 Session，而在 Session 的内部会通过 WindowManagerService 来实现 Window 的添加，WindowManagerService 内部会为每一个应用保留一个单独的 Session。

总结来说，一个 Window 添加一个 View 的流程是： WindowManagerImpl  → 单例的 WindowManagerGlobal（存储 View、ViewRootImpl、Params）→ ViewRootImpl → IWindowSession/Session IPC →WindowManagerService。

Activity 的启动过程中，在 ActivityThread 的 performLaunchActivity() 方法中会完成 Activity 的创建，然后会调用 Activity.attach() 完成 Activity 和 Window 的绑定。

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback) {
    attachBaseContext(context);
    mFragments.attachHost(null);
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    // Activity 实现了 Window 的 Callback 接口，比如：onAttachedToWindow onDetachedFromWindow 等
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ...
}
```

此时，Activity 与 Window 的关系已经绑定，但 Window 中添加的 View——DecorView 还没有生成。Activity 的 setContentView 的具体实现是与其绑定的 Window 完成的，Window 的具体实现上面已经看到是 PhoneWindow，分析下 PhoneWindow 的 setContentView 方法：

```java
@Override
public void setContentView(int layoutResID) {
    // 如果没有 DevorView 就创建
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }
    // 将 Activity 传入的 ViewId 添加到 DecorView 的 mContentParent 中
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    // 回调 Activity 的 onContentChanged 方法
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

创建的 DecorView 是一个 FrameLayout，一般来说它的内部包含标题栏和内容栏，内容栏的固定 Id 是 android.R.id.content。

```java
private void installDecor() {
    mForceDecorInstall = false;
    // 创建 DecorView
    if (mDecor == null) {
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    
    if (mContentParent == null) {
        // 根据不同的系统主题，加载不同的布局文件
        mContentParent = generateLayout(mDecor);
        mDecor.makeOptionalFitsSystemWindows();
        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);
        if (decorContentParent != null) {
            mDecorContentParent = decorContentParent;
            mDecorContentParent.setWindowCallback(getCallback());
            if (mDecorContentParent.getTitle() == null) {
                mDecorContentParent.setWindowTitle(mTitle);
            }
            final int localFeatures = getLocalFeatures();
            for (int i = 0; i < FEATURE_MAX; i++) {
                if ((localFeatures & (1 << i)) != 0) {
                    mDecorContentParent.initFeature(i);
                }
            }
            ...
        } 
        ...
}
```

在 Activity.attach() 中完成了 Activity 与 Window 关系的绑定，在 Activity.setContent() 中完成了 DecorView 的创建。最后在 ActivityThread 的 handleResumeActivity 方法中，会先调用 Activity.onResume() 接着会调用 Activity.makeVisible()，这个方法中真正完成了 Window 添加 DecorView 的过程。