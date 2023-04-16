以下是Gradle构建系统在Android项目中的一些关键概念：

1. Project：一个Project对应一个构建文件（如build.gradle）。它表示一个可以构建的实体，例如一个Android应用程序或库模块。一个Project可以包含多个子项目（subprojects），这些子项目也会有自己的构建文件。一个Android工程通常至少包含一个应用模块，它是一个Project。
2. Task：Task是Gradle构建系统中的最基本的单位，它表示一个具体的操作，例如编译Java代码、合并资源文件或生成APK。每个Task可以依赖其他Task，并在执行时按照定义的顺序执行。
3. Build.gradle：这是Gradle构建系统的配置文件。一个Android项目通常有两个build.gradle文件：一个位于项目根目录，称为项目级别（Project-level）的build.gradle；另一个位于应用模块（如app模块）目录下，称为模块级别（Module-level）的build.gradle。项目级别的build.gradle文件主要用于配置整个项目的构建设置，如Gradle插件的版本信息。模块级别的build.gradle文件主要用于配置具体模块的构建设置，如依赖库、最小SDK版本、目标SDK版本等。
4. Gradle Wrapper：Gradle Wrapper是一个可执行的脚本，它包装了Gradle构建工具，允许开发者在没有安装Gradle的情况下构建项目。Gradle Wrapper会在构建时自动下载和安装合适版本的Gradle。这确保了在不同开发环境下，项目都使用相同版本的Gradle构建。
5. Plugin：Gradle插件扩展了Gradle构建系统的功能。Android开发中最常用的插件是Android Gradle Plugin，它提供了许多与Android构建相关的特性，例如构建APK、生成签名、混淆代码等。
6. Dependency：依赖是一个项目所需的外部库或模块。在Android项目中，开发者通常需要添加许多依赖库，例如Android Support Library、Retrofit、Glide等。Gradle管理这些依赖，确保它们在构建过程中被正确地添加到项目中。

### Gradle 任务

#### build.gradle

每个 Project 都会有一个 Build 文件，该文件是该 Project 构建的入口，可以在这里针对该 Project 进行配置，比如配置版本，需要哪些插件，依赖哪些库等。

```groovy
subprojects{

}
allprojects{

}
```

#### task 任务

task 其实是 Project 对象的一个函数，原型是 create(String name, Closure configureClosure). 

#### 任务依赖

在创建任务的时候，通过 dependOn 可以指定其依赖的任务，一个任务可以同时依赖多个任务。

```
task task1{}
task task2(dependsOn: task1){}
task task3{
	dependsOn task1, task2
}
```

#### 任务创建方式

##### 以任务名字创建任务

```
def Task customTask = task(‘customTask’)
```

这种方式其实是调用 Project 对象中的 task 方法，该方法的完整定义是：

```groovy
Task task(String name) throws InvalidUserDataException
```

##### 以任务名字+任务配置 Map 对象创建任务

```
def Task customTask = task("customTask", group:BasePlugin.BUILD_GROUP)
```

相比较第一种方式，多了一个 Map 参数，用于对要创建的 Task 进行配置。该方法的原型是：

```
Task task(Map<String, ?> args, String name) throws InvalidUserDataException
```

例子中的为其指定了分组为 BUILD，除此之外还有以下配置：

|   配置项    |                   描述                   |   默认值    |
| :---------: | :--------------------------------------: | :---------: |
|    type     |  基于一个存在的 Task 来创建，和继承类似  | DefaultTask |
|  overwrite  | 是否替换存在的Task，这个和 type 配置使用 |    false    |
|  dependsOn  |            用于配置任务的依赖            |     []      |
|   action    |     添加到任务的一个 Action 或者闭包     |    null     |
| description |              配置任务的描述              |    null     |
|    group    |              配置任务的分组              |    null     |

##### 以任务名字+闭包配置创建任务

因为第二种 Map 参数配置的方式可配置项有限，可以使用闭包的方式进行更灵活的配置

```
task customTask {
	description 'demo'
	doLast {
		println 'do last action'
	}
}
```

