# Window

Android 的 Window 机制就为了解决屏幕上 View 的显示混乱问题，让所有的 View 都按照秩序来显示，满足开发需求。Window 是一个抽象的概念，是 Window 机制中的操作单位，每一个 Window 对应一个 View 树。

## Window 相关属性

### type 属性

Window 根据 type 属性分为了应用程序窗口、子窗口、系统级窗口，通过这个属性决定了 Window 的显示次序。

```java
// 应用程序 Window 的开始值
public static final int FIRST_APPLICATION_WINDOW = 1;

// 应用程序 Window 的基础值
public static final int TYPE_BASE_APPLICATION = 1;

// 普通的应用程序
public static final int TYPE_APPLICATION = 2;

// 特殊的应用程序窗口，当程序可以显示 Window 之前使用这个 Window 来显示一些东西
public static final int TYPE_APPLICATION_STARTING = 3;

// TYPE_APPLICATION 的变体，在应用程序显示之前，WindowManager 会等待这个 Window 绘制完毕
public static final int TYPE_DRAWN_APPLICATION = 4;

// 应用程序 Window 的结束值
public static final int LAST_APPLICATION_WINDOW = 99;
```

应用程序窗口：应用程序窗口一般位于最底层，type 在 1-99 区间。

```java
// 子 Window 类型的开始值
public static final int FIRST_SUB_WINDOW = 1000;

// 应用程序 Window 顶部的面板。这些 Window 出现在其附加 Window 的顶部。
public static final int TYPE_APPLICATION_PANEL = FIRST_SUB_WINDOW;

// 用于显示媒体(如视频)的 Window。这些 Window 出现在其附加 Window 的后面。
public static final int TYPE_APPLICATION_MEDIA = FIRST_SUB_WINDOW + 1;

// 应用程序 Window 顶部的子面板。这些 Window 出现在其附加 Window 和任何Window的顶部
public static final int TYPE_APPLICATION_SUB_PANEL = FIRST_SUB_WINDOW + 2;

// 当前Window的布局和顶级Window布局相同时，不能作为子代的容器
public static final int TYPE_APPLICATION_ATTACHED_DIALOG = FIRST_SUB_WINDOW + 3;

// 用显示媒体 Window 覆盖顶部的 Window， 这是系统隐藏的 API
public static final int TYPE_APPLICATION_MEDIA_OVERLAY  = FIRST_SUB_WINDOW + 4;

// 子面板在应用程序Window的顶部，这些Window显示在其附加Window的顶部， 这是系统隐藏的 API
public static final int TYPE_APPLICATION_ABOVE_SUB_PANEL = FIRST_SUB_WINDOW + 5;

// 子 Window 类型的结束值
public static final int LAST_SUB_WINDOW = 1999;
```

子窗口：子窗口一般是显示在应用窗口之上，type 在 1000-1999 区间。

```java
// 系统Window类型的开始值
public static final int FIRST_SYSTEM_WINDOW = 2000;

// 系统状态栏，只能有一个状态栏，它被放置在屏幕的顶部，所有其他窗口都向下移动
public static final int TYPE_STATUS_BAR = FIRST_SYSTEM_WINDOW;

// 系统搜索窗口，只能有一个搜索栏，它被放置在屏幕的顶部
public static final int TYPE_SEARCH_BAR = FIRST_SYSTEM_WINDOW+1;

// 已经从系统中被移除，可以使用 TYPE_KEYGUARD_DIALOG 代替
public static final int TYPE_KEYGUARD = FIRST_SYSTEM_WINDOW+4;

// 系统对话框窗口
public static final int TYPE_SYSTEM_DIALOG = FIRST_SYSTEM_WINDOW+8;

// 锁屏时显示的对话框
public static final int TYPE_KEYGUARD_DIALOG = FIRST_SYSTEM_WINDOW+9;

// 输入法窗口，位于普通 UI 之上，应用程序可重新布局以免被此窗口覆盖
public static final int TYPE_INPUT_METHOD = FIRST_SYSTEM_WINDOW+11;

// 输入法对话框，显示于当前输入法窗口之上
public static final int TYPE_INPUT_METHOD_DIALOG= FIRST_SYSTEM_WINDOW+12;

// 墙纸
public static final int TYPE_WALLPAPER = FIRST_SYSTEM_WINDOW+13;

// 状态栏的滑动面板
public static final int TYPE_STATUS_BAR_PANEL = FIRST_SYSTEM_WINDOW+14;

// 应用程序叠加窗口显示在所有窗口之上
public static final int TYPE_APPLICATION_OVERLAY = FIRST_SYSTEM_WINDOW + 38;

// 系统Window类型的结束值
public static final int LAST_SYSTEM_WINDOW = 2999;
```

