# 基础语法 

## 变量

### 变量声明

Kotlin 中声明变量是以关键字开头，然后是变量名称，最后加上变量类型。结尾不需要分号。变量名称和变量类型之间用「:」进行分隔。

```kotlin
var a : Int = 1
val b : String = "bbb"
```

其中，变量类型在可以推断的情况下，可以不写，同时「:」也不需要写

变量声明中的关键字有两个：var 和 val

- var：可变引用，这种变量的值可以改变吗，对应 Java 中的非 final 变量
- val：不可变引用，使用 val 声明的变量不能在初始化后再次赋值，对应 Java 中的 final 变量

### 延迟初始化

在 Kotlin 编程中，优先使用 val 来声明一个本身不可变的变量，这是一种防御性的编码思维模式，不可变变量意味着更加容易推理，更加安全和可靠。但声明一个 val 变量会带来一个问题，就是初始化变量时机。正常情况下， Kotlin 规定类中的所有非抽象属性成员必须在对象创建的时候被初始化，但 val 一旦被初始化为就不可修改，在 Android 的 Activity 内很多变量都是在 onCreate 中初始化的。所以可以使用 **by lazy** 、 **lateinit** 和 **Delegates.notNull\<T>** 语法实现延迟初始化。

#### by lazy

```kotlin
class Demo{
    val param1: String by lazy { if (true) "params" else "error" }
//    val param1: String by lazy(LazyThreadSafetyMode.SYNCHRONIZED) { if (true) "params" else "error" }
}
```

变量必须是 val 声明的不可变变量，它在被首次调用的时候才会进行赋值操作，一旦赋值就不能再被修改。系统会给 lazy 属性默认加上同步锁，也就是 LazyThreadSafetyMode.SYNCHRONIZED，所以他是线程安全的。如果开发者认为初始化没有线程安全的问题，可以给 lazy 传递 LazyThreadSafetyMode.PUBLICACTION 参数。

#### lateinit

```kotlin
class Demo{
    lateinit var params1 : String
    fun printParams() {
        this.params1 = if (true) "params" else "error"
        print(this.params1)
    }
}
```

lateinit 主要用于 var 声明的变量中，但它不能用于基础数据类型：Int、Long 等，需要使用 Integer 这种包装类代替。

#### Delegates.notNull\<T>

对于 var 声明的且还是基础数据类型的变量，一种解决方案是通过Delegates.notNull\<T>，利用委托的语法来实现。

```
var test by Delegates.notNull<Int>()
var testLong by Delegated.notNull<Long>
```

另外可以利用代码来判断一个全局变量是否已经完成了初始化

```
class Demo{
    lateinit var params1 : String
    fun printParams() {
       if(!::param1.isInitalized){
       		...
       }
    }
}
```

## Kotlin 变量的类型系统

与 Java 相比，Kotlin 引入了可空的类型和只读的集合，并把数组作为头等公民来支持。

### Kotlin 的非空和可空类型

在 Kotlin 中区分非空和可空类型，任何类型单独存在时是非空类型，在类型后面加上「?」就是可空类型

> 可空和非空的对象在运行时没有什么区别，可空类型并不是非空类型的包装。所有的检查都发生在编译期，以为着使用 Kotlin 的可空类型并不会在运行时带来额外的开销。

#### 安全调用运算符「?.」

对于一个可空类型的变量，在调用其方法或者属性的时候，经常会进行判空的逻辑操作，即：

```kotlin
if (a != null) {
    a,do()
} else {
    ...
}
```

安全调用运算符取代了这个繁琐的样式。

```kotlin
a?.do()
```

整个表达式的值等于：如果 a 不为空，值是 a.do() 的结果；如果 a 是空，值是 null。

#### Elvis 运算符「?:」

也称作 null 合并运算符，它接收两个参数，如果第一个参数不为空，运算结果就是第一个参数；如果第一个参数是 null，运算结果就是第二个参数

```kotlin
a ?: b
```

#### 安全转换运算符「as?」

as 是用来转换类型的常规运算符，如果被转换的值不是试图转换的类型，就会抛出 ClassCastException 异常。如果使用安全转换运算符，当值不是试图转换的类型就会返回 null 了。

#### 非空断言「!!」

可以把任何值转换成非空类型，如果对 null 做非空断言，运行时会产生 NullPointerException。

### Kotlin 基础数据类型

Kotlin 的基本数据类型包括：Byte、Short、Int、Long、Double、Float、Char、Boolean等。Kotlin 并不区分基本数据类型和包装类型，在运行时，数字类型会尽可能的使用最高效的方式表示，对于变量、属性——Kotlin 的 Int 类型会编译成 Java 的基本数据类型 int ，泛型类中用作泛型类型参数的基本数据类型会被编译成 Java 对应的包装类型。

Kotlin 中处理数字转换时，不会自动把数字从一个类型转换成另一个类型，必须进行显式转换。每一种基本类型都定义了转换函数：toByte()、toShort()等。

### Kotlin 字符串

Kotlin 中通过双引号来定义一个字符串，它是不可变对象。通过三引号来定义一个原生字符串，原生字符串的意义是最终的打印格式和引号中的内容格式一致，不会解释转化转移字符。通过 ${} 来定义字符串模板。

字符串的判等型主要有两种类型：

- 结构相等：通过操作符「==」来判断两个对象的内容是否相等
- 引用相等：通过操作符「===」来判断两个对象的引用是否一样。

### Kotlin 根类型

Any 类型是 Kotlin 所有非空类型的超类型，和 Object 作为 Java 的类层级结构的根类型差不多。但在 Java 中 Object 只是所有引用类型的超类型，而基本数据类型不在其中。但 Any 是所有非空类型的超类型，包括基础数据类型。

Any? 类型是 Kotlin 的所有类型的超类型。在底层，Any 类型对应 java.lang.Object，当 Kotlin 函数使用 Any 时，它会被编译成 Java 字节码中 Object。

### Kotlin 的 “void”：Unit 类型