##### TaskContainer 对象 create 方法创建任务

在 Gradle 里 Project 对象定义了一个 TaskContainer (tasks)。前三种方法最终都是调用它的 create 方法。create 方法的擦承诺书和 Project 的 Task 方法基本一样。

```
tasks.create("customTask2"){
	description 'customTask2 demo'
	doLast {
		println 'customTask2 do last action'
	}
}
```

#### 任务访问方式

##### 使用任务名访问

创建的任务都会作为项目 Project 的一个属性，属性名就是任务名，所以可以直接通过任务名称访问和操纵该任务。

##### TaskContainer 对象访问任务

任务都是由 TaskContainer 创建的，它是创建任务的集合，所以可以通过 tasks 来访问任务

```
task customTask{}

tasks['customTask'].doLast{}
```

##### TaskContainer 通过路径访问任务

第二种方式的实际上调用的 findByName(String name) 方法。除了通过名称访问，还可以通过路径访问任务。通过路径访问任务有两种方式：get 和 find。两者的区别是：get 如果找不到会抛出 UnknownTaskException 异常，find 如果找不到会返回 null。

```
tasks.findByPath(':exampleProject:customTask')
tasks.getByPath(':exampleProject:customTask')
```

#### 任务执行分析

当执行一个 Task 的时候，其实就是执行其拥有的 actions 列表，这个列表保存在 Task 对象实例中的 actions 成员变量中，类型是一个 List：

```groovy
private List<ContextAwareTaskAction> actions = new ArrayList<ContextAwareTaskAction>
```

Task 之前执行、本身执行以及 Task 之后执行分别称为 doFirst、doSelf 以及 doLast：

```
class CustomTask extends DefaultTask {
    @TaskAction
    doSelf() {
        println 'doSelf'
    }
}
task hello(type: CustomTask) {
    doLast {
        println 'doLast'
    }
    doFirst {
        println 'doFirst'
    }
}

> Task :hello
doFirst
doSelf
doLast
```

声明的 doSelf 方法被 TaskAction 注解标注，表示该方法就是 Task 本身要执行的方法。Gradle 会解析带 TaskAction 标注的方法，然后通过 Task 的 prependParallelSafeAction 方法把该 Action 添加到 actions 列表中。

#### 任务排序

任务的排序功能是通过 shouldRunAfter 和 mustRunAfter 两个方法来控制一个任务应该或者必须在某个任务之后执行。

shouldRunAfter 表示应该，所以有可能任务顺序并不会按照预设的执行。

mustRunAfter 表示必须，所以任务会按照预设的执行。

```
task world(type: CustomTask) {
    doLast {
        println 'world doLast'
    }
    doFirst {
        println 'world doFirst'
    }
}
task hello(type: CustomTask) {
    doLast {
        println 'hello doLast'
    }
    doFirst {
        println 'hello doFirst'
    }
}
world.mustRunAfter(hello)
```

#### 任务的启用和禁用

Task 有一个 enabled 属性，用于启用和禁用任务，默认是 true，表示启用；设置为 false 则禁止该任务执行。输出会提示该任务被跳过。

#### 任务的 onlyIf 断言

Task 有一个 onlyIf 方法，接受一个闭包作为参数，如果改闭包返回 true，则该任务执行，否则跳过。

### Gradle 插件

#### 插件类型

插件的应用都是通过 Project.apply 方法完成的， apply 方法有好几种用法，插件也分为二进制插件和脚本插件。

##### 应用二进制插件

二进制插件是实现了 org.gradle.api.Plugin 接口的插件，它可以有 plugin id。比如：

```
apply plugin:‘java’
```

就把 Java 插件应用到项目中，其中 'java' 是 Java 插件的 plugin id，它对应的类型是 org.gradle.api.plugins.JavaPlugin，所以也可以：

```
apply plugin:org.gradle.api.plugins.JavaPlugin
```

又因为包 org.gradle.api.plugins 是默认导入的，所以可以取消包名，写为：

