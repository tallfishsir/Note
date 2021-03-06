##### Q1：反射是什么？

反射是 Java 的特征之一，允许运行中的 Java 程序获取自身的信息，操作类或对象的内部属性。通过反射机制，我们可以动态的创建对象并调用、修改其相关的属性。提供的功能：

- 运行时判断一个对象所属的类
- 运行时构造任意一个类的对象
- 运行时判断任意一个类所具有的方法和成员变量
- 运行时调用任意一个对象的方法

反射除了可以在运行时创建对象、获取对象成员变量的值、调用对象方法，还可以利用反射完成运行时注解解析，动态代理的实现。

##### Q2：反射如何使用？

编写的 .java 文件通过 Java 编译器编译后会生成 class 文件，这时再通过 JVM 的类加载器加载到 JVM 的运行时数据区的方法区(.class) 文件，所谓的方法区保存到类的相关信息就是指这一过程的产生的 class 对象。反射依赖于 Class 类和 java.lang.reflect 类库其中有三个重要的类 Field、 Method、 Constructor。

###### Class 相关操作

获取 Class 对象的方法：

1. Class 的 Class.forName 静态方法
2. 直接获取类的 Class 对象
3. 调用对象的 getClass 方法

判断一个 Class 对象是否是某个类的类型，使用 Class 的方法 isInstance()

获取 Class 对象相关信息：

1. getName()：获取的是二进制形式的全限定类名，但是对于内部类，用 $ 开头
2. getSimpleName()：获取的是简单的类名
3. getCanonicalName()：获取的是二进制形式的全限定类名，但是对于内部类，不用 $ 开头，而用 .。局部类和匿名内部类不存在
4. getModifiers()：获取类的修饰符，返回一个 int 数值，可以通过 Modifier 的相关静态方法进行解析。
5. 如果 Class.isArray 是数组的话， getComponentType() 获取数组中的元素类型
6. Array.newInstance(Class<?> componentType, int... dimensions) 创建数组
7. 如果 Class.isEnum 是枚举的话，getEnumConstants() 获取所有的枚举常量

###### Constructor 相关操作

Class.getDeclaredXxx()：获取的是 Class 中所有构造方法，包括被 private 修饰的构造方法

```java
public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)

public Constructor<?>[] getDeclaredConstructors() throws SecurityException 
```

Class.geXxx()：获取的是 Class 中所有 public 修饰的构造方法

```java
public Constructor<T> getConstructor(Class<?>... parameterTypes)

public Constructor<?>[] getConstructors() throws SecurityException 
```

Constructor 创建对象的方法：

1. Class.newInstance()：只能调用无参的构造函数，并且会直接抛出异常
2. Constructor.newInstantce()：可以调用任意的构造方法，包括被 private 修饰的。抛出的异常会被包装在 InvocationTargetException  中。

###### Field 相关操作

Class.getDeclaredFields()/getDeclaredField(String name)：获取的是 Class 中所有成员变量，包括被 private 修饰的

Class.getFields()/getField(String name)：获取的是 Class 中和父类中的所有 public 修饰的Field

获取 Field 的修饰符：

1. getModifiers()：获取类的修饰符，返回一个 int 数值，可以通过 Modifier 的相关静态方法进行解析。

获取 Field 的类型有两个方法：

1. Type getGenericType()：能够获取泛型类型
2. Class<?> getType()：获取类类型

获取 Field 的名称

1. getName()

Field 内容的读取和赋值，因为 Class 本身不存=存储成员，所以要获取具体的成员时，需要一个具体的类对象来承载。

1. get(Object obj)/getXxx(Object obj)
2. set(Object obj, Object value)

###### Method 相关操作

Class.getMethods()/getMethod(String name, Class<?>... parameterTypes)：获取的是 Class 中所有方法，包括被 private 修饰的

Class.getDeclaredMethods()/getDeclaredMethod(String name, Class<?>... parameterTypes)：获取的是 Class 中和父类中的所有 public 修饰的方法