Kotlin 中的 Unit 类型完成了 Java 中的 void 一样的功能。当函数没有意义的结果返回是，可以用作函数的函数类型。

### Kotlin 数组

Kotlin 中的一个数组是一个带有类型参数的类，其元素类型被指定为相应的类型。创建数组的方式有：

1. arrayOf 函数创建一个数组，它包含的元素是指定为该函数的实参
2. arrayOfNulls 创建一个指定大小的数据，包含的是 null 元素
3. Array 构造方法接收数组的大小和一个 Lambda 表达式，调用 Lambda 表达式来创建一个数组元素

```
val firstArray = arrayOf(1, 2, 3, 4, 5)
val secondArray = arrayOfNulls<Int>(5)
val thirdArray = Array<Int>(5) {it}
```

数组类型的类型参数会变成对象类型，也就是如果声明了 Array\<Int>，它将会是一个包含装箱整型类型的数组，如果需要创建没有装箱的基本数据类型的数组，需要引入了特殊类。每一种基本数据类型都对应一个：Int 类型值的数组叫作 IntArray，Char 类型值的数组叫作 CharArray。

### Kotlin 集合

Koltin 把访问集合数据的接口和修改集合数据的接口分开了。**Collection** 接口中可以遍历元素、获取集合大小、判断集合是否包含某个元素，以及执行其他从该集合读取数据的操作，但这个接口没有任何添加和移除元素的方法。**MutableCollection** 接口可以修改集合中的数据，它继承了 Collection 接口，增加了添加、移除元素的方法。

| 集合类型 |  只读  |                      可变                      |
| :------: | :----: | :--------------------------------------------: |
|   List   | listOf |           mutableListOf arrayListOf            |
|   Set    | setOf  | mutableSetOf hashSetOf linkedSetOf sortedSetOf |
|   Map    | mapOf  | mutableMapOf hashMapOf linkedMapOf sortedMapOf |

集合的高阶函数 API ：

- forEach 普通遍历集合方式

  ```kotlin
  stringList.forEach { println("forEach:" + it) }
  ```

  map、count、sumBy、reduce 等

- forEachIndexed 编译集合且有下标

  ```kotlin
  stringList.forEachIndexed { index, string -> println("forEachIndexed:$index $string") }
  ```

- firstOrNull/lastOrNull 返回与指定条件相符的第一个/最后一个元素，如果没有就返回 null

  ```kotlin
  stringList.firstOrNull { it == ”1“ }?.let { println("firstOrNull:" + it) }
  stringList.lastOrNull  { it == ”1“ }?.let { println("lastOrNull :" + it) }
  ```

- filter 过滤出符号条件的元素

  ```kotlin
  stringList.filter { it == "1" || it == "3" }.forEach { println("filter:" + it) }
  ```

- filterNotNull 过滤出所有不为空的字段

  ```kotlin
  stringList.filterNotNull().forEach { println("filterNotNull:" + it) }
  ```

- filterNot 类似于代码中 "`!`" 非的功能, 打印出所有不等于1的字段

  ```kotlin
  stringList.filterNot { it == "1" }.forEach { println("filterNot:" + it) }
  ```

- map 返回一个列表，该列表包含对原始集合中每个元素进行转换后结果

  ```kotlin
  intList.map { it + 20 }.forEach { println("map:" + it) }
  ```

## 函数

### 函数的声明和调用

#### 函数的声明

Kotlin 中函数的函数体有两种形式：表达式函数体和代码块函数体。

表达式函数体是用单行表达式与等号的语法来定义函数：

```
fun sum(x: Int = 1, y: Int = 2): Int = x + y
```

代码块函数体是以{}来定义函数内容

```kotlin
fun sum(x: Int = 1, y: Int = 2): Int{
    return x + y
}
```

函数的声明以关键字 fun 开头，然后是函数名，然后是括号括起来的参数列表，然后是冒号加上返回类型，最后是函数体。

- 参数的默认值时可以省略
- 返回值如果是可以根据 return 语句推断出，是可以省略的

Kotlin 通过 **varargs** 关键字来定义函数中的可变函数，类似 Java 中的 「..」，但 Java 中的可变参数必须是最后一个参数，Kotlin 中没有这个限制。两者都可以在函数体中以数组的方式来使用可变参数变量。

#### 函数的调用

当调用函数时，可以显式表明一些参数的名称，当调用一个函数时，指明了一个参数的名称，为了避免混淆，那它之后的参数都需要标明名称。

```kotlin
sum(x = 2, y = 3)
```

如果函数声明是设置了默认参数值，调用此函数时，如果是常规调用没有标明名称，可以省略排在末尾的参数；如果标明了名称，可以省略中间的一些参数。

### 扩展函数

扩展函数可以理解为定义在类的外面的成员函数，它的声明是以关键字 fun  开头，然后把要扩展的类或者接口名称，然后是扩展函数名称。这个类的名称称为接收者类型，用来调用这个扩展函数的那个对象，叫作接收者对象。在这个扩展函数中，可以像其他成员函数一样使用 this ，也可以省略它。

```
fun String.lastChar(): Char = this.get(this.length - 1)
```

本质上，扩展函数会编译成静态方法，独立于类对任何对象，被该类的所有实例共享，也就是一个全局方法，不会带来性能消耗。

除了扩展函数，Kotlin 也支持扩展属性，和扩展函数一样，扩展属性也向接收者的一个普通的成员属性，但因为没有支持的字段，因此没有默认 getter 的实现，所以必须定义 getter 函数。

```kotlin
val String.lastChar : Char
	get() = get(length - 1)

var StringBuilder.lastChar : Char
	get() = get(length - 1)
	set(value : Char) {
        this.setCharAt(length - 1, value)
    }
```

当扩展函数和现有类的成员方法同时存在时，Kotlin 将会默认使用类的成员方法。

### 高阶函数

Kotlin 的高阶函数是以其他函数作为参数或者返回值的函数。

#### 声明参数中函数类型