```
apply plugin:JavaPlugin
```

##### 应用脚本插件

应用脚本插件就是把这个脚本加载进来，它使用的关键字是 from，后面紧跟一个脚本文件，可以是本地文件，也可以是网络存在的，网络存在的需要使用 HTTP URL。

```groovy
// build.gradle
apply from:'version.gradle'
task customTask{
	doLast{
		println "app version is ${versionName}"
	}
}

// version.gradle
ext{
	versionName = '1.0'
}
```

##### 应用第三方发布的插件

在应用第三方发布的作为 jar 的二进制插件，必须先在 buildscript{} 里配置 classpath 才能使用，比如 Android Gradle 插件就属于 Android 发布的第三方插件：

```
buildscript{
	repositories{
		jcenter()
	}
	dependencies{
		classpath 'com.android.tools.build:gradle:1.5.0'
	}
}
```

buildscript{} 块是在一个构建项目之前，为项目进行前期准备和初始化相关配置依赖的地方，配置好所需要的依赖就可以应用插件了：

```
apply plugin:'com.android.application'
```

#### Java Gradle 插件

##### SourceSet 

SourceSet 是源代码集合，简称源集。是 Java 插件用来描述和管理源代码及其资源的集合。通过它可以设置源集的属性，更改源集的 Java 目录或者资源目录。

Java 插件在 Project 下提供了一个 SourceSets 属性以及一个 SourceSets{} 闭包来访问和配置。

| 属性名              | 类型               | 描述                        |
| ------------------- | ------------------ | --------------------------- |
| name                | String             | 只读，比如main              |
| output.classesDir   | File               | 源集编译后的 class 文件目录 |
| output.resourcesDir | File               | 编译后生成的资源目录        |
| compileClasspath    | FileCollection     | 编译是源集所需的 classpath  |
| java                | SourceDirectorySet | 源集的 Java 源文件          |
| java.srcDirs        | Set                | 源集的 java 源文件所在目录  |
| resources           | SourceDirectorySet | 源集的资源文件              |
| resource.srcDirs    | Set                | 源集的资源文件所在目录      |

```groovy
sourceSets{
	main{
		resources{
			srcDir 'src/resources'
		}
	}
	vip{
	
	}
}
```

这样就可以修改资源文件的存放目录和定义新的源集。

##### Java 插件添加的任务和属性

Java 插件添加的任务

| 任务名称            | 类型        | 描述                                        |
| ------------------- | ----------- | ------------------------------------------- |
| compileJava         | JavaCompile | 使用 javac 编译 Java 源文件                 |
| processResource     | Copy        | 把资源文件复制到生成的资源文件目录里        |
| classes             | Task        | 组装产生的类和资源文件目录                  |
| compileTestJava     | JavaCompile | 使用 javac 编译测试 Java 源文件             |
| processTestResource | Copy        | 把测试资源文件复制到生成的资源文件目录里    |
| testClasses         | Task        | 组装产生的测试类和测试资源文件目录          |
| jar                 | Jar         | 组装 Jar 文件                               |
| javadoc             | JavaDoc     | 使用 javadoc 生成 Java API 文档             |
| uploadArchives      | Upload      | 上传包含 Jar 的构建，用 archives{} 闭包配置 |
| clean               | Delete      | 清理构建生成目录文件                        |
| cleanTaskName       | Delete      | 删除指定任务生成的文件                      |

java 插件添加在 Project 中的属性

| 属性名              | 类型               | 描述                             |
| ------------------- | ------------------ | -------------------------------- |
| sourceSets          | SourceSetContainer | Java 项目的源集                  |
| sourceCompatibility | JavaVersion        | 编译 Java 源文件使用的 Java 版本 |
| targetCompatibility | JavaVersion        | 编译生成的类的 Java 版本         |
| archivesBaseName    | String             | 打包生成的 Jar 或者 Zip 文件名称 |
| libsDir             | File               | 存放生成的类库目录               |
| distsDir            | File               | 存放生成的发布的文件的目录       |

#### Android Gradle 插件 build.gradle 配置

