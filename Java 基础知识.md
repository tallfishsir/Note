# Java 基础知识

## JVM(Java Visual Machine)

### 运行时数据区

JVM 运行时会把所管理内存区域分为线程共享区和线程私有区

线程共享区包括两个部分：

- 方法区：用于存储被 JVM 加载的类信息、运行时常量池、静态变量等
- 堆区：JVM 分配对象实例的内存区域，字符串常量池从方法区移动到堆。堆内存分为年轻代（Young Generation）和老年代（Old Generation），年轻代又分为 Eden 区、Survivor 0 区和 Survivor 1 区。不同区域的对象会根据它们的生命周期经历不同的垃圾回收策略。

线程私有区包括三个部分：

- Java栈：用于存储执行方法时的局部变量表、操作数栈、动态链接、方法出口等信息
- 本地方法栈：用于存储执行本地方法时的局部变量表等
- 程序计数器：用于存储当前线程正在执行指令地址

### 类加载

#### 触发条件

JVM 运行并不是一次性加载所需要的全部类的，它是按需加载，也就是延迟加载，当以下几种情况发生时，Java 虚拟机会触发类加载过程：

第一种，执行 new、getstatic、putstatic 或 invokestatic 这四条字节码指令时，如果没有被加载和初始化，需要先触发类加载过程。这四条指令分别对应创建类的实例、访问类的静态变量、修改类的静态变量、调用类的静态方法。

```java
public class Test {
    public static void main(String[] args) {
        // 触发类加载过程
        MyClass obj = new MyClass(); // new
        int a = MyClass.staticVar; // getstatic
        MyClass.staticVar = 10; // putstatic
        MyClass.staticMethod(); // invokestatic
    }
}
```

第二种，使用 Class.forName() 或 ClassLoader.loadClass() 方法显式加载类时，会触发类加载过程。Class.forName() 方法除了加载类还会执行类的初始化，而 ClassLoader.loadClass() 只会加载类而不会初始化。

```java
public class Test {
    public static void main(String[] args) {
        try {
            // 触发类加载过程，并初始化
            Class<?> clazz1 = Class.forName("com.example.MyClass");
            // 触发类加载过程
            Class<?> clazz2 = Test.class.getClassLoader().loadClass("com.example.MyClass");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

第三种，当使用子类时，如果父类尚未加载和初始化，需要先触发父类的加载过程。

```java
public class Test {
    public static void main(String[] args) {
        // 触发子类的加载过程，同时也触发父类的加载过程
        SubClass obj = new SubClass();
    }
}

class SuperClass {
    static {
        System.out.println("SuperClass init");
    }
}

class SubClass extends SuperClass {
    static {
        System.out.println("SubClass init");
    }
}
```

第四种，虚拟机启动时，用户指定的执行主类。

#### 加载流程

类的加载过程主要分为五个阶段：加载、验证、准备、解析、初始化。

加载阶段主要是 JVM 寻找类的字节码并将其加载到内存中，然后创建对应的 Class 对象，主要完成了：

- 通过类的全限定名找到对应的字节码文件
- 将字节码文件数据读入内存并创建一个 Class 对象，用于封装类在方法区内的数据结构
- 在堆区中生成一个代表这个类的 Class 对象，作为方法区类数据的访问入口

验证阶段是确保加载进来的字节码文件符合 JVM 规范。包括以下的验证：

- 文件格式验证：验证字节码文件的格式是否符合 JVM 规范
- 元数据验证：验证类、字段、方法等元数据信息是否符合 Java 语言规范
- 字节码验证：验证字节码指令是否符合 JVM 规范，是否存在非法指令等
- 符号引用验证：验证符号引用是否可以正确解析到实际引用

准备阶段主要是为类的静态变量分配内存并设置数据类型的默认初始值

解析阶段主要是将类的符号引用替换为直接引用。符号引用是通过符号描述目标，比如类名，字段名。直接引用是直接指向目标内存地址的指针等。

初始化阶段主要是执行类的静态初始化代码，包括静态变量的赋值和静态代码块的执行

#### 类加载器

不管是 JVM 还是 Android 中的 Dalvik/ART 虚拟机，都是使用 ClassLoader 将字节码文件加载到内存中。不同的是， Java 的字节码文件是 class 文件，Android 的字节码文件是对 class 文件优化生成的 dex 文件。

JVM 运行过程中会存在多个 ClassLoader，其中内置了三个重要的 ClassLoader，分别是：

- 启动类加载器(Bootstrap ClassLoad)：类加载器层次中最顶层的类加载器，负责加载位于 \<JAVA_HOME>\lib 下的核心类库，由 C++ 实现，属于 JVM 的一部分，在 Java 代码中无法直接获取到它的引用

- 扩展类加载器(Extension ClassLoader)：由 Java 实现，负责加载位于 \<JAVA_HOME>\lib\ext 下的 Java 平台的扩展库

- 应用程序类加载器(Application ClassLoader)：也叫作系统类加载器，负责加载用户程序的类，可以通过 ClassLoader.getSystemClassLoader() 获取它的引用


Android 也提供了多种 ClassLoader，分别是：

- BootClassLoader：继承 ClassLoader，是一个单例类。访问修饰符是默认的，只有在同一个包中才可以访问，因此在应用程序中是无法直接使用的
- BaseDexClassLoader：PathClassLoader 和 DexClassLoader的父类，主要的执行逻辑和文件的处理都在其中
- PathClassLoader：负责加载系统类和应用程序的类，context.getClassLoader获取到的就是PathClassLoader实例
- DexClassLoader：加载dex文件以及包含dex的apk文件或jar文件，也支持从SD卡进行加载，这也意味着DexClassLoader可以在应用未安装的情况下加载dex相关文件

Android 中的 ClassLoader 是一个抽象类，定义了 ClassLoader 的主要功能，而 BaseDexClassLoader 则是 ClassLoader 的具体实现类，PathClassLoader 和 DexClassLoader 都继承它。ClassLoader 的几个重要方法：

- loadClass()：加载字节码文件时调用，遵循双亲委派模型
- findClass()：需要子类重写，完成把字节码文件加载到内存的过程
- defineClass()：在 findClass() 中调用，将读取到字节码文件生成的 byte[] 转为 Class 对象

```java
public class DexClassLoader extends BaseDexClassLoader {
	public DexClassLoader(String dexPath, String optimizedDirectory,
			String librarySearchPath, ClassLoader parent) {
		super(dexPath, null, librarySearchPath, parent);
	}
}

public class PathClassLoader extends BaseDexClassLoader {
	//parent是BootClassLoader实例
	public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
		super(dexPath, null, librarySearchPath, parent);
	}
}

class BootClassLoader extends ClassLoader {
	private static BootClassLoader instance;
	public static synchronized BootClassLoader getInstance() {
		if (instance == null) {
			instance = new BootClassLoader();
		}
		return instance;
	}
}

public class BaseDexClassLoader extends ClassLoader {
}

