# 从 Activity 的启动过程学习 ActivityManagerService 

AMS 所提供的主要功能包括：

- Activity 调度
- 内存管理
- 进程管理

## Activity 调度机制

AMS 定义了几个重要的数据类，分别用来保存进程（Process）、活动（Activity）和任务（Task）

### 进程数据类 ProcessRecord

#### 进程文件信息

与该进程对应的 APK 文件的内部信息，比如：

- ApplicationInfo info
- String processName
- HashSet<String> pkgList  // 保存该进程中所有的 APK 文件包名

#### 进程的内存信息

用于 Linux 系统的 Out Of Memory 情况的处理

#### 进程中包含的 Context

- ArrayList activities
- ArrayList services
- ...

### 活动数据类 ActivityRecord

#### 环境信息

Activity 的工作环境，比如：所在的进程名称、文件路径、图标、主题等

#### 运行状态信息

比如：idle、stop、finishing 等

### 任务数据类 TaskRecord

ActivityRecord 中包含了一个 int task 变量，保存该 Activity 属于哪个 Task

### 重要调度相关变量

#### MAX_ACTIVITIES

后台的 Activity 最多保存数量

#### MAX_RECENT_TASKS

记录最近启动的 Activity 的数量

#### PAUSE_TIMEOUT

AMS 通知 Activity onPause 的超时时间，超时后悔强制关闭该 Activity

#### LAUNCH_TIMEOUT

AMS 启动 Activity 的超时时间，超时后放弃启动

#### ArrayList mHistory

保存正在运行的 Activity

#### ArrayList mLRUActivities

保存最近启动过的 Activity

#### ArrayList mStoppingActivities

当某个 Activity 已经运行，AMS 启动另一个 Activity 会先暂停第一个 Activity，并保存到 List 中，等到第二个 Activity 启动并处于空闲时，再从 List 中取出并停止第一个 Activity

#### ArrayList mFinishingActivities

某个 Activity 处于 finish 状态，不会立即杀死，而是会保存到 List，直到超过系统设定的警戒线后，才回收该 Activity

#### ActivityRecord mPausingActivity

正在暂停的 Activity，该变量只在暂停某个 Activity 时才有值，代表正在暂停的 Activity

#### ActivityRecord mResumedActivity

正在运行的 Activity

#### ActivityRecord mFocusedActivity

AMS 通知 WMS 应该和用户交互的 Activity

#### ActivityRecord mLastPausedActivity

上一次暂停的 Activity

### 重要相关类

ActivityRecord

Instrumentation

ActivityThread.ApplicationThread