```
ClassName.(Int, Int) -> Int
```

函数类型格式声明遵循以下几点：

1. 通过「->」来连接**参数类型**和**返回值类型**，左边是参数类型，右边是返回值类型
2. 参数类型必须用 () 包裹，如果没有参数，也不能省略括号，多个参数使用 ，分隔
3. 返回值类型就算是 Unit，也必须显示声明
4. 函数类型前加上 ClassName 就表示这个函数类型定义在哪个类中，将自动拥有 ClassName 代表的类的上下文

#### 调用含有函数类型参数的函数

当调用含有函数类型参数的函数时，需要传入一个函数的引用，这里说的函数引用其实就是一个函数类型的对象。函数的引用有以下方法获取：

1. Kotlin 中可以通过两个冒号来实现对于某个类的方法的引用，作为参数
2. 匿名函数：Kotlin 支持缺省函数名的情况下，直接定义一个函数，作为参数
3. Lambda 表达式，本质上也是匿名函数，作为参数

### Lambda 表达式

Kotlin 的 Lambda 表达式始终用花括号包围，箭头把参数类型和函数体分开。可以把 Lambda 表达式存储在一个变量中，然后把这个变量当作普通函数看待。Lambda 内部可以访问外部的变量，并不限制于 final 变量，这个过程就叫做 Lambda 捕捉。

```
val sum:(Int, Int) -> Int = { x: Int, y: Int -> x + y}
val sum = { x: Int, y: Int -> x + y}
val sum:(Int, Int) -> Int = {x, y -> x + y}
```

1. Lambda 表示式必须通过 {} 包裹
2. 如果 Lambda 声明了参数部分类型并且返回值支持类型推导， Lambda 变量可以省略函数类型声明
3. 如果 Lambda 声明了函数类型，那么 Lambda 的参数部分的类型就可以省略

### 内联函数 inline

Lambda 表达式会被编译成匿名类，每调用一次 Lambda 表达式，一个额外的类就会被创建，并且如果 Lambda 捕捉了某个变量，每次调用的时候还会创建一个对象，这会带来额外的开销。Kotlin 使用 **inline** 修饰符标记一个函数，在函数使用的时候，编译器并不会生成函数调用的代码，而是使用函数实现的真是代码替换每一次的函数调用。

由于内联的运作方式，对于使用 Lambda 作为参数的函数而言，不是所有使用 Lambda 作为参数的函数可以被内联。一般来说，如果函数参数被直接调用或者传递给另一个 inline 函数，这个函数是可以被内联的。

使用 **inline** 关键字只能提高带有 Lambda 参数的函数的性能，节约了函数调用、为 Lambda 创建匿名类、创建 Lambda 实例对象的开销。但内联函数只允许传递给另一个内联函数，而非内联函数可以自由的传递给其他任何函数。

#### 内联函数 let

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R = block(this)
```

调用 T 对象的 let 函数，则该对象为函数的参数，在函数内可以通过 it 指代该对象。返回值是函数的最后一行或者指定 return 表达式。可以利用它进行空安全验证：

```kotlin
val data = ...
data?.let{
	...
}
```

#### 内联函数 also

```kotlin
@kotlin.internal.InlineOnly
@SinceKotlin(“1.1”)
public inline fun T.also(block: (T) -> Unit): T { block(this); return this }
```

调用 T 对象的 also 函数，在函数范围内可以通过 it 指代该对象，返回值是该对象本身。与 let 的区别是 let 返回值是最后一行。

#### 内联函数 with

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R = receiver.block()
```

将对象作为函数的参数，在函数内可以通过 this 指代该对象，返回值是函数的最后一行或者指定 return 表达式。

#### 内联函数 run

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R = block()
```

调用 T 对象的 run 函数，会执行函数内容，返回值是函数的最后一行或者指定 return 表达式。

#### 内联函数 apply

```kotlin
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T { block(); return this }
```

调用 T 对象的 apply 函数，在函数范围内，可以任意调用该对象的方法，最后返回该对象。与 run 的区别是 run 返回的是最后一行。适用于对象初始化需要给其属性赋值的情况。

### inline/noinline/crossinline

简单总结：

- inline：通过内联的方式（函数内容直接插入到调用处）优化代码层级和内存占用
- noinline：在局部关闭 inline 这个优化
- crossinline：在局部加强 inline 这个优化

**inline** 不止可以内联自己内部代码，还可以内联自己的函数类型的参数，比如：

```kotlin
inline fun sayHi(action : () -> Unit) {
	println("hi")
	action()
}
```

在调用时传入的 Lambda 表达式，在编译后会变成实际上 action() 代码，而不会创建一个对象。假如有一个 inline 关键字修饰的高阶函数接收了两个及以上的函数类型的参数， Kotlin 编译器会自动将所有的 Lambda 表达式进行内联。如果我们只想内联其中一个 Lambda 表达式，就要将不需要内联的参数前使用 **noinline** 关键字进行修饰。

```kotlin
inline fun sayHi(action1 : () -> Unit,noinline action2 : () -> Unit) {
	action1()
	println("hi")
	action2()
}
```

为什么会不想内联某个参数呢？因为一旦内联，这个参数就只是一个形式而已，在函数内部只能调用它，而不能对它有其他形式的访问（比如用它当做返回值）。

Lambda 表达式有一个规则是：不允许使用 return 来返回结果，除非是个 inline 函数。如果在内联函数的 Lambda 表达式中使用了 return，结束的不是直接的外层函数，而是外层再外层的函数。

这个规则间接引入另一个规则： inline 函数的参数中 Lambda 表达式不允许间接调用，为了解除不能间接调用的限制，要在参数前加上 **crossinline** 解除。但一旦加上这个关键字，Lambda 中就又不允许使用 return了。

## 类和接口

声明类时，使用关键字 class ，类的属性完全代替了字段和访问器方法，Kotlin 中的属性除非显式的声明延迟初始化，不然就需要指定属性的默认值。声明成 val 的属性是只读的，所以自定义访问器方法时只有 getter 方法，声明成 var 的属性时可变的，所以自定义访问器方法时有 setter 和 getter 方法。

```kotlin
class Person {
    val name: String = ""
        get() = field + "real"
    var age : Int = 0
        get() = field + 1
        set(value) {
            field = value - 1
        }
}
```

声明接口时，使用关键字 interface，接口的方法可以有默认实现。

```
interface Clickable{
    fun click()
    fun showOff() = print("i am clickable")
}
```

### 构造函数

Kotlin 的构造函数区分了主构造函数和从构造函数，主构造函数在类体外部声明，从构造函数在类体内部声明，同时也允许在初始化语句块中添加额外的初始化的逻辑。

#### 主构造函数

```kotlin
class Person constructor(name: String){
    val nickName: String
    init {
        nickName = name
    }
}
```

在类名后，constructor 和括号围起来的语句块叫作主构造函数。**constructor** 关键字用来开始一个主构造函数或者从构造函数的声明。**init** 关键字用来引入一个初始化语句块。初始化语句块包含了在类被创建时指定的代码，并会和主构造函数一起使用。

当 **construction** 关键字没有注解或者可见性修饰符作用时，可以省略。当定义类时，并没有显式提供一个主构造函数，Kotlin 编译器会自生成一个无参的主构造函数。

如果在主构造函数中使用 val/var 关键字声明了属性，可以将初始化语句块简化，等价于在类内部声明了一个同名的属性。

```kotlin
class Person(val nickName: String)
```

和函数一样，构造函数可以为参数声明一个默认值，创建类的对象时，只需要调用构造方法，不需要 new 关键字。

```
class Person(val nickName: String = "Tony")
```

如果类具有一个父类，主构造函数同样需要初始化父类，可以通过在基类列表的父类引用中提供父类构造方法参数来实现：

```kotlin
open class User(val nickName: String)