系统级窗口：系统级窗口一般位于最顶层，不会被其他的 window 遮住，如 Toast，type 在2000-2999 区间。如果要弹出自定义系统级窗口需要动态申请权限。

### flags 属性

flag 属性控制的范围包括各种情景下的显示逻辑（锁屏，游戏等）和触控事件的处理逻辑。

```java
// 当 Window 可见时允许锁屏
public static final int FLAG_ALLOW_LOCK_WHILE_SCREEN_ON = 0x00000001;

// Window 后面的内容都变暗
public static final int FLAG_DIM_BEHIND = 0x00000002;

// Window 不能获得输入焦点，即不接受任何按键或按钮事件，例如该 Window 上 有 EditView，点击 EditView 是 不会弹出软键盘的
// Window 范围外的事件依旧为原窗口处理；例如点击该窗口外的view，依然会有响应。另外只要设置了此Flag，都将会启用FLAG_NOT_TOUCH_MODAL
public static final int FLAG_NOT_FOCUSABLE = 0x00000008;

// 设置了该 Flag,将 Window 之外的按键事件发送给后面的 Window 处理, 而自己只会处理 Window 区域内的触摸事件
// Window 之外的 view 也是可以响应 touch 事件。
public static final int FLAG_NOT_TOUCH_MODAL  = 0x00000020;

// 设置了该Flag，表示该 Window 将不会接受任何 touch 事件，例如点击该 Window 不会有响应，只会传给下面有聚焦的窗口。
public static final int FLAG_NOT_TOUCHABLE      = 0x00000010;

// 只要 Window 可见时屏幕就会一直亮着
public static final int FLAG_KEEP_SCREEN_ON     = 0x00000080;

// 允许 Window 占满整个屏幕
public static final int FLAG_LAYOUT_IN_SCREEN   = 0x00000100;

// 允许 Window 超过屏幕之外
public static final int FLAG_LAYOUT_NO_LIMITS   = 0x00000200;

// 全屏显示，隐藏所有的 Window 装饰，比如在游戏、播放器中的全屏显示
public static final int FLAG_FULLSCREEN      = 0x00000400;

// 表示比FLAG_FULLSCREEN低一级，会显示状态栏
public static final int FLAG_FORCE_NOT_FULLSCREEN   = 0x00000800;

// 当用户的脸贴近屏幕时（比如打电话），不会去响应此事件
public static final int FLAG_IGNORE_CHEEK_PRESSES    = 0x00008000;

// 则当按键动作发生在 Window 之外时，将接收到一个MotionEvent.ACTION_OUTSIDE事件。
public static final int FLAG_WATCH_OUTSIDE_TOUCH = 0x00040000;

@Deprecated
// 窗口可以在锁屏的 Window 之上显示, 使用 Activity#setShowWhenLocked(boolean) 方法代替
public static final int FLAG_SHOW_WHEN_LOCKED = 0x00080000;

// 表示负责绘制系统栏背景。如果设置，系统栏将以透明背景绘制，
// 此 Window 中的相应区域将填充 Window＃getStatusBarColor（）和 Window＃getNavigationBarColor（）中指定的颜色。
public static final int FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS = 0x80000000;

// 表示要求系统壁纸显示在该 Window 后面，Window 表面必须是半透明的，才能真正看到它背后的壁纸
public static final int FLAG_SHOW_WALLPAPER = 0x00100000;
```

### 其他属性

x 与 y 属性：指定 window 的位置

alpha：window 的透明度

gravity：window 在屏幕中的位置，使用的是 Gravity 类的常量

format：window 的像素点格式，值定义在 PixelFormat 中

## Window 添加 View 过程

Window 添加 View 都是通过 WindowManager 的 addView 方法来添加的，WindowManager 的实现类是 WindowManagerImpl，也就是说 WindowManagerImpl 的作用就是封装添加 Window（View树）的过程。

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
            mContext.getUserId());
}
```

mGlobal 是一个 WindowManagerGlobal 类型的全局单例，WindowManagerImpl 的逻辑利用桥接模式，都是由 mGlobal 来处理的。

```java
private final ArrayList<View> mViews = new ArrayList<View>();
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
private final ArrayList<WindowManager.LayoutParams> mParams =
        new ArrayList<WindowManager.LayoutParams>();

