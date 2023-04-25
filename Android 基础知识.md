# Android 基础知识

## Activity

### 生命周期

Activity 在其生命周期中会经历多个状态。以下是 Activity 的完整生命周期及其相关的回调方法：

- onCreate()：当 Activity 被创建时调用。在这个阶段，你可以进行初始化操作，比如设置布局、初始化变量、绑定数据等
- onRestart()：当 Activity 从停止状态恢复为活跃状态时调用。这个方法在 onStart() 之前调用，用于恢复 onStop() 期间释放的资源
- onStart()：当 Activity 从不可见状态变为可见状态时调用。此时，Activity 还未显示在屏幕上，用户无法与其交互
- onNewIntent(Intent intent)：在 Activity 的实例已经存在于任务栈顶，并且再次通过 Intent 启动时，onNewIntent() 会在 onResume() 之前调用
- onRestoreInstanceState(Bundle savedInstanceState)：当 Activity 需要从先前保存的状态中恢复时，系统会在 onStart() 之后、onResume() 之前调用此方法
- onResume()：当 Activity 准备好与用户进行交互时调用。此时，Activity 已经完全呈现在屏幕上，用户可以与其进行交互
- onPause()：当系统即将暂停 Activity 时调用。此时，Activity 仍然可见，但可能已经失去了焦点。例如，当用户按下 Home 键或打开另一个 Activity 时，当前 Activity 会进入 onPause 状态。通常在这个阶段，你需要保存与 UI 相关的数据
- onStop()：当 Activity 完全不可见时调用。这可能是因为另一个 Activity 遮盖住了当前 Activity，或者当前 Activity 被销毁。在这个阶段，你可以释放不再需要的资源，如动画、监听器等
- onSaveInstanceState(Bundle outState)：在 Activity 被意外销毁（如因屏幕旋转、内存不足等原因）之前，系统会调用此方法来保存当前 Activity 的状态。通常在 onStop() 方法之后调用
- onDestroy()：当 Activity 被销毁时调用。这可能是因为系统需要回收资源，或者调用了 finish() 方法。在这个阶段，你需要释放所有持有的资源，如数据库连接、网络连接等。

### 启动模式

Activity 的启动模式决定了当一个新的 Activity 实例被启动时，系统如何处理任务栈。以下是 Android 中提供的四种启动模式：

- standard（标准模式）：这是默认的启动模式。当使用此模式启动 Activity 时，系统将在当前任务栈的顶部创建一个新的 Activity 实例，每次启动此 Activity 时都会创建一个新的实例，即使该 Activity 已经存在于任务栈中。
- singleTop（栈顶复用模式）：当使用此模式启动 Activity 时，如果当前任务栈的顶部已经是要启动的 Activity 类型，则不会创建新的实例，而是复用已有的实例。如果该 Activity 不在栈顶或栈中不存在该 Activity 类型，系统将在任务栈的顶部创建一个新的 Activity 实例。
- singleTask（栈内复用模式）：当使用此模式启动 Activity 时，系统会检查任务栈中是否已经存在该 Activity 的实例。如果存在，则将该实例调至栈顶，并清除它上面的所有 Activity。如果不存在，则在任务栈的顶部创建一个新的 Activity 实例。此模式只允许一个任务栈中存在一个 Activity 实例。
- singleInstance（单实例模式）：当使用此模式启动 Activity 时，系统将为此 Activity 创建一个新的任务栈，并将其放入新任务栈的栈顶。这个 Activity 在整个系统中将只有一个实例。当再次启动该 Activity 时，系统将切换到已经存在的实例所在的任务栈，并将其置于栈顶。

启动模式的有两种设置方法：静态设置和动态设置，动态设置的优先级要高于静态设置的：

- 静态设置： Manifest.xml中指定Activity启动模式

  ```xml
  <activity android:name=".MyActivity"
      android:launchMode="singleTop"/>
  ```