class Person(val nickName: String = "Tony") : User(nickName)
```

如果想要类不被其他代码实例化，必须把构造函数标记为 private

```
class Secreive private constructor(){}
```

#### 从构造函数

从构造函数是在函数体内部使用 **constructor** 关键字声明。如果类声明了主构造函数，从构造函数需要使用 this 关键字来调用主构造函数。主构造函数会作为从构造函数的第一条语句。

```kotlin
class Person constructor(val name: String){
    constructor(a: Int = 1) : this(a.toString()) {
    }
}
```

#### init 语句块

init 语句块属于主构造函数的一部分，如果需要在初始化时进行其他的额外操作，就可以使用 init 语句块。当有多个 init 语句块时，他们会在对象被创建时按照类中从上到下的顺序先后执行。

主构造函数，从构造函数，初始化代码块的执行顺序：先主构造函数执行，然后执行初始化代码块，然后执行从构造函数。

### 可见性限制修饰符

可见性修饰符用于控制对代码中声明的访问。Java 中的默认可见性——包私有，在 Kotlin 中没有使用，Kotlin 中的默认可见性是 **public**。Kotlin 只把包作为在命名空间里组织代码的方法使用，并没有将其用于可见性控制。Kotlin 中新增了 **internal** 可见性修饰符，表示在模块内部可见。一个模块就是一组一起编译的 Kotlin 文件。它的优势在于提供了对模块实现细节的真正封装，使外部代码将类定义到与你代码相同的包中，以此破坏封装性的操作，不再能实现。

| 修饰符         | 类成员       | 顶层声明     |
| -------------- | ------------ | ------------ |
| public（默认） | 所有地方可见 | 所有地方可见 |
| internal       | 模块中可见   | 模块中可见   |
| protected      | 子类中可见   | -            |
| private        | 类中可见     | 文件中可见   |

Kotlin 中的 public、protected、private 修饰符在编译成 Java 字节码时会被保留，internal 会变成 public。

### 继承结构和相关修饰符

Kotlin 声明类时，在类名后面使用冒号来代替 Java 中的 extends 和 implements 关键字。一个类可以实现多喝接口，但只能继承一个类。

#### 使用 override 重写方法

当重写方法时，需要使用 **override** 修饰符写在函数名前。当类实现了两个接口，这两个接口中有一个名称相同的方法。这时需要类必须显式实现这个同名方法。

```kotlin
interface Clickable{
    fun click()
    fun showOff() = print("i am clickable")
}

interface Focusable{
    fun showOff() = print("i am focusable") 
}

class View : Clickable, Focusable{
    override fun click() {
    }