public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    // 首先判断参数是否合法
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (display == null) {
        throw new IllegalArgumentException("display must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    // 如果不是子窗口，会对其做参数的调整
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams);
    } else {
        final Context context = view.getContext();
        if (context != null
                && (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }

    synchronized (mLock) {
        ...
        // 这里新建了一个viewRootImpl，并设置参数
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);

        // 添加到windowManagerGlobal的三个重要list中
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        // 最后通过viewRootImpl来添加window
        try {
            root.setView(view, wparams, panelParentView);
        }
        ...
    }
}
```

WindowManagerGlobal 中有三个 List，每次 addView 时创建的对象都会保存在这里，之后对 Window（View树） 的一些操作就可以直接来取对象了。当 Window（View树） 被删除的时候，这些对象也会被从 List 中移除。

```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        ...
        try {
            mOrigWindowType = mWindowAttributes.type;
            mAttachInfo.mRecomputeGlobalAttributes = true;
            collectViewAttributes();
            // 这里调用了windowSession的方法，调用wms的方法，把添加window的逻辑交给wms
            res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                    mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                    mTempInsets);
            setFrame(mTmpFrame);
        }
        ...
        // 将要添加的view的parent指向ViewRootImpl对象
        view.assignParent(this);
    }
}
```

addView 的逻辑最后交给了创建的 ViewRootImpl 对象 setView 方法，先通过 WindowSession 的 addToDisplay 方法调用 WindowManagerService 添加 View，然后将要添加的view的 parent 指向自身，所以 ViewRootImpl 是 Window 和 View 之间的桥梁，可以处理两边的对象，然后联结起来。

mWindowSession 是一个 IWindowSession 对象，IWindowSession 是一个 IBinder 接口，他的具体实现类在 WindowManagerService，通过这个 mWindowSession 就可以直接调用 WMS 的方法进行跨进程通信。

mAttachInfo 存储一些 View attach 到 Window 时候的一些信息，并保存了 ViewRootHandler 对象引用。ViewRootHandler  类是 Handler 的子类，内部负责处理一些 View 布局绘制时的逻辑。

```java
public ViewRootImpl(Context context, Display display) {
	...
	mWindowSession = WindowManagerGlobal.getWindowSession();
	mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
		context);
	...
}

// WindowManagerGlobal.java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                ...
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            ...
                        });
            }
            ...
        }
        return sWindowSession;
    }
}
```

mWindowSession 是在 ViewRootImpl 构造函数中创建的，看出来这个 mWindowSession 是一个单例，也就是整个应用的所有 ViewRootImpl 的 IWindowSession 都是同一个，也就是**一个应用只有一个 IWindowSession**。

对于 WMS 而言，他是服务于多个应用的，他在内部给每个应用 IWindowSession 分配了一些数据结构如 List，用于保存每个应用的每个  Window（View树） 以及对应的 ViewRootImpl 。这个 List 称之为 WindowState。

> 添加过程总结：
>
> 调用 WindowManagerGlobal.getWindowSession() 获取 IWindowSession 对象，然后利用 IWindowSession 对象创建 ViewRootImpl 对象，在 ViewRootImpl 内部通过 IWindowSession 对象访问 WMS 申请床架 Window，最后再通过 ViewRootImpl  绘制 View。

## 常见组件的 Window 创建流程

WindowManagerImpl 是管理 PhoneWindow 的，他们是同时出现的。因而有两种创建 Window（View树）的方式：

- 已经存在 PhoneWindow，直接通过 WindowManagerImpl 创建 Window（View树）
- PhoneWindow 尚未存在，先创建 PhoneWindow，再利用 WindowManagerImpl 来创建 Window（View树）

### PopupWindow

PopupWindow 是利用 WindowManager 往屏幕上添加 Window（View树），但 PopupWindow 是依附于 Activity 而存在的，当 Activity 未运行时，是无法弹出 PopupWindow 的，所以 PopupWindow 不能在 onCreate、onStart、onResume 方法中弹出。

```java
public PopupWindow(View contentView, int width, int height, boolean focusable) {
    if (contentView != null) {
        mContext = contentView.getContext();
        mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);
    }

    setContentView(contentView);
    setWidth(width);
    setHeight(height);
    setFocusable(focusable);
}
```

PopupWindow 有多个重载方法，但最终都会调用到这个有四个参数的方法。通过 contentView 得到的 Context 和 由此得到的 WindowManager 对象，其实就是 Activity 对应的对象。

```java
public void showAtLocation(View parent, int gravity, int x, int y) {
    mParentRootView = new WeakReference<>(parent.getRootView());
    showAtLocation(parent.getWindowToken(), gravity, x, y);
}