Android Gradle 插件继承于 Java 插件，可以根据 Android 工程的属性分为三类：

- App plugin id：com.android.application
- Library plugin id：com.android.library
- Test plugin id：com.android.test

通过以上三种不同的插件，可以配置 Android 工程是 Android App 工程，还是 Library 工程还是测试功能。

Android Gradle 的build.gradle 配置文件

```groovy
apply plugin:'com.android.application'

android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
	
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
	}
	
	buildType{
		release{
			minifyEnable false
			proguardFiles getDefaultProguardFile('proguard-android.txt'),'proguard-rules.pro'
		}
	}
}

dependencires{
	compie fileTree(dir:'libs',include:['*.jar'])
	testCompile 'junit:junit:4.12'
	compile 'com.android.supoort:appcompat-v27:23.1.1'
}
```

Android Gradle 功能的配置都是在 android{} 中，这里是唯一的入口，它的具体实现是 com.android.build.gradle.AppExtension，是 Project 的一个扩展。

##### compileSdkVersion

是配置编译 Android 工程的 SDK 版本，该配置的原型是 compileSdkVersion(int apiLevel) 方法。还有一个同名方法是 compileSdkVersion(String version) ，除此之前它还有一个 set 方法，所以可以把它当做 Android 的一个属性使用。所以 Build 脚本配置还可以写成：

```groovy
android{
	compileSdkVersion 'android-23'
	...
}

android.compileSdkVersion = 23
//or
android.compileSdkVersion = 'android-23'
```

##### buildToolsVersion

是 Android 构建工具的版本，是一个工具包，包括 appt、dex 。可以通过该方法赋值，也可以通过 android.buildToolsVersion 这个属性读写它的值。

#### Android Gradle 插件 defaultConfig 配置

defaultConfig 是默认的配置，它是一个 ProductFlavor，ProductFlavor 允许根据不同的情况生成多个不同的 APK 包。

##### applicationId 

applicationId 是配置的包名，是 ProductFlavor 的一个属性，默认情况是 null。这时候在构建时会从 AndroidManifest.xml 文件中读取 manifest 标签的 package 属性值。

##### minSdkVersion 

minSdkVersion 是最新支持的 Android 系统的 API Level

##### targetSdkVersion  

targetSdkVersion 是基于哪个 Android 版本开发的

##### versionCode 

versionCode 是 App 应用内部版本号，一般用于控制 App 升级。没有配置的时候会从 AndroidManifest.xml 文件中读取

##### versionName 

versionName 是App 应用的版本名称，用户可以看到

##### testApplicationId

用于配置测试 App 的包名，默认情况是 applicationId+".test"

##### signingConfig

配置默认的签名信息，对生成的 App 前面，它是一个 SigningConfig，也是 ProductFlavor 的一个属性，可以对其直接进行配置。

#### Android Gradle 插件签名配置

```groovy
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
	
	signingConfig{
		release{
			storeFile file("releasekey.keystore")
			storePassword "MypPssword"
			keyAlias "myReleaseKey"
			keyPassword "MypPssword"
		}
	}
	
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
}
```

Android Gradle 提供了 signingConfig{} 配置块便于生成多个签名配置信息。然后在 defaultConfig 中对签名配置应用。

- storeFile 签名证书文件
- storePassword 签名证书文件的密码
- storeType 前面证书的类型
- keyAlias 签名证书中的密钥别名
- keyPassword 签名证书中该秘钥的密码

#### Android Gradle 插件 buildTypes 配置

buildTypes 是 NamedDomainObjectContainer 类型是一个域对象，和 SourceSet 类似，它默认含有 release、debug 两个构建类型，这两种模式到的主要差别在于能否在设备上调试以及签名不一样。 buildTypes{} 中可以新增任意多个我们需要构建的类型。

##### applicationIdSuffix

applicationIdSuffix 是 BuildType 的一个属性，用于配置基于默认 applicationId 的后缀。比如在 debug 的 BuildType 中指定 applicationIdSuffix 为 .debug。那么生成的包名就是 applicationId+".debug"。