    override fun showOff() {
        super<Clickable>.showOff()
        super<Focusable>.showOff()
    }
}
```

#### open、final 和 abstract 修饰符

Kotlin 中的类和方法默认是 **final** 的，这就意味着：类是不可以被继承的，方法也不可以被重写。

如果想要创建一个类的子类，需要使用 **open** 修饰符来标识这个类。如果想要重写某个方法，需要使用 **open** 修饰符来标识这个方法。一旦重写了一个类的函数，这个重写的方法同样默认是 open 的，如果想再次改变这个行为，组织子类的子类重写这个方法，可以显式的将重写的方法标识为 **final**。

Kotlin 中可以将一个类声明为 **abstract** ，这种类不能被实例化。一个抽象类通常包含一些没有实现，必须要在子类中实现的抽象方法。抽象函数始终都是 **open** 的。

```kotlin
abstract class Animated{
    abstract fun animate()
    open fun stopAnimate() {
    }
    fun startAnimate() {
    }
}
```

| 修饰符   | 相关成员               | 说明                                         |
| -------- | ---------------------- | -------------------------------------------- |
| final    | 不能被重写             | 类和类中成员默认使用                         |
| open     | 可以被重写             | 需要明确的表明                               |
| abstract | 必须被重写             | 只能在抽象类中使用，抽象成员不能有实现       |
| override | 重写父类或接口中的方法 | 如果没有使用 final，重写的成员默认是 open 的 |

### 内部类和嵌套类

Kotlin 中可以在一个类中声明另一个类。Java 中这种形式下，类中的类，被称为内部类，默认含有一个外部类的引用。如果要修复这个问题，需要将内部类声明为 static ，从而变成嵌套类。在 Kotlin 中这个情况下，类中的类，默认是嵌套类，如果想要把它变成内部类，需要使用 **inner** 修饰符。

| 类 A 在另一个类 B 中声明     | 在 Java 中     | 在 Kotlin 中  |
| ---------------------------- | -------------- | ------------- |
| 嵌套类（不保存外部类的引用） | static class A | class A       |
| 内部类（保存外部类的引用）   | class A        | inner class A |

### 数据类

Kotlin 在 class 关键字前加 **data** 表明这是一个数据类。数据类自动实现 toString、equals、hashCode 方法，对主构造函数中的属性，自动生成 getter 和 setter。

声明一个数据类，需要满足以下条件：

- 数据类必须拥有一个构造方法，该方法至少包含一个参数；
- 数据类的构造方法的参数强制使用 val 或 var 进行声明
- data class 之前不能用 abstract、open、sealed、inner 进行修饰

### 密封类

Kotlin 在 class 关键字前加 **sealed** 表明这个是一个密封类。密封类的子类只能定义在密封类内部或者同一个文件中，因为其构造方式是私有的。密封类相比于普通的 open 类，可以不被此文件外继承。与枚举的区别在于：密封类适用于子类可数的情况，枚举适用于实例可数的情况。

```kotlin
sealed class Expr{
    class Num(val value: Int) : Expr() {
    }
    class Sum(val left: Int, val right: Int) : Expr() {
    }
}
```

### "object" 关键字和伴生对象

object 关键字表示：声明一个类并且创建一个实例。它有以下的使用场景：

- 对象声明，用于创建单例
- 创建伴生对象，在调用伴生对象时不依赖类实例，直接通过类名访问
- 创建对象表达式，用于替代 Java 的匿名内部类

#### 创建单例对象

Kotlin 使用 object 来声明类，一个对象声明可以以一句话来定义一个类和一个该类的单一实例。和类一样，一个对象声明也可以包含属性、方法、初始化代码块等声明，但不允许有构造函数（主构造函数和从构造函数）与普通类的实例不同，对象声明在定义的时候就创建了，不需要在代码的其他调用构造方法。

```kotlin
object Payroll{
    val allEmployees = arrayListOf<Person>()
    fun calSalary(){
        allEmployees.forEach { print(it) }
    }
}

Payroll.allEmployees.add(Person())
Payroll.calSalary()
```

#### 创建伴生对象

在 Java 中，一个类既有静态变量、静态方法、也有普通变量和普通方法的声明。然而静态变量和静态方法时属于一个类的，普通变量和普通方法是属于一个具体对象的。虽然有 static 作为区分，但在代码结构上区分的并不清晰。Kotlin 通过 **companion object** 两个关键字引入伴生对象。伴生对象意为伴随某个类的对象，它属于这个类所有，因此伴生对象和 Java 中 static 修饰的效果性质一样，全局只有一个单例。它需要声明在类的内部，在类被装载时会被初始化。

```kotlin
class Prize(val name: String, val type: Int) {
    companion object{
        val TYPE_REDPACK = 0
        val TYPE_COUPON = 1
        fun isRedType(prize: Prize): Boolean {
            return prize.type == TYPE_REDPACK
        }
    }
    
    fun createRed(){
        // do something
    }
}
```

#### 创建对象表达式

object 关键字除了可以创建单例，还能用来声明匿名对象，匿名对象替代了 Java 中匿名内部类的用法。

```kotlin
Collections.sort(arrayListOf(1,2,3), object : Comparator<Int>() {
    override fun compare(o1: Int?, o2: Int?): Int {
        return (o1 - o2)
    }
})	
```

和对象声明不同，匿名对象不是单例，每次对象表达式被执行都会创建一个新的对象实例。对象表达式的代码可以访问创建它的函数中的变量，而且访问并没有限制在 final 变量，还可以在对象表达式中修改变量的值

# 进阶语法

## 运算符重载和其他约定

Kotlin 中一些功能与特定的函数命名有关，而不是与特定的类型绑定，这个技术称为**约定**。重载运算符需要使用 **operator** 关键字来标记方法，表示将这个方法作为相应的约定的实现。

### 重载算术运算符

比如「+」号运算符，需要重载 plus 方法：

```kotlin
data class Point(var x: Int, var y: Int){
    operator fun plus(other: Point): Point {
        return Point(x + other.x, y + other.y)
    }
}

val p1 = Point(1, 2)
val p2 = Point(2, 3)
val p3 = p1 + p2
```

可重载的运算符有：

| 表达式   | 函数名      |
| -------- | ----------- |
| a * b    | times       |
| a / b    | div         |
| a % b    | mod         |
| a + b    | plus        |
| a - b    | minus       |
| +a       | unaryPlus   |
| -a       | unaryMinus  |
| ++a, a++ | inc         |
| --a, a-- | dec         |
| +=       | plusAssign  |
| -=       | minusAssign |
| *=       | timesAssign |

需要注意的是：

- Kotlin 运算符不会自动支持交换律
- 当 plus 和 plusAssign 都定义且适用的时候，编译器会报错

另外，Kotlin 中没有为标准数字类型定义任何位运算符，因此也不允许为自定义类型定义它们，Kotlin 支持使用中缀调用语法来进行位运算。

| 中缀语法的常规函数 | 意义       |
| ------------------ | ---------- |
| shl                | 带符号左移 |
| shr                | 带符号右移 |
| ushr               | 无符号右移 |
| and                | 按位与     |
| or                 | 按位或     |
| xor                | 按位异或   |
| inv                | 按位取反   |

### 重载比较运算符

Kotlin 中对可以任意对象使用比较运算符（==、!=、>、<等），而不仅仅限于基本数据类型。不用像 Java 调用 equals 或 compareTo 函数，可以直接使用比较运算符。

「==」和「!=」运算符会被转换成 equals 函数的调用，而且这两个运算符可以用于**可空运算数**，因为这些运算符会检查运算数是否是 null。

「===」恒等运算符是用于比较两个对象的引用是否是同一个（如果是基础数据类型，检查是否是相同的值），且它不能被重载。

「>」、「>=」、「<」、「<=」运算符会被转换成 Comparable 接口中的 CompareTo 函数的调用。

### 集合和区间的约定

处理集合最常见的一些操作是通过下标来获取和设置元素，以及检查元素是否属于当前集合。所有的这些操作都支持运算符语法：array[index]。

通过下标运算符来获取元素会被转换成 get 方法的调用，写入数据会被转换成 set 方法的调用。

```kotlin
data class Point(var x: Int, var y: Int){
    operator fun get(index: Int): Point {
        return Point(x, y)
    }
    operator fun set(index: Int, value: Int): Unit {
        y = value
    }
}
```

「in」用于检查某个对象是否属于集合，被转换成 contains 方法。

「..」用于创建一个区间，被转换成 rangeTo 方法

### 解构声明

解构声明支持将一个复合值展开成单个值，并初始化这些单独变量。它的使用如下：

```kotlin
data class Point(val x: Int, val y: Int)

