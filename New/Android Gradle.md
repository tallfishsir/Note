# Android Gradle

## Gradle 配置文件

在 Android 项目中，Gradle 构建系统使用了一系列的配置文件来管理项目的构建过程，包含了：

- (Project) build.gradle：位于项目根目录下，定义了整个项目的构建设置
- (Module) build.gradle：位于每个模块的根目录下。定义了模块的构建设置
- settings.gradle：位于项目根目录下。负责配置项目的基本设置
- gradle.properties：位于项目根目录下，用于配置项目的全局 Gradle 属性
- local.properties：位于项目根目录下，包含本地环境相关的设置

### (Project) build.gradle

定义了整个项目的构建设置，比如项目中使用的 Gradle 插件和版本，全局变量以及所有模块共享的依赖库仓库。

#### buildscript

配置构建脚本本身的设置，例如例如插件和依赖库的版本。

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:7.2.2'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
```

#### allprojects

配置项目中的所有模块的通用设置，可以在这里配置项目中所有模块共享的仓库。

```groovy
allprojects {
    repositories {
        google()
        mavenCentral()
    }
}
```

#### subprojects

配置所有子项目的通用设置。这可以用于在子项目中应用通用的插件、依赖项或其他设置。

```groovy
subprojects {
    apply plugin: 'java-library'
    dependencies {
        implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    }
}
```

#### configurations

用于配置依赖项的不同配置，可以使用它来更改依赖项解析的行为，例如排除特定的依赖项或更改依赖项的版本。

```groovy
configurations.all {
    resolutionStrategy {
        force 'com.google.code.findbugs:jsr305:3.0.1'
    }
}
```

#### ext

定义全局变量，例如依赖库的版本。这些全局变量可以在项目中的所有 build.gradle 文件中使用。

```groovy
ext {
    kotlin_version = '1.5.21'
    supportLibVersion = '28.0.0'
}
```

#### task

定义项目级别的自定义任务。这可以用于编写特定于整个项目的构建逻辑。

```groovy
task clean(type: Delete) {
    delete rootProject.buildDir
}
```

### (Module) build.gradle

定义了模块的构建设置，例如编译和目标 SDK 版本，构建类型（如Debug和Release），依赖项以及签名设置等。

#### apply plugin

引入 Gradle Plugin，为模块提供构建和编译的功能，Android 项目常用的 Plugin 有：

- com.android.application

  配置项目是 Android App 项目，为构建 Android 应用程序提供了必要的功能，如编译、打包、签名等。是创建 Android 应用程序的项目所必需的插件。

- com.android.library

  配置项目是 Android 库项目，库项目可以被其他Android应用程序或库作为依赖项使用，但不生成可安装的APK。

- com.android.test

  配置项目是测试项目

- kotlin-android

  使项目能够识别和编译 Kotlin 代码，并允许同一个模块中进行 Kotlin 和 Java 混合开发，并支持增量编译，提高构建效率。

```groovy
apply plugin: 'com.android.application'
// apply plugin: 'com.android.library'
apply plugin: 'kotlin-android'
```

#### android

配置 Android 项目的构建设置，例如编译 SDK 版本、构建工具版本和构建变体，常见的设置有：

- compileSdkVersion

  编译 Android 项目的 SDK  版本

- buildToolsVersion

  构建工具包的版本号，包括 appt，dex 这些工具

- defaultConfig

  构建 apk 的默认配置

  - applicationId：配置 apk 的包名
  - minSdkVersion：最低支持的 Android 系统 API Level
  - targetSdkVersion：基于哪个 Android 系统 API Level 开发，可以理解为最高支持
  - versionCode：App 的内部版本号，用于控制  App 升级
  - versionName：App 的版本名称，用户可以看到

- SourceSets

  修改或者添加新的代码目录和资源文件，java.srcDirs 表示代码目录，res.srcDirs 表示资源文件目录

- signingConfigs

  配置默认的签名信息，可以创建多个签名配置。然后在 buildType 中使用 signingConfigs

  - storeFile：签名证书文件
  - storePassword：签名证书文件的密码
  - storeType：签名证书的类型
  - keyAlias：签名证书中的密钥别名
  - keyPassword：签名证书中该秘钥的密码

- buildType

  默认包含 release 和 debug 两种构建类型，他们的差别在于能否在设备上调试和签名的不同。可以创建多个需要的构件类型

  - applicationIdSuffix：生成的 Apk 的包名是在 applicationId 对应的包名后再添加后缀
  - debugable：生成的 Apk 是否可调试
  - jniDebugable：生成的 Apk 是否可调试 Jni 代码
  - minifyEnable：生成的 Apk 是否启用混淆
  - proguardFile：开启混淆后使用的混淆配置文件
  - multiDexEnabled：生成的 Apk 是否自动拆分多个 Dex 
  - shrinkResources：是否自动清理未使用的资源
  - zipAlignEnabled：是否开启优化 apk 的工具，提高系统和应用的运行效率

- productFlavors

  配置应用生成不同的 apk，需要配合 flavorDimensions 使用

- compileOptions

  配置 Java 编译选项包括 Java 的编码、源码版本、编译版本等

- kotlinOptions

  配置Kotlin编译选项，例如语言特性和JVM目标版本

```groovy
android {
    compileSdkVersion 33
    buildToolsVersion "31.0.0"
    viewBinding {
        enabled = true
    }
    defaultConfig {
        applicationId "com.tallfish.androiddevelopdemo"
        minSdkVersion 21
        targetSdkVersion 33
        versionCode 1
        versionName "1.0"
    }
    sourceSets {
        main {
            java.srcDirs = ['src/main/java', 'src/shared/java']
            res.srcDirs = ['src/main/res', 'src/shared/res']
        }
        myCustomSourceSet {
            java.srcDirs = ['src/myCustomSourceSet/java']
            res.srcDirs = ['src/myCustomSourceSet/res']
        }
    }
    signingConfigs {
        release {
            storeFile file("mykeystore.jks")
            storePassword "mykeystorepassword"
            keyAlias "mykeyalias"
            keyPassword "mykeypassword"
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    flavorDimensions 'paying', 'nation'
    productFlavors {
        free {
            dimension 'paying'
        }
        paid {
            dimension 'paying'
        }
        china {
            dimension 'nation'
        }
        global {
            dimension 'nation'
        }
    }
    compileOptions {
        encoding = 'utf-8'
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
    }
}
```

#### dependencies

配置模块的依赖，例如第三方库、Android支持库和项目中其他模块的依赖。

```groovy
dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation 'androidx.appcompat:appcompat:1.3.1'
    implementation 'androidx.core:core-ktx:1.6.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.0'
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.3'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'
}
```

在 Gradle 中，有多种依赖配置可以用于声明依赖库，以下是常用的依赖配置：

- implementation：应用程序运行时和编译时都需要这些依赖库，依赖库不会传递给依赖该模块的其他模块
- api：应用程序运行时和编译时都需要这些依赖库，依赖库会传递给依赖该模块的其他模块
- compile：旧的依赖配置，功能与 api 一致
- testImplementation：仅在单元测试编译和运行时需要这些依赖库，不会被打包到最终的APK中
- androidTestImplementation：仅在 Android 测试编译和运行时需要这些依赖库，不会被打包到最终的APK中
- compileOnly：仅在编译时需要这些依赖库，运行时不需要这些库，不会被打包到最终的APK中，常用于在编译时提供注解处理器的库
- runtimeOnly：仅在运行时需要这些依赖库，编译时不需要这些库，因此它们不会影响编译类路径。常用于仅在运行时提供功能的库

#### configurations

在两个或更多的库依赖于不同版本的相同库时，可能发生依赖冲突，可以使用这个配置来解决冲突：

- 强制使用特定版本

  强制 Gradle 使用特定版本的冲突库，使用 resolutionStrategy 实现

  ```groovy
  configurations.all {
      resolutionStrategy {
          force 'com.example.library:library-name:1.0.0'
      }
  }
  ```

- 排除依赖

  某个库依赖了不需要的库，使用 exclude来排除它

  ```groovy
  configurations {
      implementation.exclude group: 'com.example', module: 'unnecessary-library'
  }
  ```

### settings.gradle

负责配置项目的基本设置，使用 include 指令将每个模块添加到项目中，以便Gradle构建系统能够识别它们。

#### include

用于声明项目中的模块，只有使用 clude 添加到项目中，对应的模块才能被 Gradle 识别并参与到构建过程中。

```groovy
include ':app'
include ':module1', ':module2'
```

#### projectDir

设置模块的项目目录，常用于模块目录结构和默认值不同的情况。

```groovy
include ':module1'
project(':module1').projectDir = new File(rootDir, 'custom-directory/module1')
```

### gradle.properties

配置项目的全局 Gradle 属性，例如 JVM 堆大小、代理设置和特性开关。这些属性对整个项目的构建过程都有效。

#### org.gradle.jvmargs

设置用于运行 Gradle 的 JVM 的参数。例如，设置 org.gradle.jvmargs=-Xmx2048m 来增加JVM的最大内存分配量，加快大型项目的构建速度。

#### org.gradle.parallel

设置 Gradle 是否在构建过程中并行执行任务，开启并行构建可以显著提高构建速度。

#### org.gradle.configureondemand

设置 Gradle 是否在构建过程中按需配置项目，可以在构建过程中跳过不需要构建的模块，从而提高构建速度。

#### org.gradle.caching

设置 Gradle 是否启用构建缓存，构建缓存可以在多次构建之间重用构建结果，从而提高构建速度。

#### android.useAndroidX

设置项目是否使用 AndroidX 库，AndroidX是支持库的新版本，使用了新的命名空间和功能。

#### android.enableJetifier

设置是否启用 Jetifier，在构建过程中将旧版 Android 支持库自动转换为对应的 AndroidX 库的工具。

```properties
org.gradle.jvmargs=-Xmx2048M -Dkotlin.daemon.jvm.options\="-Xmx2048M"
org.gradle.parallel=true
org.gradle.configureondemand=true
org.gradle.caching=true
android.useAndroidX=true
android.enableJetifier=true
```

### local.properties

配置本地环境相关的设置，例如 Android SDK 路径和 NDK 路径。这个文件通常不会被添加到版本控制系统中，因为它包含了特定于开发者环境的配置。

#### sdk.dir

指定 Android SDK 的路径。Gradle 构建系统需要知道 Android SDK 的位置，以便在构建过程中使用 Android SDK 中的工具和资源。

#### ndk.dir

指定 Android NDK 的路径，NDK 用于在 Android 项目中开发本地（C/C++）代码。

#### flutter.sdk

指定 Flutter SDK 的路径

```properties
sdk.dir=C\:\\Software\\application\\Android\\AndroidSDK
ndk.dir=/Users/username/Library/Android/sdk/ndk/21.0.6113669
flutter.sdk=/Users/username/development/flutter
```

## Gradle Plugin

在 Android Gradle 构建流程中，Project 用于表示构建过程中的一个独立构建单元。一个 Android 项目可能包括一个 App 模块和多个库模块，每个模块都被视为一个独立的 Project。

在 Project 的 buildscript 中都会通过 apply 来引入不同的 Plugin 来帮助编译和打包，Plugin 可以添加 Task  和 Extensions 来提供了额外的功能和自定义行为，扩展和定制构建过程。

### Task

Task 代表了构建过程中的要执行的单独工作单元，可以执行各种操作，例如编译源代码、打包资源、生成 APK、运行测试等。每个 Task 都有唯一的名称，在构建过程中可以通过该名称引用 Task。

#### 创建

Task 最常见的创建方式是 Name+Closure，其中 Closure 中可以配置：

- type：继承一个已存在的 Task
- overwrite：替换一个已存在的 Task
- dependsOn：依赖一个已存在的 Task，之前当前 Task 之前需要先执行依赖的 Task
- description：设置 Task 的功能描述
- group：设置 Task 的分组
- from/into：设置 Task 的输入输出
- enable：设置 Task 的启用和禁用，禁用后 Task 会被跳过
- doFirst：Task 执行开始之前的 Closure
- doLast：Task 执行结束后的 Closure

```groovy
task copyFiles(type: Copy) {
    from 'src/main/resources'
    into 'build/outputs'
}

task myTask(dependsOn: Delete) {
    group = 'Custom Tasks'
    description = 'This task prints a message.'
    doFirst：Task  {
        println 'doFirst：Task '
    }
    doLast {
        println 'doLast'
    }
}
```

#### 执行

当执行一个 Task 的时候，其实就是执行其拥有的 actions 列表，这个列表保存在 Task 对象实例中的 actions 成员变量中，类型是一个 List：

```groovy
private List<ContextAwareTaskAction> actions = new ArrayList<ContextAwareTaskAction>
    
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

需要注意的是，除了 doFIrst/doSelf/doLast，其他在 Closure 中的语句都属于配置，运行在 Configuration 阶段。

### Android Plugin

Plugin：Gradle插件扩展了Gradle构建系统的功能。Android开发中最常用的插件是Android Gradle Plugin，它提供了许多与Android构建相关的特性，例如构建APK、生成签名、混淆代码等

#### 类型

Plugin 分为二进制 Plugin 和脚本 Plugin。

二进制 Plugin 实现了 org.gradle.api.Plugin 接口，有 Plugin ID。比如 'com.android.application' 对应的类型是 com.android.build.gradle.AppPlugin。

```
apply plugin: 'com.android.application'
```

脚本 Plugin 加载的关键字是 from，后面紧跟一个脚本文件，可以是本地文件，也可以是网络存在的，网络存在的需要使用 HTTP URL。

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

#### Extensions

Extensions 用于向构建过程中添加自定义功能和配置。扩展允许 Gradle 构建脚本的行为，并在构建过程中使用自定义属性和方法。

buildscript 中的 android 配置，实际行就是在 AppPlugin 中注册的 Extensions，它的具体实现是 com.android.build.gradle.AppExtension。在 AppPlugin 中通过 create() 创建和注册 Extensions 过程核心代码就是：

```
class AppPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        project.extensions.create('android', AppExtension)
    }
}
```

在 android Extension中有一些修改，可以增强功能：

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

BuildConfig 是由 Android Gradle 构建脚本在编译后生成的。Android Gradle 提供了 buildConfigField(String type, String name, String value)方法添加自定义常量到 BuildConfig 中。

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

在 BuildType 和 ProductFlavor 两个对象中都存在 resValue 方法，来自定义资源。它的方法原型是 resValue(String type, String name, String value)。它会添加生成一个资源，效果和在 res/values 文件中定义一个资源是等价的。

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

### 自定义 Plugin

如果想实现自定义 Plugin，需要完成以下步骤。

- 新建一个 Module，并且必须命名为 buildSrc，并检查 Settings.gradle 没有 include '':buildScr'

- 在 buildScr 的 build.gradle 中添加依赖同项目级 build.gradle ，防止无法找到 Plugin 类

  ```groovy
  repositories {
      google()
      mavenCentral()
  }
  dependencies {
      classpath 'com.android.tools.build:gradle:7.2.2'
      classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:1.8.0"
  }
  ```

- 创建 Extension 文件，并把文件改为 .groovy 格式

  ```groovy
  class TallFishExtensions {
      def name = "123"
  }
  ```

  

- 创建 Plugin 子类，实现 apply()，并把文件改为 .groovy 格式

  ```groovy
  class TallFishPlugin implements Plugin<Project> {
      @Override
      void apply(Project project) {
          def extension = project.extensions.create("tallfish", TallFishExtensions)
          project.afterEvaluate {
              println "hello plugin extension : $extension.name"
          }
      }
  }
  ```

- 在 buildScr 的 main 文件夹下按照层级创建文件夹 resource/META-INF/gradle-plugin

- 在 gradle-plugin 文件夹下创建 com.tallfish.properties 文件，com.tallfish 就是在 App buildscript 中 apply 的字段，内容填写 TallFishPlugin 的全路径

  ```properties
  implementation-class=com.tallfish.plugin.TallFishPlugin
  ```

- 在 App 的 buildscript 中引入自定义 Plugin

  ```groovy
  apply plugin: 'com.tallfish'
  tallfish {
      name 'tall-fish'
  }
  ```

## Android 构建流程

Android 项目的 Gradle 构建流程，包括了初始化阶段、配置阶段和执行阶段。

### 初始化阶段

Gradle 根据 settings.gradle 文件中的 include 配置，确定了包含哪些模块。如果项目内还有 buildSrc 这个特殊模块，会将 buildSrc 模块添加到所有 include 模块的最前面。

### 配置阶段

Gradle 解析并执行所有项目和子项目的 build.gradle 文件，生成有向无环图：

- 解析 Project build.gradle 文件。此文件位于项目根目录中，定义了全局配置和项目级别的依赖项
- 解析 Module build.gradle 文件，这些文件位于每个模块的根目录中，定义了模块的特定设置和依赖项
- 解析其他 build.gradle 文件，有些项目可能包含多个模块，Gradle 会解析每个模块自己的 build.gradle 文件

### 执行阶段

Gradle 根据任务依赖关系图执行构建任务。根据您运行的构建命令，Gradle 会确定要执行哪些任务，然后按照依赖关系的顺序执行它们。