- 动态设置：Intent.addFlags() 指定启动模式

  ```java
  Intent intent = new Intent(); intent.setClass(context, MainActivity.class); intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
  context.startActivity(intent);
  ```

  常见的 Flag 有

  - FLAG_ACTIVITY_NEW_TASK 作用是为Activity指定 “SingleTask”启动模式，跟在AndroidMainfest.xml指定效果相同
  - FLAG_ACTIVITY_SINGLE_TOP 作用是为Activity指定 “SingleTop”启动模式，跟在AndroidMainfest.xml 指定效果相同
  - FLAG_ACTIVITY_CLEAN_TOP
    具有此标记位的Activity，启动时会将与该Activity在同一任务栈的其它Activity出栈，一般与SingleTask 启动模式一起出现。它会完成SingleTask的作用，但其实SingleTask启动模式默认具有此标记位的作用
  - 4.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：具有此标记位的Activity不会出现在历史Activity的列表中

## Fragment

### 生命周期

Fragment 的生命周期回调：

- onAttach(Context context)：当 Fragment 与它的宿主 Activity 关联时调用。在这个方法中，您可以获取到宿主 Activity 的 Context
- onCreate(Bundle savedInstanceState)：当 Fragment 第一次创建时调用。在这个方法中，您可以进行初始化操作，例如读取参数、恢复保存的状态等
- onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState)：当 Fragment 的视图需要创建时调用。在这个方法中，需要创建 Fragment 的布局，并返回创建的 View
- onViewCreated(View view, Bundle savedInstanceState)：当 Fragment 的视图已经创建好时调用。在这个方法中，您可以进行视图相关的初始化操作
- onActivityCreated(Bundle savedInstanceState)：当宿主 Activity 完成 onCreate() 方法后调用
- onStart()：当 Fragment 变得可见时调用
- onResume()：当 Fragment 变得可交互时调用
- onPause()：当 Fragment 不再可交互时调用
- onStop()：当 Fragment 不再可见时调用
- onDestroyView()：当 Fragment 的视图被销毁时调用。在这个方法中，您可以释放与视图相关的资源
- onDestroy()：当 Fragment 被销毁时调用。在这个方法中，您可以释放 Fragment 持有的资源
- onDetach()：当 Fragment 与宿主 Activity 解除关联时调用

### 添加到 Activity

Fragment 是通过 FragmentTransaction（由 FragmentManager 提供）被添加到 Activity 中的。通常，您需要先获取 FragmentManager 实例，然后使用它来开始一个 FragmentTransaction，接着调用 add 或 replace 方法，最后调用 commit 来完成事务。、

- add(int containerViewId, Fragment fragment, String tag)：将一个新的 Fragment 添加到 Activity 中。containerViewId 参数是一个布局容器的资源 ID，新的 Fragment 会被添加到这个容器中。fragment 参数是要添加的 Fragment 实例。tag 参数是一个可选的字符串，用于标识这个 Fragment，以便稍后通过 FragmentManager.findFragmentByTag(tag) 方法找到它。

  add 方法不会移除已经存在的 Fragment，它会在现有 Fragment 之上添加新的 Fragment。如果容器中已经有其他 Fragment，那么新添加的 Fragment 会覆盖在它们之上。

- replace(int containerViewId, Fragment fragment, String tag)：将一个新的 Fragment 替换掉 Activity 中的现有 Fragment。containerViewId 参数是一个布局容器的资源 ID，新的 Fragment 会被添加到这个容器中。fragment 参数是要替换的 Fragment 实例。tag 参数是一个可选的字符串，用于标识这个 Fragment，以便稍后通过 FragmentManager.findFragmentByTag(tag) 方法找到它。

  replace 方法会移除容器中所有现有的 Fragment，并将新的 Fragment 添加到容器中。如果容器中没有任何 Fragment，replace 方法的行为类似于 add 方法。

总结一下，add 和 replace 的主要区别在于：

add 方法在现有 Fragment 之上添加新的 Fragment，而不会移除已经存在的 Fragment。之后通过 show/hide 设置显示的 Fragment。
replace 方法会移除容器中所有现有的 Fragment，并将新的 Fragment 添加到容器中。

## Service

Service 的生命周期包括以下方法：