public void showAtLocation(IBinder token, int gravity, int x, int y) {
    // 如果contentView是空直接返回
    if (isShowing() || mContentView == null) {
        return;
    }
    TransitionManager.endTransitions(mDecorView);
    detachFromAnchor();
    mIsShowing = true;
    mIsDropdown = false;
    mGravity = gravity;
    // 得到WindowManager.LayoutParams对象
    final WindowManager.LayoutParams p = createPopupLayoutParams(token);
    // 做一些准备工作
    preparePopup(p);

    p.x = x;
    p.y = y;
    // 执行popupWindow显示工作
    invokePopup(p);
}

private void invokePopup(WindowManager.LayoutParams p) {
    ...
    // 调用windowManager添加window
    mWindowManager.addView(decorView, p);
    ...
}
```

最后通过 WindowManager 对象创建 Window（View树），添加到屏幕上。

### Dialog

Dialog 和 PopupWindow 本质区别是：Dialog 会新建一个 PhoneWindow，Dialog 的设计就是一个和 Activity 独立开来的 PhoneWindow，他会直接显示在 Activity 所有 view 的上面。

```java
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    ...
    // 获取windowManager
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    // 构造PhoneWindow
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    // 初始化PhoneWindow
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setOnWindowSwipeDismissedCallback(() -> {
        if (mCancelable) {
            cancel();
        }
    });
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);
    mListenersHandler = new ListenersHandler(this);
}

public void show() {
   ...
    // 回调onStart方法，获取前面初始化好的decorview
    onStart();
    mDecor = mWindow.getDecorView();
    ...
    WindowManager.LayoutParams l = mWindow.getAttributes();
    ...
    // 利用windowManager来添加window
    mWindowManager.addView(mDecor, l);
    ...
    mShowing = true;
    sendShowMessage();
}
```

## WindowManagerService 添加流程

在 ViewRootImpl 的 setView 方法中，会通过 IWindowSession 对象向 WindowManagerService 请求添加一个 Window（View树），addToDisplay() 跨进程调用到了 WindowManagerService.addWindow()。

```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    ...
    //mWindowSession是一个aidl，ViewRootImpl利用它来和WindowManagerService交互
    //mWindow是一个aidl，WindowManagerService可以利用这个对象与服务端交互
    //mAttachInfo可以理解为是一个data bean，可以跨进程传递
    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
    ...
}

// WindowManagerService.java
public int addWindow(Session session, IWindow client...) {
    ...
    //WindowState 用来描述一个 Window
    final WindowState win = new WindowState(this, session, client, token, parentWindow,
                appOp[0], seq, attrs, viewVisibility, session.mUid,
                session.mCanAddInternalSystemWindow);
    ...
    //会创建一 个SurfaceSession
    win.attach(); 
    //mWindowMap 是 WindowManagerService 用来保存当前所有 Window 新的的集合
    mWindowMap.put(client.asBinder(), win); 
    ...
    //一个token下会有多个win state。 其实token与PhoneWindow是一一对应的。
    win.mToken.addWindow(win); 
    ...
}
```

WindowManagerService 创建了一个 WindowState，用来表示客户端的一个 Window（View树），在 WindowState.attach() 方法中，创建了一个 SurfaceSession，SurfaceSession 会和 SurfaceFlinger 构建链接，并创建一个 SurefaceComposerClient 对象，一个应用程序只具有一个这个对象。SurefaceComposerClient 创建了一个 Client 对象，用于维护这应用程序的渲染核心数据，并负责与 SurfaceFlinger 通信。



[直面底层：这一次，彻底搞懂Android 中的Window (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650833731&idx=1&sn=161bdfb957e8ced264f20d55b47adb8d&chksm=80b757ddb7c0decb46ab72b01df363e0758b8d7f43b2bf010d6d2bb900560ecc2f25e66abbba)

[Android的UI显示原理之Surface的创建 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903761292886030)

[Android的UI显示原理之Surface的渲染 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903766219571214)

[Android UI 显示原理分析小结 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903769235275784)