public abstract class ClassLoader {
	protected Class<?> loadClass(String name, boolean resolve)
		throws ClassNotFoundException {
			Class<?> c = findLoadedClass(name);
			if (c == null) {
				try {
					if (parent != null) {
						c = parent.loadClass(name, false);
					} else {
						c = findBootstrapClassOrNull(name);
					}
				} catch (ClassNotFoundException e) {
				}
				if (c == null) {
					c = findClass(name);
				}
			}
			return c;
	}
}
```

loadClass() 中遵循了双亲委派模型：

- findFromLoaded() 判断是否已经加载过类了
- 如果没有，调用 parent.loadClass() 让 parent ClassLoader 加载类
- 如果 parent ClassLoader 没有加载，调用 findClass() 加载类

双亲委托模型的好处：

- 先执行父类的查找，避免了同一个类的多次加载
- 先从父类查找的机制，使程序无法修改和替代Java基础包中的类，提高程序的安全性

如果需要自定义 ClassLoader，可以参考下面：

```java
public class FileSystemClassLoader extends ClassLoader {
    private String classPath;
    public FileSystemClassLoader(String classPath, ClassLoader parent) {
        super(parent);
        this.classPath = classPath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException("Class " + name + " not found");
        }
        return defineClass(name, classData, 0, classData.length);
    }

    private byte[] loadClassData(String className) {
        String fileName = className.replace('.', '/') + ".class";
        try (InputStream is = new FileInputStream(classPath + fileName);
             ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream()) {
            int bufferSize = 4096;
            byte[] buffer = new byte[bufferSize];
            int bytesRead;

            while ((bytesRead = is.read(buffer)) != -1) {
                byteArrayOutputStream.write(buffer, 0, bytesRead);
            }

            return byteArrayOutputStream.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

### 垃圾回收

Java 的垃圾回收（Garbage Collection, GC）机制是 Java 内存管理的重要组成部分。它负责自动回收不再被引用的对象所占用的内存，以避免内存泄漏和浪费。主要有两个阶段：确认哪些对象需要回收和怎么回收这些对象。

#### 标记算法

确认哪些对象回收有两种算法：引用计数算法和可达性算法。

引用计数算法是给对象添加一个引用计数器，当对象被引用的时候，计数器就加1，当引用失效的时候，计数器就减1。当计数器为 0 的对象就是可以被回收的。但这种算法无法有效的解决循环引用的问题。

可达性算法是先定义一系列 GC Root 对象作为起始点，然后搜索形成引用链，当一个对象不在引用链中，说明这个对象不可达，可以被回收。可以被定义为 GC Root 的对象有：

- 虚拟机栈中的本地变量表：这些变量包括局部变量、方法参数等
- 方法区中的类静态属性引用的对象
- 方法区中的常量引用的对象
- Java 虚拟机内部的引用，如基本类型对应的 Class 对象、常驻的异常对象
- 本地方法方法栈中 Native 引用的对象

#### 回收算法

被判定为需要回收的对象，会被第一次标记，并进行一次筛选，筛选条件是此对象有没有重写 finalize() 方法或有没有执行过。如果 finalize() 没有重写或已经执行过一次，就进行第二次标记进入回收集合。对已经进入回收集合的对象需要执行垃圾回收算法，常见的算法有：

- 标记-清理

  首先标记处所有需要回收的对象，然后统一回收被标记的对象。但这个方法效率不高，还会产生大量的内存碎片。

- 复制

  将内存分为大小相等的两块，每次只使用其中一块。当一块使用完，就把存活的对象复制到另外一块上，再把使用的一块空间全部清理。但这种方法内存使用率不高

- 标记-整理

  首先标记处所有需要回收的对象，然后让所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。

- 分代收集

  根据对象存活周期的不同，将内存划分为新生代、老生代。在新生代，使用复制算法，在老年代使用标记-清理或者标记整理算法。由于新生代使用了复制算法，所以新生代可以细分为 Eden 块和两个 Survivor 块

#### 引种类型

在引用计数算法或可达性算法中，都会有引用失效的情况。Java 提供了四种引用类型，用于满足不同场景下的内存管理需求。这四种引用类型分别是：强引用（Strong Reference）、软引用（Soft Reference）、弱引用（Weak Reference）和虚引用（Phantom Reference）。

强引用是 Java 中最常见的引用类型，当我们通过赋值操作创建一个对象引用时，默认创建的就是强引用。强引用指向的对象不会被垃圾回收器回收。

软引用可以用于实现内存敏感的缓存。当内存充足时，软引用指向的对象不会被回收；当内存不足时，软引用指向的对象可能会被回收，适合内存敏感的缓存。

```java
SoftReference<Bitmap> softRef = new SoftReference<>(BitmapFactory.decodeStream(new URL(imageUrl).openStream()));
Bitmap bitmmap = softRef.get();
if (bitmmap != null) {
	...
}
```

弱引用的生命周期比软引用更短。当垃圾回收器扫描到弱引用时，无论内存是否充足，都会回收弱引用指向的对象，用于实现短暂缓存、监听器等。

```java
WeakReference<Context> weakRef = new WeakReference<>(new Activity());
Context context = weakRef.get();
if (context != null) {
	...
}
```

虚引用的主要作用是跟踪对象被垃圾回收器回收的活动。虚引用无法获取到引用的对象，只能用于跟踪对象是否已被回收。

### 内存问题

#### 内存溢出

内存溢出是指程序运行过程中，需要分配的内存超过了可用内存上限，无法满足需求，发生 Out of Memory。内存溢出分为堆溢出和栈溢出，一般情况下，分别对应的是堆区和虚拟机栈。

堆溢出一般发生在：

- 创建过多对象，导致堆内存不足
- 大对象占用过多内存，如大数组或大文件缓存
- 频繁创建和销毁对象，导致内存碎片化，无法分配连续内存空间

栈溢出一般发生在：

- 递归调用过深，导致栈内存不足
- 内部逻辑出现死循环，导致栈内存不足

#### 内存泄漏

内存泄漏是指程序在运行过程中，无法释放已经不再使用的内存，导致该内存长时间占用，无法被其他对象使用。一般发生在：

- 未关闭的文件流、数据库连接或网络连接。

- 使用集合或缓存存储对象，但未从集合或缓存中删除不再使用的对象。

- 静态集合或缓存对象引用，导致对象无法被垃圾回收器回收。

- 监听器或回调函数未正确注销，导致长时间持有对象引用。

- 内部类、匿名内部类或 Lambda 表达式不正确地持有外部类的引用。

- 使用了不恰当的单例模式，导致单例对象长期占用内存。

## 多线程

进程和线程都是操作系统的基本概念，在多任务环境下负责管理和调度计算机资源。

进程是操作系统进行资源分配和调度的基本单位，它是一个独立运行的程序实例，每个进程都拥有独立的内存空间、地址空间和系统资源。进程之间的通信需要使用进程间通信（IPC）机制，如管道、消息队列、共享内存、信号量等。

线程是操作系统进行调度的最小单位，它是进程中的一个实例，一个进程中的多个线程共享进程的内存空间和资源，但各自拥有独立的运行堆栈和程序计数器。线程之间的通信相对简单，因为它们共享进程的资源。线程可以直接访问和修改共享资源，无需额外的通信机制。

### 线程属性

线程有编号、名字、类别以及优先级四个属性。

```java
class Thread implements Runnable {
	private long tid;
	private volatile String name;
    private boolean daemon = false;
    private int priority;
}
```

- tid：用于标识不同的线程，每条线程拥有不同的编号。线程运行结束后，线程的编号可能被后续创建的线程使用，因此编号不适合用作唯一标识
- name：名字的默认值是 Thread-线程编号
- daemon：线程的类别分为守护线程和用户线程。JVM 退出时，会保证所有的用户线程执行完毕，但不会考虑守护线程是否执行完成
- priority：用于表示应用希望优先运行哪个线程，线程调度器会根据这个值来决定优先运行哪个线程

### 线程状态

线程也有自己的生命周期，生命周期事件由开发者触发：

- NEW：线程创建后未启动时
- RUNNABLE：调用线程的 start() 方法后，等待系统分配 CPU 时间片
- RUNNING：线程获得 CPU 时间片开始执行 run() 任务
- Blocked：线程发起阻塞式 I/O 操作或者申请其他线程持有的锁
- Waiting：当线程调用如 wait()、join() 或 LockSupport.park() 等方法
- Timed Waiting：当线程调用如 sleep(long millis)、wait(long millis)、join(long millis) 或 LockSupport.parkNanos(long) 等带有超时参数的方法
- Terminated：线程的 run() 方法执行完毕，或者线程被中断、抛出异常而终止

![](https://mmbiz.qpic.cn/mmbiz_png/MOu2ZNAwZwPUugQRfibpGDCG8bmkCU8csJPca8ibsK0xIDy5iaLhfKWJyoVyZRwh9iaficewNdCnicUw0u3vgpuQhVLg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 线程安全

在一个进程内，方法区和堆区就是在主内存里，而虚拟机栈则在高速缓存区里，这也导致线程之间需要处理同步和互斥问题，以避免资源竞争和数据不一致，也就是线程安全。实现 Java 线程安全主要涉及三个要素：原子性，可见性和有序性。

可见性是指一个线程对共享变量的修改，能够及时地被其他线程感知到。可见性通常通过以下几种方式保证：

- 关键字 volatile：volatile 关键字可以确保一个线程对变量的修改对其他线程立刻可见
- 原子类(Atomic)：内部实际上也是通过 volatile 关键字保证可见性
- 关键字 synchronized：具有内存屏障的作用，确保锁定的变量对其他线程的修改在解锁时立即可见
- Lock 接口：提供了与 synchronized 类似的功能，但更灵活

原子性是指一个操作（可能含有多个子操作）要么全部执行，要么全部不执行。原子性通常是通过以下几种方式保证：

- 原子类(Atomic)，内部通过 CPU 级别的 CAS 机制，保证原子性
- 关键字 synchronized：通过锁定 monitor，确保同一时刻只有一个线程可以访问代码块
- Lock 接口：提供了与 synchronized 类似的功能，但更灵活

有序性是指程序执行的顺序按照代码的先后顺序进行。在多线程环境下，由于编译器优化和处理器优化，指令可能会发生重排序，导致代码执行顺序与预期不符。通常是通过以下几种方式保证：

- 关键字 volatile：阻止了编译器和处理器对其修饰的变量进行指令重排序
- 关键字 synchronized：防止指令重排序，确保代码块内的操作按照顺序执行
- Lock 接口：提供了与 synchronized 类似的功能，但更灵活

#### volatile

volatile 解决了多线程环境下变量的可见性和有序性问题，适用于以下场景：

- 变量在多个场景之间共享，且一个线程对变量的修改需要立即对其他线程可见
- 变量的读写操作不依赖其他状态变量，即不涉及复合操作

当一个变量被声明为 volatile 时，该对象就不会保存在线程的高速缓冲区，而是会被放到主内存里特意划分的内存屏障空间里。

内存屏障会强制刷新 CPU 缓存，使其他线程能够看到最新的值，并且内存屏障会禁止编译器和处理器对变量的读写操作进行重排序，确保它们的执行顺序与代码中的顺序相符。

#### Atomic

Atomic 类提供了基于硬件级别的原子指令实现了无锁的线程安全操作。内部持有代表值的 value 和 关联 value 地址的 valueOffset，value 使用 volatile 修饰，通过 CAS 机制改变量时。

```java
 public class AtomicInteger extends Number implements java.io.Serializable {
     ...
     private volatile int value;
 
     public AtomicInteger(int initialValue) {
         value = initialValue;
     }
 
     public AtomicInteger() { }
 
     private static final long valueOffset;
 
     static {
         try {
             valueOffset = unsafe.objectFieldOffset
                 (AtomicInteger.class.getDeclaredField("value"));
         } catch (Exception ex) { throw new Error(ex); }
     }
     
     public final boolean compareAndSet(int expect, int update) {
         return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
     }
 }
```

原子类包括以下几种：

- AtomicInteger: 提供原子整数操作
- AtomicLong: 提供原子长整数操作
- AtomicBoolean: 提供原子布尔操作
- AtomicReference: 提供原子对象引用操作
- AtomicIntegerArray/AtomicLongArray/AtomicReferenceArray: 提供原子数组操作

#### synchronized

Java 为我们提供了 synchronized 关键字来实现内部锁，被 synchronized 关键字修饰的方法和代码块就叫同步方法和同步代码块。内部锁的七个特点：

- 监视器锁：因为使用 synchronized 实现的线程同步是通过监视器（monitor）来实现的，所以内部锁也叫监视器锁。
- 自动获取/释放：线程对同步代码块的锁的申请和释放由 JVM 内部实施，线程在进入同步代码块前会自动获取锁，并在退出同步代码块时自动释放锁，这也是同步代码块被称为内部锁的原因。
- 锁定方法/类/对象：synchronized 修饰方法时，持有的 Monitor 是该方法所属的对象实例；synchronized 修饰静态方法时，持有的 Monitor 是该方法所属的 Class 对象；synchronized 修饰代码块时，需要显示传入 Monitor 对象
- 临界区：同步代码块就是内部锁的临界区，线程在执行临界区代码前必须持有该临界区的内部锁。
- 锁句柄：内部锁锁的对象就叫锁句柄，锁句柄通常会用 private 和 final 关键字进行修饰。因为锁句柄变量一旦改变，会导致执行同一个同步代码块的多个线程实际上用的是不同的锁。
- 不会泄漏：泄漏指的是锁泄漏，内部锁不会导致锁泄漏，因为 javac 编译器把同步代码块编译为字节码时，对临界区中可能抛出的异常做了特殊处理，这样临界区的代码出了异常也不会妨碍锁的释放。
- 非公平锁：内部锁是使用的是非公平策略，是非公平锁，也就是不会增加上下文切换开销。

在持有当前 Monitor 的线程执行完成之前，其他线程想要调用相关方法就会 block，直到持有当前 Monitor 的线程执行结束或者发生异常，释放 Monitor ，下一个线程才可获取 Monitor 执行。

```java
public class Test{
	//Monitor是Test对象实例
	public synchronized void Method1(){ 
    }
    
    //Monitor是Test对象实例，与方式1等价，也可以传入其他对象
	publi void Method2(){
        synchronized(this){
        }
    }
    
    //Monitor是Test.class
    public static synchronized void Method1(){
    }
}
```

在 synchronized 代码块中可以通过 Object 类的 wait()/notify()/notifyAll() 进行线程间通信，在同步代码块中，一个线程可以调用 wait() 方法等待条件满足，而另一个线程可以调用 notify() 或 notifyAll() 方法唤醒等待的线程。这三个方法的使用场景如下：

- wait(): 当前线程进入等待状态，释放锁，直到其他线程调用 notify() 或 notifyAll() 方法唤醒它。
- notify(): 唤醒在当前对象上等待的一个线程（具体哪个线程是不确定的）。
- notifyAll(): 唤醒在当前对象上等待的所有线程。

注意：调用 wait()、notify() 和 notifyAll() 方法时，线程必须持有当前对象的锁，否则会抛出 IllegalMonitorStateException 异常。

```java
import java.util.LinkedList;
import java.util.Queue;

public class Buffer {
    private final int maxSize;
    private final Queue<Integer> queue;

    public Buffer(int maxSize) {
        this.maxSize = maxSize;
        this.queue = new LinkedList<>();
    }

    public synchronized void put(Integer value) throws InterruptedException {
        while (queue.size() == maxSize) {
            wait();
        }
        queue.add(value);
        notifyAll();
    }

    public synchronized Integer take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        Integer value = queue.poll();
        notifyAll();
        return value;
    }
}

public class Producer implements Runnable {
    private final Buffer buffer;

    public Producer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 10; i++) {
                buffer.put(i);
                System.out.println("Produced: " + i);
                Thread.sleep((long) (Math.random() * 1000));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class Consumer implements Runnable {
    private final Buffer buffer;

    public Consumer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 10; i++) {
                int value = buffer.take();
                System.out.println("Consumed: " + value);
                Thread.sleep((long) (Math.random() * 1000));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class Main {
    public static void main(String[] args) {
        Buffer buffer = new Buffer(5);
        Producer producer = new Producer(buffer);
        Consumer consumer = new Consumer(buffer);

        new Thread(producer).start();
        new Thread(consumer).start();
    }
}
```

synchronized 关键字有一些缺陷：

- 无法判断锁状态，不知当前线程是否锁住了，从java层上来讲是无法知道线程是否锁住了该对象
- 可中断，如果线程一直获取不到锁，那么就会一直等待，直到占用锁的线程把锁释放
- synchronized 是非公平锁，新来的线程与等待的线程都有同样机会获得锁

#### ReentrantLock

显式锁是 Lock 接口的实例，Lock 接口对显式锁进行了抽象，ReentrantLock 是它的实现类，显式锁的四个特点：

- 可重入：式锁是可重入锁，也就是一个线程持有了锁后，能再次成功申请这个锁。
- 手动获取/释放：显式锁要自己释放和获取锁，为了避免锁泄漏，要在 finally 块中释放锁
- 临界区：lock() 与 unlock() 方法之间的代码就是显式锁的临界区
- 公平/非公平锁：显式锁允许选择锁调度策略

ReentrantLock 通过 Condition 的 await()/signal()/signalAll() 进行线程间通信，实现生产-消费模式。

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class Buffer {
    private final int maxSize;
    private final Queue<Integer> queue;
    private final ReentrantLock lock;
    private final Condition notFull;
    private final Condition notEmpty;

    public Buffer(int maxSize) {
        this.maxSize = maxSize;
        this.queue = new LinkedList<>();
        this.lock = new ReentrantLock();
        this.notFull = lock.newCondition();
        this.notEmpty = lock.newCondition();
    }

    public void put(Integer value) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == maxSize) {
                notFull.await();
            }
            queue.add(value);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Integer take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }
            Integer value = queue.poll();
            notFull.signal();
            return value;
        } finally {
            lock.unlock();
        }
    }
}