- onCreate(): 当 Service 首次创建时，系统会调用此方法。您可以在此方法中执行一次性的初始化操作，如创建线程、设置资源等。如果 Service 已经在运行，系统不会再次调用此方法
- onStartCommand(Intent intent, int flags, int startId): 每当客户端使用 startService() 方法启动 Service 时，系统都会调用此方法。服务是运行在主线程的，因此需要注意不要执行耗时操作，以免影响应用程序的性能
- onBind(Intent intent): 当客户端使用 bindService() 方法绑定到 Service 时，系统会调用此方法。在此方法中，需要返回一个 IBinder 接口，以便客户端与 Service 进行通信
- onUnbind(Intent intent): 当所有客户端取消绑定时，系统会调用此方法。在此方法中，可以执行适当的清理操作，如停止线程、释放资源等
- onRebind(Intent intent): 如果有客户端重新绑定到已经解除绑定的 Service，系统会调用此方法。默认情况下，onRebind 返回 void
- onDestroy(): 当服务不再需要并被销毁时，系统会调用此方法。在此方法中，您可以执行清理操作，如停止线程、释放资源等。

调用 startService() 启动 Service 会经历 onCreate() 和 onStartCommand()，当调用 stopSelf() 会调用 onDestroy()

调用 bindService() 绑定 Service 会经历 onCreate() 和 onBind()，当调用 ubbindService() 会调用 onUnbind() 和 onDestroy()

## BroadcastReceiver

广播分为两种基本类型：

- 标准广播：异步执行的广播，所有的广播接收器会在同一时间接收到
- 有序广播：同步执行的广播，优先级高的接收器先接收，可以调用 abortBroadcast() 打断

注册 BroadcastReceiver 分为动态注册和静态注册。

动态注册是指在代码中注册，步骤如下 ：

- 实例化自定义的广播接收器
- 创建 IntentFilte r实例，调用 addAction() 方法添加监听的广播类型
- 调用 Context.registerReceiver(BroadcastReceiver,IntentFilter) 动态的注册广播

静态注册是指在 AndroidManifest.xml 中添加

```
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>

<receiver android:name=".MyReceiver" 
android:enabled="true" 
android:exported="true">
	<intent-filter>
		<action android:name="android.intent.action.BOOT_COMPLETED"/>
	</intent-filter>
</receiver>
```

动态注册和静态注册的不同：

 动态注册的广播接收器可以自由的实现注册和取消，有很大的灵活性。但是只有在程序启动之后才能收 到广播，此外，不知道你注意到了没，广播接收器的注销是在onDestroy()方法中的。所以广播接收器的 生命周期是和当前Activity的生命周期一样。

静态注册的广播不受程序是否启动的约束，当应用程序关闭之后，还是可以接收到广播。

## Intent

Intent 在 Android 中用于在组件之间传递消息，通常用于启动活动（Activity）、服务（Service）或发送广播（Broadcast）。Intent 可以携带额外的数据，用于传递信息或参数，Intent 在传递数据时限制数据内容在 1MB 之内。

以下是一些常用的 Intent 方法：

### 构造方法

- Intent(): 创建一个空的 Intent。
- Intent(Intent o): 创建一个与给定原型 Intent 相同的新 Intent。
- Intent(String action): 创建一个具有指定操作的新 Intent。
- Intent(String action, Uri uri): 创建一个具有指定操作和数据（URI）的新 Intent。
- Intent(Context packageContext, Class<?> cls): 创建一个用于启动指定组件（如 Activity）的新 Intent。

### 设置操作（Action）

- Intent setAction(String action): 设置 Intent 的操作。

### 设置数据（Data）

- Intent setData(Uri data): 设置 Intent 的数据（URI）。
- Intent setDataAndType(Uri data, String type): 设置 Intent 的数据（URI）和 MIME 类型。
- Intent setType(String type): 设置 Intent 的 MIME 类型。

### 添加类别（Category）

- Intent addCategory(String category): 向 Intent 添加类别。

### 指定组件

- Intent setComponent(ComponentName component): 设置 Intent 的组件名称。
- Intent setClass(Context packageContext, Class<?> cls): 设置 Intent 的组件类。
- Intent setClassName(Context packageContext, String className): 设置 Intent 的组件类名称。
- Intent setClassName(String packageName, String className): 设置 Intent 的组件类名称和包名。

### 携带额外数据