val p = Point(19, 29)
val (a, b) = p
```

解构声明看起来像一个普通的变量声明，但它的括号中有多个变量。本质上解构声明被转换成 componentN 的函数，其中 N 是声明中变量的位置，对于数据类来说，编译器为每一个在主构造函数中声明的属性生成了 componentN 函数。但标准库对 conponentN 函数的数量做了限制，值允许访问一个对象的前五个元素。

```kotlin
class Point(val x: Int, val y: Int){
    operator fun component1() = x
    operator fun component2() = y
}
```

解构声明常用在对复杂对象的展开上，比如遍历 map 时，Map.Entry 的 conponent1 和 conponent2 分别对应它的键和值。

## 委托

### 类委托

类委托的核心思想是将一个类的具体实现交给另一个类完成。如果没有 Kotlin 类委托的语法支持，完成委托模式的代码如下：

```kotlin
class MySet<T>(val helpSet: HashSet<T>) : Set<T> {
    override val size: Int
        get() = helpSet.size

    override fun contains(element: T): Boolean {
        return helpSet.contains(element)
    }

    override fun containsAll(elements: Collection<T>): Boolean {
        helpSet.containsAll(elements)
    }

    override fun isEmpty(): Boolean {
       return helpSet.isEmpty()
    }

    override fun iterator(): Iterator<T> {
       return helpSet.iterator()
    }
}
```

MySet 中的所有方法都是通过辅助对象来实现的，但如果接口中需要实现的方法较多，如果都是自己实现，工作量会很大，所以 Kotlin 的委托语法是解决这个问题出现的。

Kotlin 中使用关键字 by 来使用委托，上面的代码可以写为：

```
class MySet<T>(val helpSet: HashSet<T>) : Set<T> by helpSet {
}
```

如果需要对某个方法进行重新实现，只需要单独重写那个方法就可以

### 委托属性

委托属性的核心思想是将一个属性的具体实现交给另一个类实现，基本语法是这样的：

```kotlin
class Foo{
    var p : Type by Delegate()
}

Delegate
```

属性 p 将它的访问器逻辑委托给了另一个对象。这里时 Delegate 类的一个实例。通过关键字 **by** 对其后的表达式求值来获取这个对象。关键字 **by** 可以用于任何符合属性委托约定规则的对象。编译器会创建一个隐藏的辅助属性，并使用委托对象的实例进行初始化：

```kotlin
class Foo{
    private val delegate = Delegate()
    var p : Type
        get() = delegate.getValue()
        set(value : Type) = delegate.setValue(value)
}
```

前面变量的章节提到的延迟初始化，有一个方式是「by lazy」。lazy 函数返回的是一个对象，该对象具有一个名为 getValue 且签名正确的方法，因此可以把它与 by 关键字一起使用来创建一个委托属性。lazy 的参数是一个 Lambda，可以调用它来初始化这个值。默认情况下，lazy 函数是线程安全的。

```kotlin
class LateInit<T>(val block: () -> T) {
    var value: Any? = null
    operator fun getValue(any: Any?, prop: KProperty<*>): T {
        if (value == null) {
            value = block
        }
        return value as T
    }
}
```

## Kotlin 泛型

### 泛型类型参数

泛型是允许定义带**类型形参**的类型。但这种类型的实例被创建出来的时候，类型形参会被替换成称为**类型实参**的具体类型。Kotlin 始终要求类型实参要么被显式地的说明，要么能被编译器推导出来。

#### 声明泛型类

通过在类名后面加上一堆尖括号，并把类型参数放在尖括号内来声明泛型类及泛型接口。通过一个类继承了泛型类（或者实现了泛型接口），就需要为基础类的泛型形参提供一个类型实参，它可以是一个具体的类型或者另一个类型形参。

```kotlin
class MyClass<T> {
    fun methodA(param: T):T {
        return param
    }
}
```

#### 声明泛型函数和属性

声明泛型函数或者属性时，需要在函数名添加泛型声明。泛型声明是一对尖括号括起来的。

```kotlin
fun <T> List<T>.filter(predicate: (T) -> Boolean):List<T>