获取 Method 的修饰符

1. getModifiers()：获取类的修饰符，返回一个 int 数值，可以通过 Modifier 的相关静态方法进行解析。

获取 Method 的返回值

1. Type getGenericReturnType()：能够获取泛型类型
2. Class<?> getReturnType()：获取返回值类型

获取 Method 方法名

1. getName()

获取 Method 参数

1. Parameter[] getParameters()：获取参数
2. Parameter.getModifiers()：获取参数的修饰符
3. Parameter.getType()：获取参数类型
4. Parameter.getName()：获取参数名字

获取Method 抛出异常

1. Type[] getGenericExceptionTypes()
2. Class<?>[] getExceptionTypes()

Method 方法的执行

```java
public Object invoke(Object obj, Object... args) {}
```

第一个参数是 Class 类对应的实例对象，如何这个方法是静态的，那么 obj 为 null。

返回值是 Object ，实际执行的时候要进行强制转换

如果方法本来是 private 等修饰，无法访问， 那么需要在 invoke 之前 设置 serAccessiable(true)。

##### Q3：注解是什么？

注解用于为 Java 代码提供元数据，通过 @interface 关键字定义。

Java 原生就提供了元注解，元注解可以理解为注解的注解，包括：

1. @Retention：说明注解的保存时间，它的取值有：
   - RetentionPolicy.SOURCE：注解只保留在源码阶段，编译器编译时会被丢弃；
   - RetentionPolicy.CLASS：注解只保留在编译阶段，不会被加载到 JVM 中；
   - RetentionPolicy.RUNTIME：注解保留在运行时，会被加载到 JVM 中，在程序运行时可以获取到。
2. @Document：可以将注解中的元素添加到 javadoc 中。
3. @Target：限定了直接使用的场景，它的取值有：
   - ElementType.ANNOTATION_TYPE：可以给注解进行注解；
   - ElementType.PACKAGE：可以给包进行注解；
   - ElementType.TYPE：可以给一个类型进行注解
   - ElementType.CONSTRUCTOR：可以给构造函数进行注解；
   - ElementType.FIELD：可以给属性进行注解；
   - ElementType.METHOD：可以给方法进行注解；
   - ElementType.LOCAL_VARIABLE：可以给局部变量进行注解；
   - ElementType.PARAMETER：可以给方法的参数进行注解。
4. @Inherited：如果一个超类被它注解，如果子类没有被任何注解应用的话。子类就继承了超类的注解。

注解的属性也叫成员变量，而且注解只有成员变量，没有方法，注解的成员变量在注解的定义中以“无形参的方法”形式来声明，其方法名定义了该成员变量的名字，其返回值定义了该成员变量的类型。注解中属性可以有默认值，默认值需要用 default 关键值指定。赋值的方式是在注解的括号内以 value=”” 形式，多个属性之前用 ，隔开。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.Type)
public @interface AnnotationDemo{
    int id() default 1;
    String msg() default "world";
}

@AnnotationDemo(id=3,msg="hello")
public class Demo{
    
}
```

##### Q4：注解如何使用？

根据注解的生存周期，我们可以解析的注解可以分为：运行时注解、编译时注解。

###### 运行时注解解析

首先，我们通过反射可以获取到 Class 对象以及 Constructor、Method、Field对象，这些对象都有可能被注解

```java
//  Class、Constructor、Method、Field的 isAnnotationPresent() 方法判断它是否应用了某个注解
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {}
```

然后通过 `getAnnotation()` 或者 `getAnnotations()`获取 Annotation 对象。

```java
public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {}
public Annotation[] getAnnotations() {}
```

###### 编译时注解解析

对编译时的注解进行处理需要做

1.  继承 AbstractProcessor 类
2. 重写其中的process函数

编译器会在编译时自动查找所有继承自AbstractProcessor的类，然后调用他们的process方法去处理。

##### 动态代理的使用和原理

与动态代理相对应的是静态代理，在我看来，静态代理其实是一种装饰者模式：在真正处理逻辑的对象外部包一层装饰者（代理），以此增加功能。

###### 静态代理

```java
// 代理接口
public interface Subject{
    void doSomething();
}