- Intent putExtra(String name, boolean value): 添加布尔类型的额外数据。
- Intent putExtra(String name, byte value): 添加字节类型的额外数据。
- Intent putExtra(String name, char value): 添加字符类型的额外数据。
- Intent putExtra(String name, short value): 添加短整型类型的额外数据。
- Intent putExtra(String name, int value): 添加整型类型的额外数据。
- Intent putExtra(String name, long value): 添加长整型类型的额外数据。
- Intent putExtra(String name, float value): 添加浮点型类型的额外数据。
- Intent putExtra(String name, double value): 添加双精度类型的额外数据。
- Intent putExtra(String name, String value): 添加字符串类型的额外数据。
- Intent putExtra(String name, CharSequence value): 添加字符序列类型的额外数据。
- Intent putExtra(String name, Parcelable value): 添加可序列化（Parcelable）类型的额外数据。
- Intent putExtra(String name, Serializable value): 添加可序列化（Serializable）类型的额外数据。

### 获取额外数据：

- boolean getBooleanExtra(String name, boolean defaultValue): 获取布尔类型的额外数据。
- byte getByteExtra(String name, byte defaultValue): 获取字节类型的额外数据。
- char getCharExtra(String name, char defaultValue): 获取字符类型的额外数据
- short getShortExtra(String name, short defaultValue): 获取短整型类型的额外数据。
- int getIntExtra(String name, int defaultValue): 获取整型类型的额外数据。
- long getLongExtra(String name, long defaultValue): 获取长整型类型的额外数据。
- float getFloatExtra(String name, float defaultValue): 获取浮点型类型的额外数据。
- double getDoubleExtra(String name, double defaultValue): 获取双精度类型的额外数据。
- String getStringExtra(String name): 获取字符串类型的额外数据。
- CharSequence getCharSequenceExtra(String name): 获取字符序列类型的额外数据。
-  T getParcelableExtra(String name): 获取可序列化（Parcelable）类型的额外数据。
- T getSerializableExtra(String name): 获取可序列化（Serializable）类型的额外数据。

### 启动组件：

- void startActivity(Intent intent): 通过 Intent 启动一个 Activity。
- ComponentName startService(Intent service): 通过 Intent 启动一个 Service。
- boolean bindService(Intent service, ServiceConnection conn, int flags): 通过 Intent 绑定一个 Service。
- void sendBroadcast(Intent intent): 通过 Intent 发送一个广播。

### 标志（Flags）：

- Intent addFlags(int flags): 向 Intent 添加标志。
- Intent setFlags(int flags): 设置 Intent 的标志。
- int getFlags(): 获取 Intent 的标志。

## Context

Context 本身是一个抽象类，其主要实现类是 ContextImpl，还有一个子类是 ContextWrapper。ContextWrapper 有三个子类 Application、Service 和 ContextThemeWrapper。ContextThemeWrapper 的子类有 Activity。

ContextWrapper 就是 Context 类的修饰类。真正的实现类是 ContextImpl，ContextWrapper 里面的方法执行也是执行 ContextImpl 里面的方法。

### ContextImpl

ContextImpl 是 Android 框架中 Context 抽象类的具体实现类。它是用于访问应用程序环境的全局信息的接口。ContextImpl 提供了许多实用方法来访问资源、启动活动、发送广播等。以下是一些 ContextImpl 的常用方法：

- Resources getResources(): 获取应用程序的资源对象。
- AssetManager getAssets(): 获取应用程序的资产管理器。

- void startActivity(Intent intent): 通过 Intent 启动一个 Activity。
- ComponentName startService(Intent service): 通过 Intent 启动一个 Service。
- boolean bindService(Intent service, ServiceConnection conn, int flags): 通过 Intent 绑定一个 Service。
- void sendBroadcast(Intent intent): 通过 Intent 发送一个广播。

- <T> T getSystemService(String name): 获取指定名称的系统服务。

- File getFilesDir(): 获取应用程序的内部文件目录。
- File getCacheDir(): 获取应用程序的内部缓存目录。
- File getExternalFilesDir(String type): 获取应用程序的外部文件目录。

- SharedPreferences getSharedPreferences(String name, int mode): 获取指定名称的 SharedPreferences 对象。