public class Producer implements Runnable {
    private final Buffer buffer;

    public Producer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 10; i++) {
                buffer.put(i);
                System.out.println("Produced: " + i);
                Thread.sleep((long) (Math.random() * 1000));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class Consumer implements Runnable {
    private final Buffer buffer;

    public Consumer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 10; i++) {
                int value = buffer.take();
                System.out.println("Consumed: " + value);
                Thread.sleep((long) (Math.random() * 1000));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class Main {
    public static void main(String[] args) {
        Buffer buffer = new Buffer(5);
        Producer producer = new Producer(buffer);
        Consumer consumer = new Consumer(buffer);

        new Thread(producer).start();
        new Thread(consumer).start();
    }
}
```

#### ReadWriteLock 

由于锁的排他性使得多个线程无法以线程安全的方式在同一时刻读取共享变量，这样不利于提高系统的并发性，这样就出现了 ReadWriteLock。ReadWriteLock 是 Java 并发包中的一个接口，用于解决多线程环境下读写操作的同步问题。它包含两个锁：一个用于读操作，另一个用于写操作，读写锁有下面特点：

- 读锁共享：读写锁允许多个线程同时读取共享变量，读线程访问共享变量时，必须持有对应的读锁，读锁可以被多个线程持有。
- 写锁排他：读写锁一次只允许一个线程更新共享变量，写线程访问共享变量时，必须持有对应的写锁，写锁在任一时刻只能被一个线程持有。
- 可以降级：读写锁是一个支持降级的可重入锁，也就是一个线程在持有写锁的情况下，可以继续获取对应的读锁。
- 不能升级：读写锁不支持升级，读线程只有释放了读锁才能申请写锁

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Counter {
    private int count = 0;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    public void increment() {
        lock.writeLock().lock();
        try {
            count++;
        } finally {
            lock.writeLock().unlock();
        }
    }

    public int getCount() {
        lock.readLock().lock();
        try {
            return count;
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

## 泛型

泛型的本质是为了参数化类型，在使用过程中，起到了检查类型、自动转型等作用。

### 泛型声明和实例化

根据泛型使用场景的不同，分为泛型类、泛型接口、泛型方法。

#### 泛型类型声明

泛型类和泛型接口允许在类的定义中指定若干个类型参数，这些参数可以在类的方法和属性中使用。在声明时，还可以使用关键字 extend 来限定类型参数的范围。

需要注意的是，类型参数不能修饰类中的静态方法，也不能创建对象

```java
//修饰符 class 类名<T>
//修饰符 interface 接口名<T>
public class Fruit<T extends String> {
    private T value;

    public T getValue() {
        return value;
    }

    public void setValue(T value) {
        this.value = value;
    }
}
```

#### 泛型类型实例化

泛型实例化是指在床架泛型类对象的时候，指定类型参数具体是什么类。需要注意的是，尽管 CharSequence 是 String 的父类，但 ArrayList\<CharSequence> 并不是 ArrayList\<String>的父类，这种特性被称为协变性。

- 协变性：如果 A 是 B 的父类，则 ArrayList\<A> 是 ArrayList\<B> 的父类
- 逆变性：如果 A 是 B 的子类，则 ArrayList\<A> 是 ArrayList\<B> 的父类

```java
ArrayList<String> list = new ArrayList<>();

//编辑器报错
ArrayList<CharSequence> charSequences = new ArrayList<String>();
ArrayList<String> charSequences = new ArrayList<CharSequence>();
```

如果想要泛型类拥有协变性，需要使用上边界通配符\<? extends E> ，代表类型参数是 E 的子类，但于此同时实例化的泛型类对象变成了只读，也就是说不能调用泛型参数作为参数的函数。

```java
ArrayList<? extends String> list = new ArrayList<>();
list.add(""); //编辑器报错
list.get(0); //正常
```

如果想要泛型类拥有逆变性，需要使用下边界通配符\<? super E>，代表类型参数是 E 的父类，但于此同时实例化的泛型类对象变成了只写，也就是说不能调用泛型参数作为返回值的函数。

```java
ArrayList<? super String> list = new ArrayList<CharSequence>();
list.add("123"); //正常
CharSequence c = list.get(0); //编辑器报错
```

#### 泛型方法声明

泛型方法允许在方法返回值前面指定若干个类型参数，这些参数可以作为方法的返回值、参数或者方法中使用。在声明时，还可以使用关键字 extend 来限定类型参数的范围。

```java
//修饰符 <T> 返回值 方法名( 形参... )
public <R extends String> R test(R r) {
    R temp = r;
    int length = temp.length();
    return temp;
}

public <T extends View> T findViewById(@IdRes int id) {
    return getDelegate().findViewById(id);
}
Button btn =  findViewById(R.id.second_activity_btn);
```

在调用泛型方法时，可以指定泛型，也可以不指定泛型:

- 在不指定泛型的情况下，泛型变量的类型为该方法中的几种类型的同一父类的最小级，直到Object
- 在指定泛型的情况下，该方法的几种类型必须是该泛型的实例的类型或者其子类

```java
//这两个参数都是Integer，所以T为Integer类型 
int i = Test.add(1, 2);  
//这两个参数一个是Integer，一个是Float，所以取同一父类的最小级，为Number  
Number f = Test.add(1, 1.2); 
//这两个参数一个是Integer，一个是String，所以取同一父类的最小级，为Object  
Object o = Test.add(1, "asd"); 

/**指定泛型的时候*/  
//指定了Integer，所以只能为Integer类型或者其子类
int a = Test.<Integer>add(1, 2);   、
//编译错误，指定了Integer，不能为Float 
int b = Test.<Integer>add(1, 2.2); 
//指定为Number，所以可以为Integer和Float  
Number c = Test.<Number>add(1, 2.2); 
```

### 类型擦除

泛型特性是在 JDK 1.5 版本引入，为了兼容之前的版本，Java 会在编译阶段进行类型擦除。就是把所有的泛型表示都替换为具体的类型，在对象进入和离开的方法里添加类型检查和类型转换方法，然后还会根据需要生成桥接方法：

- 类型参数没有限制或限定符是 <?>，被替换为 Object
- 类型参数是上边界通配符\<? extends E>，被替换为 E 的具体类型
- 类型参数是下边界通配符\<? super E>，被替换为 Object

由于类型擦除的存在，泛型类型是有一些限制的：

- 泛型类没有自己独有的 Class 类对象，并不存在 List\<String>.class 这样的类，而是只有 List.class
- 泛型类型不能显式的运用在运行时类型的操作中，比如：转型，instanceOf，new
- 泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数
- 不能抛出也不能捕获泛型类的对象

### 反射获取泛型类型

由于类型擦除，在运行时无法直接获取到完整的泛型信息。但可以通过反射来获取部分泛型信息。java.lang.reflect.Type 是 Java 中所有类型的公共高级接口，代表了 Java 中的所有类型，Type 体系中类型的包括：

- 数组类型(GenericArrayType)
- 参数化类型(ParameterizedType)
- 类型变量(TypeVariable)
- 通配符类型(WildcardType)
- 原始类型(Class)
- 基本类型(Class)

```java
public interface ParameterizedType extends Type {
    // 返回确切的泛型参数, 如Map<String, Integer>返回[String, Integer]
    Type[] getActualTypeArguments();
    
    //返回当前class或interface声明的类型, 如List<?>返回List
    Type getRawType();
    
    //返回所属类型. 如,当前类型为O<T>.I<S>, 则返回O<T>. 顶级类型将返回null 
    Type getOwnerType();
}
```

#### 获取类的泛型类型

虽然泛型类型在运行时被移除，无法直接从一个泛型类实例中获取到泛型信息。但可以通过子类或匿名内部类，泛型参数具体化（即指定了具体类型）时获取到泛型信息。

```java
public class MyClass<T> {
    public static void main(String[] args) {
        //注意结尾的{}，表示这是MyClass的匿名子类
        MyClass<String> myClass = new MyClass<String>(){};
        Type superclass = myClass.getClass().getGenericSuperclass();

        if (superclass instanceof ParameterizedType) {
            ParameterizedType parameterizedType = (ParameterizedType) superclass;
            
            Type[] typeArguments = parameterizedType.getActualTypeArguments();
            for (Type typeArgument : typeArguments) {
                //打印java.lang.String
                System.out.println("泛型类型：" + typeArgument.getTypeName());
            }
        }
    }
}
```

#### 获取字段的泛型类型

如果一个字段被声明为泛型类型，可以通过 Field.getGenericType() 获取到泛型类型信息。

```java
public class MyClass {
    public List<String> myList;

    public static void main(String[] args) throws NoSuchFieldException {
        Field field = MyClass.class.getDeclaredField("myList");
        Type genericType = field.getGenericType();
        
        if (genericType instanceof ParameterizedType) {
            ParameterizedType parameterizedType = (ParameterizedType) genericType;
            
            Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
            for (Type typeArgument : actualTypeArguments) {
                //打印java.lang.String
                System.out.println("泛型类型：" + typeArgument.getTypeName());
            }
        }
    }
}
```

#### 获取方法参数的泛型类型

如果一个方法的参数被声明为泛型类型，可以通过 Method.getGenericParameterTypes() 获取到泛型类型信息。

```java
public class MyClass {
    public void myMethod(List<String> myList) {
        // ...
    }

    public static void main(String[] args) throws NoSuchMethodException {
        Method method = MyClass.class.getDeclaredMethod("myMethod", List.class);
        Type[] genericParameterTypes = method.getGenericParameterTypes();
        
        for (Type genericParameterType : genericParameterTypes) {
            if (genericParameterType instanceof ParameterizedType) {
                ParameterizedType parameterizedType = (ParameterizedType) genericParameterType;
                
                Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
                for (Type typeArgument : actualTypeArguments) {
                	//打印java.lang.String
                    System.out.println("泛型类型：" + typeArgument.getTypeName());
                }
            }
        }
    }
}
```

## 注解

注解用于对代码进行说明，可以对包、类、接口、字段、方法参数、局部变量等进行附加说明。它主要的作用有以下四方面：

- 生成文档，通过代码里标识的元数据生成javadoc文档
- 编译检查，通过代码里标识的元数据让编译器在编译期间进行检查验证
- 编译时动态处理，编译时通过代码里标识的元数据动态处理，例如动态生成代码
- 运行时动态处理，运行时通过代码里标识的元数据动态处理，例如使用反射注入实例

### 元注解和自定义注解

#### 元注解

Java 元注解(Meta-Annotations) 是用于注解其他注解的注解，为注解提供了一些元数据，来描述注解的行为和使用方式等：

- @Retention：注解的保留策略，包括了三个取值：
  - RetentionPolicy.SOURCE：注解只保留在源码阶段，编译器编译时会被丢弃
  - RetentionPolicy.CLASS：注解只保留在编译阶段，不会被加载到 JVM 中
  - RetentionPolicy.RUNTIME：注解保留在运行时，会被加载到 JVM 中，在程序运行时可以获取到
- @Document：可以将注解中的元素添加到 javadoc 中
- @Target：限定了直接使用的场景，它的取值有：
  - ElementType.ANNOTATION_TYPE：可以给注解进行注解
  - ElementType.PACKAGE：可以给包进行注解
  - ElementType.TYPE：可以给一个类型进行注解
  - ElementType.CONSTRUCTOR：可以给构造函数进行注解
  - ElementType.FIELD：可以给属性进行注解
  - ElementType.METHOD：可以给方法进行注解
  - ElementType.LOCAL_VARIABLE：可以给局部变量进行注解
  - ElementType.PARAMETER：可以给方法的参数进行注解
- @Inherited：表示注解可以被子类继承，仅对类的注解有效

#### 自定义注解

创建一个自定义注解，需要按照以下步骤：

- 使用 @interface 关键字定义注解
- 添加属性，用于存储和传递额外的信息，类似定义接口中的方法，可以使用 default 设置默认值
- 添加元注解，设置自定义注解使用范围和保留策略等
- 使用 @自定义注解 应用在对于区域

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface MyCustomAnnotation {
    String value();
    int priority() default 1;
    String[] tags() default {};
}

@MyCustomAnnotation(value = "Class level", priority = 2, tags = {"tag1", "tag2"})
public class MyClass {
    @MyCustomAnnotation(value = "Method level", priority = 1)
    public void myMethod() {
        // ...
    }
}
```

### 解析注解

#### 运行时解析

RetentionPolicy.RUNTIME 修饰的注解在运行时可以获取设置的注解信息。AnnotatedElement 接口是所有程序元素（Class、Method 和 Constructor）的父接口，可以通过反射获取了某个 Annotated 对象：

- boolean isAnnotationPresent(Class<? extends Annotation> annotationClass)

  判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false

- \<T extends Annotation> T getAnnotation(Class\<T> annotationClass)

  返回元素上存在的指定类型的注解，如果该类型注解不存在，则返回null。此方法会忽略注解对应的注解容器

- \<T extends Annotation> T[] getAnnotationsByType(Class\<T> annotationClass)

  返回元素上存在的指定类型的注解数组，没有注解对应类型的注解时，返回长度为0的数组。与 getAnnotation() 的区别是如果类是可继承的，还会去父类上检测对应的注解

- Annotation[] getAnnotations()

  返回元素上存在的所有注解，若没有注解，返回长度为0的数组

- \<T extends Annotation> T getDeclaredAnnotation(Class\<T> annotationClass)

  返回直接存在于此元素上的所有注解，忽略继承的注解

- \<T extends Annotation> T[] getDeclaredAnnotationsByType(Class\<T> annotationClass)

  返回直接存在于此元素上的所有注解，忽略继承的注释

- Annotation[] getDeclaredAnnotations()

  返回直接存在于此元素上的所有注解及注解对应的重复注解容器，忽略继承的注解

然后就可以获取自定义注解对象的属性及设置的数值。

#### 编译时解析

RetentionPolicy.SOURCE 修饰的注解在编译时会被丢弃，所以需要在编译阶段使用注解处理器( Annotation Processor) 进行处理，有以下步骤：

- 创建自定义注解处理器类，继承 AbstractProcessor 

- 在路径「src/main/resources/META-INF/services/」下创建文件 javax.annotation.processing.Processor
- javax.annotation.processing.Processor 中添加自定义注解处理器的全限定路径
- 自定义注解处理器类的 process() 中处理逻辑，返回 boolean 表示注解是否被处理
- 自定义注解处理器类的 getSupportedAnnotationTypes() 返回包含支持的注解类型名称（全限定名）的字符串集合
- 自定义注解处理器类的 getSupportedSourceVersion() 返回处理器支持的 Java 源代码版本

```java
//src/main/resources/META-INF/services/javax.annotation.processing.Processor
com.example.MyAnnotationProcessor

package com.example;

public class MyAnnotationProcessor extends AbstractProcessor {

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        // 在处理器实例化后立即被调用，用于初始化处理器
        // 例如保存 processingEnv 供后续使用
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        for (TypeElement annotation : annotations) {
            //获取在当前处理轮次中使用了指定注解的元素集合，如如包、类或方法等
            Set<? extends Element> annotatedElements = roundEnv.getElementsAnnotatedWith(annotation);
            for (Element element : annotatedElements) {
                ...
            }
        }
        return true;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton("com.example.MyCustomAnnotation");
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
}
```

Element 接口表示程序元素，如包、类或方法。以下是 Element 接口中的一些常用方法：

- getKind(): 返回表示元素类型的 ElementKind 枚举值，例如类、接口、枚举、字段、方法等
- getModifiers(): 返回一个 Set，包含表示元素修饰符的 Modifier 枚举值，例如 public、private、static、final 等
- getSimpleName(): 返回一个 Name 对象，表示元素的简单名称（不包括包名或外部类名）。
- getQualifiedName(): 对于包和类等类型的元素，返回一个 Name 对象，表示元素的完全限定名称（包括包名和外部类名）
- getEnclosingElement(): 返回一个 Element 对象，表示直接封闭此元素的程序元素，例如类、方法或外部类等
- getEnclosedElements(): 返回一个 List，包含直接封闭在此元素中的程序元素，例如类中的字段、方法等。仅对于某些元素类型有效，如类、接口、枚举等
- getAnnotation(Class<A> annotationType): 返回此元素上指定类型的注解（如果存在），否则返回 null。A 是注解类型
- getAnnotationsByType(Class<A> annotationType): 返回此元素上指定类型的所有注解，结果是一个数组。A 是注解类型
- getAnnotationMirrors(): 返回表示此元素上所有注解的 AnnotationMirror 对象的列表
- asType(): 返回一个 TypeMirror 对象，表示此元素的类型。对于类、接口、字段等类型的元素有效
- accept(ElementVisitor<R, P> v, P p): 使用指定的访问者和参数访问此元素。这是访问者设计模式的一部分

## 反射

Java 反射允许在运行过程中访问、检查、修改自身的结构和行为，反射的主要作用如下：

- 动态创建对象
- 访问和修改类的私有属性和私有方法
- 获取类的元数据，比如类名，属性，方法，注解等信息
- 动态代理，不修改原始类的基础上，添加额外的功能

### Class

获取 Class 对象的方法有以下几种：

- Class.forName(String name)：调用 Class 的静态方法传入类的完全限定名，加载 Class 对象
- 类名.class：直接使用 类名.class 获取 Class 对象
- 对象.getClass()：使用对象的 getClass() 方法获取 Class 对象

Class.forName() 方法适用于动态加载类的场景，可能抛出 ClassNotFoundException；.class 语法适用于已知类型，编译时就可以确定，不会抛出异常；getClass() 方法适用于已有对象实例的情况。

#### 创建对象实例

Class.newIntance() 会调用类的无参构造函数创建类的新实例，这个方法可能抛出 InstantiationException 和 IllegalAccessException，需要处理这些异常。

#### 获取类名

Class 提供了几种获取不同的返回格式类名的方法：

- getName()：返回类的二进制名称，适用于JVM内部使用

  内部类：javaa.reflect.RefOuter$RefInner

  数组：[Ljava.lang.String;

- getSimple()：返回类的简单名称，适用于显示给用户

  内部类：RefInner

  数组：String[]

- getCanonicalName()：返回类的规范名称，适用于生成代码

  内部类：javaa.reflect.RefOuter.RefInner

  数组：java.lang.String[]

#### 判断类型

isArray()：判断是否是数组，如果是数组，可以使用 getComponentType() 获取数组中的元素类型。

isEnum()：判断是否是枚举，如果是枚举，可以使用 getEnumConstants() 获取所有的枚举常量。

isPrimitive()：判断是否是基本数据类型 (Boolean.TYPE, Character.TYPE, Byte.TYPE, Short.TYPE, Integer.TYPE, Long.TYPE, Float.TYPE, Double.TYPE, Void.TYPE)。

isInterface()：判断是否是接口。

isAnnotation()：判断是否是注解。

#### 获取元数据

Class<? super T> getSuperclass()：获取类的父类

Type getGenericSuperclass()：获取一个类的泛型父类

Class<?>[] getInterfaces()：获取类实现的接口

### Construct

Class 对象可以使用以下方法获取 Constructor 对象：

- Constructor<?>[] getConstructors()：获取类的所有 public 构造函数
- Constructor\<T> getConstructor(Class<?>... parameterTypes)：获取类的 public 构造函数，参数为构造函数的参数类型数组
- Constructor<?>[] getDeclaredConstructors()：获取类中声明的所有构造函数（包括私有构造函数）
- Constructor\<T> getDeclaredConstructor(Class<?>... parameterTypes)：获取类中声明的任意构造函数（包括私有构造函数），参数为构造函数的参数类型数组

由于 Java 的类型擦除机制，泛型参数在运行时被擦除。因此，如果构造函数包含泛型参数，需要使用通配符（如 ?）或原始类型来获取 Constructor 对象。

#### 创建对象实例

Constructor.newInstance(Object... initargs)：使用此构造函数对象创建类的新实例。参数是一个可变参数数组，包含用于调用构造函数的参数值。

#### 获取元数据

Class<?>[] getParameterTypes()：返回一个描述此构造函数的参数类型的 Class 对象数组

int getParameterCount()：返回描述此构造函数的参数个数

### Field

Class 对象可以使用以下方法获取 Field 对象：

- Field[] getFields()：获取类的所有 public 成员变量，包括从父类继承的 public 成员变量
- Field getField(String name)：获取类的指定名称的 public 成员变量，参数为成员变量的名称
- Field[] getDeclaredFields()：获取类中声明的所有成员变量（包括私有成员变量）
- Field getDeclaredField(String name)：获取类中声明的指定名称的成员变量（包括私有成员变量），参数为成员变量的名称

由于 Java 的类型擦除机制，泛型参数在运行时被擦除。因此，如果构造函数包含泛型参数，需要使用通配符（如 ?）或原始类型来获取 Field 对象。

#### 获取元数据

Object get(Object obj)：返回指定对象上此 Field 表示的字段的值。如果是静态字段，obj 参数可以为 null

set(Object obj, Object value)：将此 Field 表示的字段设置为指定值。如果是静态字段，obj 参数可以为 null

Class<?> getType()：返回一个 Class 对象，表示此 Field 表示的字段的声明类型

### Method

Class 对象可以使用以下方法获取 Method 对象：

- Method[] getMethods()：获取类的所有 public 方法，包括从父类继承的 public 方法
- Method getMethod(String name, Class<?>... parameterTypes)：获取类的指定名称和参数类型的 public 方法。参数包括方法的名称和可变参数数组，表示方法的参数类型
- Method[] getDeclaredMethods()：获取类中声明的所有方法（包括私有方法）
- Method getDeclaredMethod(String name, Class<?>... parameterTypes)：获取类中声明的指定名称和参数类型的方法（包括私有方法）。参数包括方法的名称和可变参数数组，表示方法的参数类型

由于 Java 的类型擦除机制，泛型参数在运行时被擦除。因此，如果构造函数包含泛型参数，需要使用通配符（如 ?）或原始类型来获取 Method 对象。

#### 调用方法

invoke(Object obj, Object... args)：在指定对象上调用此 Method 表示的方法。参数包括调用方法的对象实例和可变参数数组，表示方法的参数值。如果是静态方法，obj 参数可以为 null

#### 获取元数据

Class<?> getReturnType()：返回一个 Class 对象，表示此 Method 表示的方法的返回类型

Class<?>[] getParameterTypes()：返回一个 Class 对象数组，表示此 Method 表示的方法的参数类型

int getParameterCount()：返回此 Method 表示的方法的参数个数

Parameter[] getParameters()：返回此 Method 表示的方法的参数

### Parameter

 Method 和 Constructor 对象可以通过以下方法获取 Parameter 对象：

- Method.getParameters()：返回一个 Parameter 对象数组，表示 Method 对象表示的方法的参数
- Constructor.getParameters()：返回一个 Parameter 对象数组，表示 Constructor 对象表示的构造函数的参数

#### 获取元数据

String getName()：返回参数的名称。注意，参数名称在编译时可能被丢弃，除非使用 -parameters 编译选项。如果参数名称不可用，则返回一个合成名称，如 "arg0"、"arg1" 等

Class<?> getType()：返回一个 Class 对象，表示参数的类型

Type getParameterizedType()：返回一个 Type 对象，表示参数的声明类型。与 getType() 不同，此方法可以获取泛型参数的完整类型信息

### Modifier

Class，Construct，Method，FIeld，Parameter 对象都可以调用 getModifiers() 方法返回一个 int 对象，表示类的修饰符，这个整数的每个二进制位表示一个特定的修饰符，可以使用 Modifier 类中静态方法解析：

- isPublic(int mod)：判断修饰符是否包含 public。
- isPrivate(int mod)：判断修饰符是否包含 private。
- isProtected(int mod)：判断修饰符是否包含 protected。
- isStatic(int mod)：判断修饰符是否包含 static。
- isFinal(int mod)：判断修饰符是否包含 final。
- isAbstract(int mod)：判断修饰符是否包含 abstract。
- isInterface(int mod)：判断修饰符是否表示一个接口。
- isSynchronized(int mod)：判断修饰符是否包含 synchronized（仅适用于方法）。
- isVolatile(int mod)：判断修饰符是否包含 volatile（仅适用于字段）。
- isTransient(int mod)：判断修饰符是否包含 transient（仅适用于字段）。
- isNative(int mod)：判断修饰符是否包含 native（仅适用于方法）。
- isStrict(int mod)：判断修饰符是否包含 strictfp（仅适用于方法和类）。

### 动态代理

Java 的动态代理是基于反射机制完成的，具体来说，Java 动态代理分为两种：基于接口的动态代理和基于类的动态代理。基于接口的动态代理使用的是 Proxy 类，它提供了静态方法用于生成动态代理实例，整体流程可以总结为：

- 实现 InvocationHandler 接口创建自己的调用处理器 
- 生成接口对应的 Class 对象，Class 对象的构造函数参数是 InvocationHandler 类型
- 通过反射获取参数是 InvocationHandler 类型的 Constructor 对象
- 通过反射 Constructor 对象创建动态代理实例

```java
//使用方式
Proxy.newProxyInstance(
    service.getClassLoader(),
    new Class<?>[] {service},
    new InvocationHandler() {
      private final Platform platform = Platform.get();
      private final Object[] emptyArgs = new Object[0];
      @Override
      public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
          throws Throwable {
        if (method.getDeclaringClass() == Object.class) {
          return method.invoke(this, args);
        }
        args = args != null ? args : emptyArgs;
        return platform.isDefaultMethod(method)
            ? platform.invokeDefaultMethod(method, service, proxy, args)
            : method.invoke(subjectImpl, args);;
      }
    });

//Proxy.java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    Objects.requireNonNull(h);
    final Class<?>[] intfs = interfaces.clone();
    
    //通过classloader加载参数interfaces对应的class对象
    Class<?> cl = getProxyClass0(loader, intfs);
    try {
        //获取参数是InvocationHandler的构造函数
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            cons.setAccessible(true);
        }
        //反射创建动态代理实例
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

getProxyClass0() 加载 Class 对象的过程，使用到了 WeakCache，WeakCache 的构造函数中第一个参数 KeyFactory 可以自定义 Key 的样式。WeakCache 内部是通过 ConcurrentMap 和弱引用（Weak Reference）来缓存数据，因此 WeakCache 的特点如下：

- 存储的数据使用弱引用引用，不需要手动清除，避免内存泄漏

- 数据存储方式是线程安全的，可以在多线程环境下使用

proxyClassCache 如果没有缓存对应的 Class 对象，就会调用 ProxyClassFactory.apply() 生成 Class 对象。

```java
//Proxy.java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
    proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());

private static final class KeyFactory
    implements BiFunction<ClassLoader, Class<?>[], Object> {
    @Override
    public Object apply(ClassLoader classLoader, Class<?>[] interfaces) {
        switch (interfaces.length) {
            case 1: return new Key1(interfaces[0]); 
            case 2: return new Key2(interfaces[0], interfaces[1]);
            case 0: return key0;
            default: return new KeyX(interfaces);
        }
    }
}

private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    return proxyClassCache.get(loader, interfaces);
}

private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>> {
	@Override
	public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
		Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
		for (Class<?> intf : interfaces) {
			Class<?> interfaceClass = null;
			try {
                 //创建Classs对象
				interfaceClass = Class.forName(intf.getName(), false, loader);
			} catch (ClassNotFoundException e) {
			}
			if (interfaceClass != intf) {
				throw new IllegalArgumentException(
					intf + " is not visible from class loader");
			}
			//检查是否是接口
			if (!interfaceClass.isInterface()) {
				throw new IllegalArgumentException(
					interfaceClass.getName() + " is not an interface");
			}
			//检查是否重复
			if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
				throw new IllegalArgumentException(
					"repeated interface: " + interfaceClass.getName());
			}
		}
		String proxyPkg = null;     
		int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
		//设置代理类具有 final 和 public 修饰符
		for (Class<?> intf : interfaces) {
			int flags = intf.getModifiers();
			if (!Modifier.isPublic(flags)) {
				accessFlags = Modifier.FINAL;
				String name = intf.getName();
				int n = name.lastIndexOf('.');
				String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
				if (proxyPkg == null) {
					proxyPkg = pkg;
				} else if (!pkg.equals(proxyPkg)) {
					throw new IllegalArgumentException(
						"non-public interfaces from different packages");
				}
			}
		}
		if (proxyPkg == null) {
			proxyPkg = "";
		}
		{
			List<Method> methods = getMethods(interfaces);
			Collections.sort(methods, ORDER_BY_SIGNATURE_AND_SUBTYPE);
			validateReturnTypes(methods);
			List<Class<?>[]> exceptions = deduplicateAndGetExceptions(methods);
			Method[] methodsArray = methods.toArray(new Method[methods.size()]);
			Class<?>[][] exceptionsArray = exceptions.toArray(new Class<?>[exceptions.size()][]);
			long num = nextUniqueNumber.getAndIncrement();
             //生成唯一的代理类名称：包名+$Proxy+Int数量
			String proxyName = proxyPkg + proxyClassNamePrefix + num;
             //native方法，生成Class对象，构造函数参数是 InvocationHandler 类型
			return generateProxy(proxyName, interfaces, loader, methodsArray,
								exceptionsArray);
		}
	}
}
```

动态代理生成的代理类本身有一些特点：

- 如果所代理的接口都是 public 的，它将被定义在顶层包，即包路径为空
- 如果所代理的接口中有非 public 的接口，它将被定义在该接口所在包
- 类修饰符具有 final 和 public 修饰符
- 类名：格式是“$ProxyN”，其中 N 是一个逐一递增数字，代表 Proxy 类第 N 次生成的动态代理类
- Object 的三个方法 hashCode，equals 和 toString 也会被分派到代理类的 invoke 方法执行



[看完这篇JVM内存管理机制，面试再也不慌了！ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650250707&idx=1&sn=d647d80f0cc0e184da5f657302b26299&chksm=88636cbcbf14e5aa049a3374e997b214cd40fc080487603d204031a7b3451625a6c9b5f961ec)

[字符串常量池、class常量池和运行时常量池_运行时常量池的关系_zhifeng687的博客-CSDN博客](https://blog.csdn.net/qq_26222859/article/details/73135660)

[我将自定义 ClassLoader 的坑都踩了一遍 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650263888&idx=1&sn=1f4f7cfb5659296f5530a72db872df28&chksm=8863203fbf14a9297b4ff41dcbb7c081475fc6d90097212340fcbf66fd445c409038cbce29a5)

[老大难的 Java ClassLoader 再不理解就老了 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/51374915)

[ClassLoader解析（二）：Android中的ClassLoader - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/54745848)

[Android开发中关于内存的那些事，一篇全搞懂 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650270023&idx=1&sn=46974d3d045f4db2254f52bd2b974718&chksm=88631828bf14913ef434660ed6e4e9848771547e3ed6fb4d5a8b5cfcb17e52df72171f592d99)

[Java中的四种引用类型 Strong, Soft, Weak And Phantom (一)_rodbate的博客-CSDN博客](https://blog.csdn.net/rodbate/article/details/72857447)

[Java的值传递和引用传递，你真的搞清楚了吗？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650249167&idx=1&sn=fb761c2450da7e191afd44b6411f6be4&chksm=886366a0bf14efb61306700f7844f49bbf42f472460af1a5f74489da3ebbdc522b7aa64c9518)

[吹爆系列：探索 Android 多线程的一切 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650834312&idx=1&sn=3d5ad284471be428dd94a330d1c26ab0&chksm=80b75116b7c0d800f215621928d4057c46a4e359b76e8d66d620944671f275694e27e77e7b8b)

[似懂非懂 CAS ？这次全都能搞明白 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650273511&idx=1&sn=69c2dc86d1185cb8d67fd4369b56bccc&chksm=88630788bf148e9e01ca25502a39a9e7dd2ac7f0b8cd58f2291f529908ef77fffc799b6262cb&mpshare=1&scene=1&srcid=0320gmObxWAtTWaXeXEdhTD4&sharer_sharetime=1679274749863&sharer_shareid=6abfe554f60ba3cca7d5d67848a573e9#rd)

[ReentrantLock详解 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903598984282119)

[Java 基础 - 泛型机制详解 | Java 全栈知识体系 (pdai.tech)](https://pdai.tech/md/java/basic/java-basic-x-generic.html)

[Java 基础 - 注解机制详解 | Java 全栈知识体系 (pdai.tech)](https://pdai.tech/md/java/basic/java-basic-x-annotation.html)

[Java 基础 - 反射机制详解 | Java 全栈知识体系 (pdai.tech)](https://pdai.tech/md/java/basic/java-basic-x-reflection.html)

[都了解retrofit背后是动态代理，那动态代理的背后是？ (qq.com)](https://mp.weixin.qq.com/s/DMnYWXVx0Gf3Mjs38pfOiA)

[动态代理与静态代理区别_ikownyou的博客-CSDN博客](https://blog.csdn.net/ikownyou/article/details/53081426)