// 真正执行任务的类
public class RealSubject implements Subject{
    @Override
    public void doSomething(){
        // do something
	}
}

// 代理类
public class ProxySubject implement Subject{
    // 代理类持有一个委托类的对象引用
    RealSubject realSubject;
    public ProxySubject(RealSubject realSubject){
        this.realSubject = realSubject;
    }
    
    @Override
    public void doSomething(){
        // do more something
        realSubject.doSomething();
        // do more something
	}
}

// 静态代理类工厂 
public class SubjectFactory{
    public static Subject getInstance(){
        return new ProxySubject(new RealSubject);
    }
}

// 调用方
public class Client1 {
    public static void main(String[] args) {
        Subject proxy = SubjectStaticFactory.getInstance();
        proxy.doSomething();
    }   
} 

```

静态代理类优缺点 
优点：业务类只需要关注业务逻辑本身，保证了业务类的重用性。这是代理的共有优点。 
缺点： 

1. 代理对象的一个接口只服务于一种类型的对象，如果要代理的方法很多，势必要为每一种方法都进行代理。 
2. 如果接口增加一个方法，除了所有实现类需要实现这个方法外，所有代理类也需要实现此方法。增加了代码维护的复杂度。

###### 动态代理

动态代理是在程序运行期间由 JVM 根据反射等机制动态的生成，也就是不需要我们在源码阶段增加 ProxySubject 文件，但相关增加的逻辑还是需要在 invoke() 中添加。

```java
// 实现InvocationHandler接口创建自己的调用处理器 
InvocationHandler handler = new InvocationHandlerImpl(...);

// 通过 Proxy 为包括 Subject 接口在内的一组接口动态创建代理类的类对象  
Class clz = Proxy.getProxyClass(classLoader, new Class[]{Subject.class,...});

// 通过反射从生成的类对象获得构造函数对象  
Constructor constructor = clz.getConstructor(new Class[]{InvocationHandler.class});

// 通过构造函数对象创建动态代理类实例
Subject proxySubject = (Subject) constructor.newInstance(new Object[]{handler});
```

Proxy类的静态方法 newProxyInstance 对上面具体步骤的后三步做了封装，简化了动态代理对象的获取过程。 

```java
Subject subjectImpl = new SubjectImpl();
Subject subjectProxy = (Subject) Proxy.newProxyInstance(SubjectImpl.class.getClassLoader(),
        SubjectImpl.class.getInterfaces(),
        new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                // do more something
                method.invoke(subjectImpl, args);
                // do more something
                return null;
            }
        });
subjectProxy.doSomething();
```

##### Proxy.newProxyInstance 源码解析

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException {
    Objects.requireNonNull(h);
    final Class<?>[] intfs = interfaces.clone();
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }
    // 第一步，生成 intfs 接口代理类的类对象
    Class<?> cl = getProxyClass0(loader, intfs);
    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }
        // constructorParams 就是 InvocationHandler.class 对象
        // 第二步，获取入参是 InvocationHandler 的代理类构造函数
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        // 第三步，通过构造函数对象创建动态代理类实例
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

那么这个代理类的类对象是如何生成的就是关键了，它是通过 proxyClassCache 这个对象获取的。

```java
private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    return proxyClassCache.get(loader, interfaces);
}
```

proxyClassCache 是一个 WeakCache 对象，其中 map 是最重要的成员变量，从本质来看，map 的第一层 key 是 ClassLoader，第二层 key 是 interface，由此来缓存某个 Factory 对象。proxyClassCache 的 get 方法返回 ProxyClassFactory.apply 的返回值，返回值实际上最终是通过 ProxyGenerator.generateProxyClass( proxyName, interfaces, accessFlags) 来生成最后的代理类。

```java
final class WeakCache<K, P, V> {
    private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
    = new ConcurrentHashMap<>();
    