val <T> List<T>.penultimate: T
```

普通（即非扩展）属性不能拥有类型参数，不能在一个类的属性中存储多个不同类型的值，因此声明泛型非扩展函数没有任何意义。

#### 类型参数约束

类型参数约束可以限制作为泛型类和泛型函数的**类型实参**的类型。如果把一个类型指定为泛型**类型形参**的上界约束，在泛型类型具体的初始化中，对应的**类型实参**就必须是这个具体类型的子类型。

在泛型声明时，将冒号放在类型参数名称之后，作为类型形参上界的类型。如果类型上界有多个，可以使用 **where** 关键字来实现。

```kotlin
fun <T : Number> List<T>.sum(): T {}
fun <T> cut(t: T) where T : CharSequence, T : Appendable {}
```

### 运行时的泛型

#### 类型擦除

Kotlin 的泛型在运行时也会被擦除，这意味着泛型类实例不会携带用于创建它的类型实参的信息。

在运行时，如果想要检查一个实例是否是 List，可以使用星号投影语法：

```kotlin
if (value is List<*>){ ... }
```

但因为类型擦除的原因，并不能检查这个实例是 List\<String> 还是 List\<Int>。

#### 实化类型参数

由于类型擦除的存在，在调用泛型函数的时候，在函数体中不能调用它的类型实参：

```kotlin
fun <T> isT(value : Any) = value is T
Error: Cannot check for instance of erased type : T
```

如果想要上面这中判断正常运行，就需要将类型参数实化。实化类型参数可以使用 **inline** 内联函数和 **reifield** 关键字来完成。

使用 **inline** 关键字将一个函数标记为内联函数，编译器会把每一次函数调用都转换成函数实际的代码实现（如果该函数使用了 Lambda 实参，Lambda 的代码也会被内联，不会创建内部类），并且用 reifield 标记类型参数。

```kotlin
inline fun <reifield T> isT(value : Any) = value is T 
```

实化类型参数有一些限制，具体来说可以按下面的方式来使用实化类型参数

- 用在类型检查和类型转换中（is、!is、as、as?）
- 获取相应的 Class（javaClass，KClass）
- 作为调用其他函数的类型实参

不能做下面这些事情：

- 创建指定为类型参数的类的实例
- 调用类型参数类的伴生对象的方法
- 调用带实化类型参数函数的时候使用非实化类型形参作为类型形参
- 把类、属性或者非内联函数的类型参数标记为 reifield

### 变型

#### 类、类型和子类型

类和类型在非泛型类时可以当作相同概念来使用，但在泛型类中，这两者是由区别的。比如 List\<Number> 还是 List\<Int> 都是 List 类，但并不是相同的类型。

**子类型**：任何时候如果需要的是类型 A 的值，都可以使用类型 B 的值当作 A，类型 B 就是类型 A 的字类型。所以 Int 是 Number 的子类型，但 List\<Int> 不是 List\<Number> 的子类型，Int 还是 Int? 的子类型。**超类型**是子类型的反义词。

一个泛型类，比如 MutableList\<A> 既不是   MutableList\<B> 的子类型，也不是超类型，就称这个泛型类在该类型参数上是**不变型**的。

#### 协变和逆变

有了子类型和超类型的概念，就可以解释什么是协变性和逆变性。

一个**协变类**是一个泛型类（以 List\<T>为例）来说，如果 A 是 B 的子类型，那么List\<A> 是 List\<B> 的子类型，那么就说这个类在该类型参数上具有协变性。

一个**逆变类**是一个泛型类（以 List\<T>为例）来说，如果 A 是 B 的子类型，那么List\<B> 是 List\<A> 的子类型那么就说这个类在该类型参数上具有逆变性。

首先，对于 Java 来说，Java的泛型本身是不支持协变和逆变的，也就是**不变型**。但是可以通过通配符「? extends」和「? super」来解决这个问题：

- 可以使用泛型通配符`? extends`来使泛型支持协变，但是「只能读取不能写入」
- 可以使用泛型通配符`? super` 来使泛型支持逆变，但是「只能写入不能读取」

```java
List<? extends TextView> textViews = new ArrayList<Button>();
TextView textView = textViews.get(0); // OK
textViews.add(textView); //  add 会报错，no suitable method found for add(TextView)

List<? super Button> buttons = new ArrayList<TextView>();
Object object = buttons.get(0); // get 出来的是 Object 类型
buttons.add(button); // add 操作是可以的
```

和 Java 泛型一样，Kotlin 中的泛型本身也是**不变型**的，但是可以使用关键子「out」和「in」来解决这个问题：

- 在该类型参数的名称前加上 **out** 关键字，声明类在某个类型参数上是可以**协变**的，但是「只能读取不能写入」
- 在该类型参数的名称前加上 **in** 关键字，声明类在某个类型参数上是可以**逆变**的，但是「只能写入不能读取」

```kotlin
var textViews: List<out TextView> = List<Button>
var textViews: List<in TextView> = List<Button>
```

**out** 和 **in** 修饰符被称为**型变注解**。为了保证类型安全，out 修饰的参数只能出现在函数的返回值位置（out 位置），in 修饰的参数只能出现在函数的参数位置（in 位置）。

```kotlin
class Animal<in T : Number, out R : CharSequence>{
    fun create(index: T) {}
    fun get(): R {}
}
```

在类型参数上使用 **out** 关键字有两个含义：

- 子类型化被会被保留，让该泛型类在该类型参数上变成协变
- 该类型参数只能用在 out 位置上

在类型参数上使用 **in** 关键字有两个含义：

- 子类型化会被反转，让该泛型类在该类型参数上变成逆变
- 该类型参数只能用在 in 位置上，被这个方法消费

#### 声明处型变和类型投影

当 out/in 修饰符在一个类的类型参数处提供，则称之为**声明处型变**。

当  out/in 修饰符在一个函数参数的类型参数处提供，则称之为**类型投影**。

```kotlin
fun <T> copyDate(
    source: MutableList<T>,
    destination: MutableList<T>
) {
    for (item in source) {
        destination.add(item)
    }
}
```

由于 source 这个参数只会读取，而 destination 只会写入，Kotlin提供了更优化的表达方式，当函数的实现调用了那些类型参数只出现在 out/in 位置上的方法时，可以在函数定义中给特定用户的类型参数参数加上变形修饰符。

```kotlin
fun <T> copyDate(
    source: MutableList<out T>,
    destination: MutableList<in T>
) {
    for (item in source) {
        destination.add(item)
    }
}
```

#### 星号投影

星号投影用来表明：不知道泛型实参的任何信息。

例如一个包含位置类型的元素的列表可以使用 List\<\*> 来表示。Kotlin 中的`*` 相当于 `out Any`。而 Java 中单个 `?` 号也能作为泛型通配符使用，相当于 `? extends Object`

## Kotlin 反射和注解

### Kotlin 反射

#### KClass

Kotlin 反射 API 的主要入口是 KClass 它代表了一个类。KClass 对应的是 java.lang.class，Kotlin Kotlin 中获取 Class 的方式有：

```kotlin
// 通过对象获取
demo.javaClass //javaClass
demo.javaClass.kotlin // KClass
demo::class //KClass
demo::class.java // javaClass

