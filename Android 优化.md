# Android 优化

## ANR

ANR全称是Applicatipon No Response，是系统通过与之交互的组件以及用户交互进行超时监控，用来判断应用进程是否存在卡死或响应过慢的问题。

ANR 触发分为两种，一种是 Service、Broadcast、ContentProvider 执行超时，另一种是 InputDispatch 超时。

前台 Service 在 20s 内未执行完成，后台 Service 在 200s 内没有执行完成

前台广播在 10s 内未执行完成，后台广播在 60s 内未执行完成。

ContentProvider publish 在 10s 内没有完成

输入事件在 5s 内没有完成

```
static final int SERVICE_TIMEOUT = 20*1000;
static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;

//com.android.server.am.ActiveServices.java
private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    ......
    //发个延迟消息给AMS的Handler
    bumpServiceExecutingLocked(r, execInFg, "create");

    ......
    try {
        //IPC通知app进程启动Service，执行handleCreateService
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                app.getReportedProcState());
    } catch (DeadObjectException e) {
    } finally {
    }
}

private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
    scheduleServiceTimeoutLocked(r.app);
    .....
}

void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    //mAm是AMS，mHandler是AMS里面的一个Handler
    Message msg = mAm.mHandler.obtainMessage(
            ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    //发个延迟消息给AMS里面的一个Handler
    mAm.mHandler.sendMessageDelayed(msg,
            proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

在 ApplicationThread.scheduleCreateService() 通知 AMS 启动完成，移除延时操作

```java
private void handleCreateService(CreateServiceData data) {
    ......
    Service service = null;
    try {
        //1. 初始化Service
        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = packageInfo.getAppFactory()
                .instantiateService(cl, data.info.name, data.intent);
        ......
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManager.getService());
        //2. Service执行onCreate，启动完成
        service.onCreate();
        mServices.put(data.token, service);
        try {
            //3. Service启动完成，需要通知AMS
            ActivityManager.getService().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
        }
    } catch (Exception e) {
    }
}

private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
        boolean finishing) {
    ......
    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
    ......
}
```

## 冷启动优化

冷启动指的是应用首次启动或者从系统进程中被杀死后重新启动的过程，以下是一些可以优化 Android 冷启动的方法。

### 减少应用启动时间

- 减少 Application 类中 onCreate 方法的执行时间。尽量避免在 onCreate 方法中执行耗时的操作。
- 使用懒加载策略。将一些非必要的初始化操作推迟到真正需要的时候进行。
- 使用多线程并行处理耗时任务。在启动过程中使用异步任务或者线程池处理耗时任务。
- 将耗时初始化任务放置到 Idle 中

### 优化布局

- 减少布局层次。使用 ConstraintLayout 等高性能布局，尽量避免过多的布局嵌套。
- 使用 include 和 merge 标签重用布局。

### 优化资源加载

- 减少启动时加载的资源文件数量。只加载必要的资源文件。
- 使用正确的图片格式和压缩算法。例如，使用 WebP 格式替代 PNG 和 JPEG 格式。
- 使用资源分离。针对不同屏幕尺寸和密度提供适当的资源。

### 使用启动画面

在应用启动时，立即展示一个启动画面，通常使用一个静态的图像。这样可以在加载资源和初始化过程中提供更好的用户体验。

## UI 优化

### 层级优化

- 避免复杂的 View 层级
- 避免 layout 顶层使用 RelativeLayout
- 布局层次相同的情况下，使用 LinearLayout
- 复杂布局建议采用 RelativeLayout 而不是多层次的 LinearLayout
- \<include/> 标签复用
- \<merge/> 标签减少嵌套
- 尽量避免 layout_weight
- 视图按需加载或者使用 ViewStub

### 绘制优化

- 去除重复或者不必要的 background
- 点击态中的 normal 尽量设置成 transparent
- 去除 window 中的 background（这个可以通过处理 decorView 或者设置 Theme 的方式）
- 若是自定义控件的话，通过 canvas.clipRect() 帮助系统识别那些可见的区域
- onDraw 中不要创建新的局部对象和执行耗时操作