    public WeakCache(BiFunction<K, P, ?> subKeyFactory, BiFunction<K, P, V> valueFactory) {
        this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
        this.valueFactory = Objects.requireNonNull(valueFactory);
    }
    
    public V get(K key, P parameter) {
        Objects.requireNonNull(parameter);
        expungeStaleEntries();
        //获取第一层的key，是一个继承WeakReference的CacheKey对象，CacheKey对象包含了classLoader和ReferenceQueue两个成员
        Object cacheKey = CacheKey.valueOf(key, refQueue);
        // 通过 classLoader 拿到第一层的 value
        ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
        // 如果 value 没有就创建一个
        if (valuesMap == null) {
            ConcurrentMap<Object, Supplier<V>> oldValuesMap = map.putIfAbsent(cacheKey, valuesMap = new ConcurrentHashMap<>());
            if (oldValuesMap != null) {
                valuesMap = oldValuesMap;
            }
        }
        // 这里关键，subKeyFactory 是在前面构造函数中赋值的，而调用它的构造函数发生在 Proxy.class 中：
        // private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        // proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
        // KeyFactory.apply 实际上就是就是根据 classLoader 和 interface 生成第二层的 key
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
        // 拿到第二层的 value，supplier对象
        Supplier<V> supplier = valuesMap.get(subKey);
        Factory factory = null;

        while (true) {
            if (supplier != null) {
                // 后面的分析会得到 supplier 实际上是 Factory 对象
                // Factory.get() 方法最后返回的其实是 ProxyClassFactory.apply()
                V value = supplier.get();
                if (value != null) {
                    return value;
                }
            }
            if (factory == null) {
                factory = new Factory(key, parameter, subKey, valuesMap);
            }
            // valuesMap 第二层的 value 实际上是一个 Factory 对象
            if (supplier == null) {
                supplier = valuesMap.putIfAbsent(subKey, factory);
                if (supplier == null) {
                    supplier = factory;
                }
            } else {
                if (valuesMap.replace(subKey, supplier, factory)) {
                    supplier = factory;
                } else {
                    supplier = valuesMap.get(subKey);
                }
            }
        }
    }
}
```

到此，WeakCache.get 方法返回的实际上是 ProxyClassFactory.apply() 的返回值，也就是说，代理类是在这个方法中生成的。

```java
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
    Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
    // 验证 interface 的合法性
    for (Class<?> intf : interfaces) {
        Class<?> interfaceClass = null;
        try {
            interfaceClass = Class.forName(intf.getName(), false, loader);
        } catch (ClassNotFoundException e) {
        }
        if (interfaceClass != intf) {
            throw new IllegalArgumentException(intf + " is not visible from class loader");
        }
        if (!interfaceClass.isInterface()) {
            throw new IllegalArgumentException(interfaceClass.getName() + " is not an interface");
        }
        if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
            throw new IllegalArgumentException("repeated interface: " + interfaceClass.getName());
        }
    }
    // 生成包名
    String proxyPkg = null; 
    int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
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
                throw new IllegalArgumentException("non-public interfaces from different packages");
            }
        }
    }
    if (proxyPkg == null) {
        proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
    }
    /*
     * 生成类名
     */
    long num = nextUniqueNumber.getAndIncrement();
    String proxyName = proxyPkg + proxyClassNamePrefix + num;
    /*
     * 最关键一步，生成 代理类
     */
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass( proxyName, interfaces, accessFlags);
    try {
        return defineClass0(loader, proxyName, proxyClassFile, 0, proxyClassFile.length);
    } catch (ClassFormatError e) {
        throw new IllegalArgumentException(e.toString());
    }
}
```

生成的 Proxy 类是实现了 interfaces 的方法的，并且在每个方法中都调用了 InvocationHandler.invoke 方法。