// 通过类获取
Demo::class //KClass
Demo::class.java //javaClass
```

我们可以首先使用一个对象的 javaClass 获取它的 Java 类，然后访问该类的 .kotlin 扩展属性，从 Java 切换到 Kotlin 的反射 API。Kotlin 反射 API 定义在包 kotlin.reflect 中，它能让开发者访问 Java 中不存在的概念，比如属性和可空类型。

KClass 的特别属性或函数：

| 属性或函数名称                   | 含义                   |
| -------------------------------- | ---------------------- |
| isCompanion                      | 是否是伴生对象         |
| isData                           | 是否是数据类           |
| isSealed                         | 是否是密封类           |
| objectInstance                   | object 实例            |
| companionObjectInstance          | 伴生对象实例           |
| declareMemberExtensionFunctions  | 扩展函数               |
| declareMemberExtensionProperties | 扩展属性               |
| memberExtensionFunctions         | 本类或者超类的扩展函数 |
| memberExtensionProperties        | 本类或者超类的扩展属性 |

#### KCallable

查看 KClass 的声明，会发现由类的所有成员组成的列表是一个 kCallable 实例的集合。KCallable 是函数和属性的超接口，它声明了 call 方法，允许开发者调用对应的函数或者对应属性的 getter

#### KParameter

KCallable.parameters 可以获取一个 List\<KParameter>，他代表的是函数的的参数。

#### KFunction

KFunctionN 代表了不同数量参数的函数，每一个类型都继承了 KFunction 并加上一个额外的成员 invoke

#### KProperty

一个成员属性由 KProperty1 的实例来表示，KProperty1 是一个泛型类 KProperty1\<T1, T2>，第一个参数表示接收者的类型，第二个参数代表了属性的类型。这样就可以对正确类型的接收者调用它的 get 方法：KProperty1.get(T1)

### Kotlin 注解

Kotlin 中创建注解，需要在 class 前增加 **annotation** 关键字：

```kotlin
annotation class FooAnnotation(val bar: String)
```

注解的参数只能是常量，且仅支持以下的类型：

- 与 Java 对象的基本类型
- 字符串
- Class 对象（javaClass 和 KClass）
- 其他注解

Kotlin 的元注解有：

| Kotlin 元注解 | 含义                     |
| ------------- | ------------------------ |
| Documented    | 文档                     |
| Target        | 限定了注解使用的场景     |
| Retention     | 说明的注解的保留时间     |
| Repeatable    | 注解在同一个可以多次出现 |

## 协程

广义的协程是一种在程序中处理并发任务的方案，也就是这个方案的一个组件。它和线程是一个层级的概念，是一种和线程不同的并发任务解决方案。

Kotlin 中的协程和广义的协程不是一种东西，Kotlin 的协程实际上是一个线程框架，可以看做是 Java 的 Excutor 和 Rxjava，Android 的 AsyncTask、Handler 等是一个层级的东西。

Kotlin 中可以使用以下四种方式创建协程：

- runBlocking
- GlobalScope 单例对象直接调用 launch 开启协程
- 通过 CoroutineContext 创建一个 CoroutineScope 对象，再调用 launch 开启协程
- async/withContext

方法一适用于单元测试的场景，业务开发中不会用到这种方法，因为它是线程阻塞的。

方法二和使用 `runBlocking` 的区别在于不会阻塞线程。但在 Android 开发中同样不推荐这种用法，因为它的生命周期会和 app 一致，且不能取消。

方法三是比较推荐的使用方法，我们可以通过 `context` 参数去管理和控制协程的生命周期（这里的 `context` 和 Android 里的不是一个东西。

```kotlin
val job = Job()
val coroutineScope = CoroutineScope(job)
coroutineScope.launch(Dispatchers.IO) { delay(100) }

job.cancel()
```

### launch、async 和 withContext 

launch 和 async 都可以开启一个新的协程，但是 launch 只能用于执行一段逻辑，却不能获取执行的结果，因为它的返回值永远都是一个 Job 对象。async 函数可以创建一个协程，并获取它的执行结果。

async 函数必须在协程作用域中才能调用，它会创建一个新的协程并返回一个 Deferred 对象，如果想要获取 async 函数代码块的执行结果，只需要调用 Deferred 对象的 await() 方法。

```kotlin
 runBlocking {
    val deffred = async(Dispatchers.IO) { delay(100) }
    val result = deffred.await()
}
```

withContext 是一个挂起函数，可以理解为 async 函数的简化版，调用 withContext 后会立即执行代码块中的代码，同时将当前的协程阻塞住。

launch、async 和 withContext 都可以指定线程参数，常用的 `Dispatchers` ，有以下三种：

- `Dispatchers.Main`：Android 中的主线程
- `Dispatchers.IO`：针对磁盘和网络 IO 进行了优化，适合 IO 密集型的任务，比如：读写文件，操作数据库以及网络请求
- `Dispatchers.Default`：适合 CPU 密集型的任务，比如计算

### suspend

withContext 是一个挂起函数，也就是被 suspend 关键字修饰。suspend 不是用来切换线程的，用来标记和提醒，在编译阶段帮助生成切换线程的代码。suspend 关键字可以将任何函数声明为挂起函数，挂起函数只能在其他挂起函数或者协程作用域中被调用。

