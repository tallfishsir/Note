常见的跨进程方案有哪些，为什么有 Binder？

管道、消息队列、共享内存、Socket

从性能角度来看， Socket 传输效率低，主要用于跨网络的进程间通信；消息队列和管道采用的是存储-转发方式，数据需要先从发送方缓存区拷贝到内核缓存区，再从内核缓存区拷贝到接收方缓存区；共享内容虽然不需要拷贝，但控制复杂。Binder 利用内存映射，只需要拷贝一次数据，通过 操作系统 内存映射的方法mmap() 将用户空间的一块内存区域映射到内核空间，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。性能上仅次于共享内存。

稳定性上，Binder 采用 client-server 架构，职责明确，架构清晰，稳定性好

安全性上，每个 app 在内核状态分配 UID。

Binder IPC 实现原理？

- Binder 驱动在内核空间创建一个内核数据接收缓存区和内核缓存区
- Binder 驱动建立 内核缓存区和内核数据接收缓存区之间的映射，内核数据接收缓存区和接收进程用户空间映射
- 发送方进程通过系统调用 copy_from_user 将数据 copy 到内核缓存区，由于之前的映射关系，便完成了一次进程间通信。

Binder 通信模型？

Binder 是基于 Client-Server 架构的，包括 Client、Server、ServiceManager、Binder驱动，其中 Client、Server、ServiceManager 运行在用户空间，Binder驱动运行在内核空间。

- Binder 驱动：负责进程之间 Binder 通信的建立，Binder 在进程之间的传递，数据包在进程之间的传递等
- ServiceManager：内部有一张表来记录当前系统 Binder 实体，使得 Client 可以通过 Binder 的名字获得一个 Binder 实体的引用。一般流程是：Server 创建一个 Binder 实体，并起一个字符形式，然后将这个 BInder 实体连同名字一起以数据包的形式通过 Binder 驱动发送给 ServiceManager ，通知 ServiceManager 注册一个的 Binder 且这个 Binder 位于某个 Server 中。驱动为这个穿越进程边界的 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用。
- Server：Server 向 ServiceManager 进行注册的过程也涉及到进程间通信，所以 ServiceManager 的 Binder 是在 Binder驱动自动创建的，这个 Binder 实体的引用在所有 Client 中都固定为0。、
- Client：Server 向 ServerManager 注册 Binder 后，Client 就可以通过名字获取到 Binder 引用。从面向对象的角度看，Server 中的 Binder 实体有两个引用：一个位于 ServiceManager 中，一个位于发起请求的 Client 中。

Binder 通信过程：

- 一个进程使用 BINDER_SET_CONTEXT_MGR 命令通过 Binder 驱动将自己注册为 ServiceManager
- Server 通过 Binder驱动向 ServiceManager 注册 Binder，表明可以对外提供服务，Binder为这个 Binder 创建位于内核中的实体节点以及对 Binder 实体的引用。然后将名字和新建的 Binder 引用传给 ServiceManager填入查找表中
- Client 通过名字在 Binder驱动下从 ServiceManager 中获取到 Binder 引用，利用这个 Binder引用实现通信

Binder 的完整定义

- 从进程间通信角度看，Binder 是一种进程间通信的机制
- 从 Server 进程的角度看，Binder 指的是 Server 中的 Binder 实体对象
- 从 Client 进程的角度看， Binder 指的是 Binder 的引用，是 Binder 实体对象的一个远程代理
- 从传输角度来看， Binder 是一个可以跨进程传输的对象， Binder驱动会对这个对象处理，自动完成代理对象和本地对象的转换