##### debugable

是 BuildType 的一个属性，用于配置是否生成一个可调试的 apk。

##### jniDebugable

是 BuildType 的一个属性，用于配置是否生成一个可调试 Jni 代码的 apk。

##### minifyEnabled

是 BuildType 的一个属性，是否为该构建类型启用混淆。

##### multiDexEnabled

是 BuildType 的一个属性，用于配置该 BuildType 是否启用自动拆分多个 Dex 的功能。

##### proguardFile

是 BuildType 的一个方法，用于配置 Proguard 混淆使用的配置文件

##### proguardFiles

是 BuildType 的一个方法，它接受一个可变参数，可以同时配置多个 Proguard 混淆使用的配置文件。`getDefaultProguardFile`是 Android 扩展的一个方法，它可以获取 Android SDK 目录下的默认的 proguard 配置文件（文件名称是传入的参数：proguard-android.txt）

##### shrinkResources

是 BuildType 的一个属性，用于配置是否自动清理未使用的资源，默认为 false。

##### zipAlignEnabled

是 BuildType 的一个属性，用于配置是否开启优化 apk 的工具，提高系统和应用的运行效率。

### Android Gradle 使用

#### Android Gradle 自定义 

##### 使用共享库

Android 的包（比如 android.app、android.view等）是默认包含在 Android SDK 库里的，所有的应用都可以直接使用它们。还有一些库（比如 com.google.android.maps）是独立的，并不会被系统自动链接，如果需要使用的话，需要单独进行生成使用，这类库被称为共享库。

在 AndroidManifest.xm 中可以指定要使用的库：

```
<uses-library 
	android:name="com.google.android.maps"
	android:required="true"/>
```

声明后，在安装 apk 时，系统会根据我们定义来检测手机系统中是否含有我们需要的共享库。因为设置的是 `android:required="true"`，如果手机系统不满足，将不能安装该应用。

##### 批量修改生成的 apk 文件名

修改生成的 apk 文件名，就要修改 Android Gradle 打包的输出，android 对象提供了三个属性：applicationVariant、libraryVariant 和 testVariant，这三个属性返回的都是  DomainObjectSet 对象集合。

```groovy
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	applicationVariants.all { variant ->
        variant.outputs.each { output ->
            if (output.outputFile != null && output.outputFile.name.endsWith('.apk')) {
                output.outputFileName = "demo_${variant.buildType.name}_v${variant.versionName}_${buildTime()}.apk"
            }
        }
    }
}

def buildTime() {
    def date = new Date()
    def formattedDate = date.format('MMddHHmm')
    return formattedDate
}
```

applicationVariants 中的 variant 都是 ApplicationVariant，它有一个 outputs 属性是 List 集合作为它的输出，再遍历 List 找到 .apk 结尾的文件修改为想要的文件名。

##### 动态配置 AndroidManifest.xml 文件

在构建过程中，动态修改 AndroidManifest.xml 文件中的一些内容，提供了 manifestPlaceholder、Manifest占位符等工具。

ManifestPlaceholders 是 ProductFlavor 的一个属性，是一个 Map 类型，所以可以同时配置多个占位符。

```
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	productFlavors{
		google{}
		baidu{}
	}
	
	productFlavors.all{ flavor ->
		manifestPlaceholders.put("key",value)
	}
}
```

##### 自定义 BuildConfig

BuildConfig 是由 Android Gradle 构建脚本在编译后生成的。Android Gradle 提供了 `buildConfigField(String type, String name, String value)`方法添加自定义常量到 BuildConfig 中。

```groovy
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	productFlavors{
		google{
			buildConfigField 'String','WEB_URL','"http://www.google.com"'
		}
		baidu{
			buildConfigField 'String','WEB_URL','"http://www.baidu.com"'
		}
	}
}
```

以上是渠道（ProductFlavor）自定义常量，构建类型（BuildType）也可以配置。

##### 动态添加自定义的资源