### Application ContextImpl

Application 的 ContextImpl 是在 Application 对象创建时被赋值的。具体过程如下：

- 当应用程序启动时，系统会创建一个 Application 对象。在创建过程中，ActivityThread 类中的 handleBindApplication 方法会被调用
- 在 handleBindApplication 方法中，ActivityThread 会创建一个新的 ContextImpl 对象，并将其传递给 Application.attachBaseContext(Context base) 方法
- Application.attachBaseContext 方法将 ContextImpl 赋值给 Application 的成员变量 mBase，从而使 Application 可以访问 ContextImpl 提供的方法

### Service ContextImpl 

Service 的 ContextImpl 是在 Service 对象创建时被赋值的。具体过程如下：

- 当启动或绑定一个 Service 时，系统会创建一个 Service 对象
- 在创建过程中，ActivityThread 类中的 handleCreateService 方法中，ActivityThread 会创建一个新的 ContextImpl 对象，并将其传递给 Service.attachBaseContext(Context base) 方法
- Service.attachBaseContext 方法将 ContextImpl 赋值给 Service 的成员变量 mBase，从而使 Service 可以访问 ContextImpl 提供的方法

### Activity ContextImpl

Activity 的 ContextImpl 是在 Activity 对象创建时被赋值的。具体过程如下：

- 当启动一个 Activity 时，系统会创建一个 Activity 对象
- 在创建过程中，ActivityThread 类中的 performLaunchActivity 方法中，ActivityThread 会创建一个新的 ContextImpl 对象，并将其传递给 Activity.attachBaseContext(Context base) 方法
- Activity.attachBaseContext 方法将 ContextImpl 赋值给 Activity 的成员变量 mBase，从而使 Activity 可以访问 ContextImpl 提供的方法

## 持久化

### 存储空间

在 Android 系统中，存储空间主要分为内部存储空间和外部存储空间。

#### 内部存储空间

内部存储空间是每个 Android 应用程序都拥有的一个专用存储区域。这个存储区域只能被该应用程序访问，其他应用程序无法访问。当用户卸载应用程序时，内部存储空间中的所有数据都会被删除。

getFilesDir() 返回应用程序内部存储空间的私有文件目录的绝对路径，通常路径格式为：/data/data/<应用包名>/files

getCacheDir() 返回应用程序内部存储空间的缓存文件目录的绝对路径。通常路径格式为：/data/data/<应用包名>/cache

#### 外部存储空间

外部存储空间通常指的是可以被多个应用程序共享的存储区域，这个存储区域可以被其他应用程序访问。当用户卸载应用程序时，内部存储空间中的所有数据都会被删除。使用外部存储空间时，需要在 AndroidManifest.xml 文件中申请相应的权限。

getExternalFilesDir(String type) 返回应用程序外部存储空间的私有文件目录的绝对路径。参数 type 指定了子目录类型，例如 Environment.DIRECTORY_PICTURES。如果传入 null，则返回根目录。通常路径格式为： /storage/emulated/0/Android/data/<应用包名>/files/<类型>

getExternalCacheDir() 返回应用程序外部存储空间的缓存文件目录的绝对路径。通常路径格式为：
/storage/emulated/0/Android/data/<应用包名>/cache（设备内部存储作为外部存储）或
/storage/<SD卡名称>/Android/data/<应用包名>/cache（物理外部存储，如 microSD 卡）

### SharePerference

SharedPreferences 是 Android 提供的一个轻量级的数据存储方式，主要用于存储基本类型的数据，如字符串、整数、布尔值等。

获取 SharedPreferences 对象有两种常用方法：

- getSharedPreferences(String name, int mode) ：传入文件名和访问模式获取，这种方式获取的 SharedPreferences 对象可跨 Activity 共享
- getPreferences(int mode) ：在 Activity 类使用，以当前 Activity 的类名作为文件名，这种方式获取的 SharedPreferences 对象只在同一个 Activity 内共享

commit 和 apply 的区别：

- apply() 的修改是异步提交的，数据在后台写入文件。但没有返回值，无法获取提交结果
- commit() 的修改是同步提交的，返回值 Boolean 表示修改是否提交成功

## 定时任务

### Handler+postDelayed

通过在 Handler 中调用 postDelayed() 方法，可以在指定时间后执行一个 Runnable 对象。这种方法适用于简单的定时任务，尤其是 UI 线程中的任务。

```java
final Handler handler = new Handler();
handler.postDelayed(new Runnable() {
    @Override
    public void run() {
        // 在此处执行定时任务
        handler.postDelayed(this, 1000); // 每隔 1000 毫秒执行一次
    }
}, 1000);

```

### Timer 

通过 Timer 类和 TimerTask 类，可以实现定时任务，这种方法适用于非 UI 线程的任务。可以控制TimerTask的启动和取消，第一次执行任务时可以指定delay的时间。

Android 需要根据页面的生命周期和显隐来控制 Timer 的启动和取消，如果Timer调度的某个TimerTask抛出异常，Timer会停止所有任务的运行。

```java
Timer timer = new Timer();
timer.schedule(new TimerTask() {
    @Override
    public void run() {
        // 在此处执行定时任务
    }
}, 1000, 1000); // 延迟 1000 毫秒后执行，每隔 1000 毫秒执行一次
```

### ScheduledExecutorService

基于线程池的定时任务执行器，可以支持多个任务并发执行。

```java
ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
executorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        // 在此处执行定时任务
    }
}, 1000, 1000, TimeUnit.MILLISECONDS); // 延迟 1000 毫秒后执行，每隔 1000 毫秒执行一次
```

### AlarmManager

AlarmManager 类可以实现跨进程、系统级的定时任务，适用于长时间、不受应用生命周期影响的定时任务。

```java
AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
Intent intent = new Intent(this, MyReceiver.class);
PendingIntent pendingIntent = PendingIntent.getBroadcast(this, 0, intent, 0);
alarmManager.setRepeating(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() + 1000, 1000, pendingIntent);
```

setRepeating() 第一个参数表示闹钟类型，有以下四种类型：

- ELAPSED_REALTIME：使用自系统启动以来的时间（不包括休眠时间）作为基准。当闹钟触发时，不会唤醒设备，闹钟会在设备下次唤醒时触发
- ELAPSED_REALTIME_WAKEUP：使用自系统启动以来的时间（不包括休眠时间）作为基准。当闹钟触发时，它会唤醒设备执行相关任务
- RTC：使用设备的实时时钟作为基准，基于绝对时间。可以通过修改系统时间触发。当闹钟触发时，不会唤醒设备，闹钟会在设备下次唤醒时触发
- RTC_WAKEUP：使用设备的实时时钟作为基准，基于绝对时间。可以通过修改系统时间触发。当闹钟触发时，它会唤醒设备执行相关任务

AlarmManager 在系统休眠后仍然会启动。但是，从 Android 6.0（API 级别 23）开始，系统引入了 Doze 模式和应用待机模式，以优化设备的电池使用。当设备进入 Doze 模式时，系统会限制应用的网络访问和 CPU 使用，从而影响到 AlarmManager 的行为。在 Doze 模式下，有两种类型的 AlarmManager 闹钟：

- 在 Doze 模式下，普通闹钟（使用 set()、setRepeating() 等方法设置的闹钟）将被推迟，直到下一个维护窗口（系统允许应用访问网络和 CPU 的时间段）才会触发。
- 使用 setExact() 和 setExactAndAllowWhileIdle() 设置的闹钟会在指定时间准确触发。setExactAndAllowWhileIdle() 设置的闹钟在触发后，系统会立即返回到低功耗状态，因此需要确保闹钟任务的执行时间尽量短。

## APK 编译流程

Android APK 编译流程包括以下几个主要步骤：

- 使用 AAPT 打包资源文件，生成 R.java 文件
- 处理 AIDL 文件生成 Java 代码
- 编译 Java 代码生成 class 文件
- 使用 dx 或者 d8 工具将 class文件转为 dex 文件
- 将 dex 文件，资源文件，Manifest 文件打包到一个 APK 文件中
- 对 APK 文件进行对齐优化，确保 APK 文件中的资源以最佳方式对齐，从而提高应用运行时的性能
- 为 APK 文件添加数字签名，确保应用的完整性和来源