在 BuildType 和 ProductFlavor 两个对象中都存在 resValue 方法，来自定义资源。它的方法原型是 `resValue(String type, String name, String value)`。它会添加生成一个资源，效果和在 res/values 文件中定义一个资源是等价的。

```groovy
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	productFlavors{
		google{
			resValue 'String','channel','google channel'
		}
		baidu{
			resValue 'String','channel','baidu channel'
		}
	}
}
```

##### Java 编译选项

android 对象提供了 compileOptions 方法，来对 Java 编译选项进行配置，它接受一个 CompileOption 类型的闭包作为参数。

CompileOption 是编译配置，它提供了三个属性，分别是 encoding、sourceCompatibility、targetCompatibility，通过对他们进行设置来设置 Java 的编码、源码版本、编译版本等。

```groovy
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	compileOptions{
		encoding = 'utf-8'
		sourceCompatibility = JavaVersion.VERSION_1_8
		targetCompatibility = JavaVersion.VERSION_1_8
	}
}
```

```groovy
public void compileOptions(Action<CompileOptions> action){
	checkWritablity();
	action.execute(compileOptions)
}
```

compileOptions 

#### Android Gradle 多项目构建

定义一个工程，包含多个项目，需要在 setting.gradle 里配置好这些项目。 setting.gradle 是 Gradle 中 Settings 文件的默认名，用于初始化以及工程树的配置。大多数的作用是为了配置子工程，一个子工程是有在配置文件中才会被识别，在构建时才会被包含

```groovy
include 'sub'
project(':sub').projectDir = new File(rootDir, 'sub/subsub')
```

在多项目配置好后，就要引用它们，尤其是库项目。 Android 库项目引用和 Gradle 的其他引用是一样的，都是通过 dependencies 实现：

```groovy
dependencies {
	compile project(':sub:subsub')
}
```

Android App 项目不仅可以引用 Android Lib 项目，还可以有引用 Java Lib 项目。Android Lib 是打包成一个 aar 包，Java Lib 打包成一个 jar 包，如果包里有资源就是用 Android Lib，如果没有并且是纯 Java 的程序就可以考虑用 Java Lib。

Android 库项目发布出来的 aar 包都是 release 版本，可以通过配置来改变：

```groovy
android{
	defaultPushlishConfig "debug"
}
```

#### Android Gradle 多渠道构建

Adnroid Gradle 中定义了一个叫 Build Variant 的构建变体的概念，一个 BuildVariant = BuildType + ProductFlavor，BuildType 是构建类型（debug、release），ProductFlavor 是构建渠道。

```groovy
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	productFlavors{
		google{
			
		}
		baidu{
			
		}
	}
}
```

##### 多渠道 applicationId、versionCode、versionName  构建定制

applicationId、versionCode、versionName 是 ProductFlavor 的属性，用于设置该渠道的包名等属性。

```
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	productFlavors{
		google{
			applicationId ‘org.demo.example.google'
			versionCode 11
			versionName "1.1"
		}
		baidu{
			applicationId ‘org.demo.example.baidu
			versionCode 21
			versionName "2.1"
		}
	}
}
```

##### 多渠道 dimension 构建定制

dimension 是 ProductFlavor 的一个属性，接受一个字符串作为该 ProductFlavor 的维度，可以理解为对 ProductFlavor 进行分组。而 dimension 接受的参数就是分组的组名，也是维度名称。

在使用维度名称之前，必须先使用 android 对象的 flavorDimensions 方法声明。

```groovy
android{
	compileSdkVersion 23
	buildToolsVersion "23.0.1"
		
	defaultConfig{
		applicationId "org.demo.example"
		minSdkVersion 14
		targetSdkVersion 23
		versionCode 1
		versionName "1.0"
		signingConfig signingConfig.debug
	}
	
	flavorDimensions 'abi','type'
	
	productFlavors{
		free {
			dimension 'type'
		}
		paid {
			dimension 'type'
		}
		x86 {
			dimension 'abi'
		}
		arm {
			dimension 'abi'
		}
	}
}
```

