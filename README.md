[TOC]
# Gradle Plugin User Guide

## 0. 前言
完全由个人翻译，能力有限，有些细节地方翻译不是很通顺，大家可以参考[Gradle Plugin User Guide](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Testing1)英文版本阅读，如果有问题，欢迎指正。

转载请事先沟通，未经允许，谢绝转载。

## 1. 新工具介绍（Introduction）
- 能够复用代码和资源
- 能够构建几种不同版本参数的应用
- 能够配置、扩展、自定义构建过程

### 1.1 为什么选择Gradle（Why Gradle?）
Gradle是一款具有优势的构建工具，通过插件可以自定义构建过程。主要优势如下：

- 基于Groovy的领域特定语言（DSL），用于描述和操作构建过程
- 支持maven/lvy的依赖管理
- 非常灵活，并不强迫用户一定要使用最佳的构建方式
- 插件可以暴露自身的语言和接口api给构建文件使用
- 支持IDE集成

### 2.2 需求（Requirements）
- Gradle 2.2（Gradle版本是2.2及以上，因为文档中有些新特性）
- SDK with Build Tools 19.0.0.

## 2. 工程基本配置（Basic Project Setup）
Gradle工程默认的配置文件名称是`build.gradle`，在主工程的根目录下。

### 2.1 配置文件示例（Simple build files）
下面是一个Android工程的最简单配置文件的内容。

```
buildscript {
    repositories { 
        jcenter()
    }

    dependencies {
        classpath 'com.android.tools.build:gradle:1.3.1'
    }
}

apply plugin: 'com.android.application'

android {
    compileSdkVersion 23
    buildToolsVersion "23.1.0"
}
```

配置内容主要分为三部分。

- `buildscript{}`，这个部分主要配置在构建过程中的依赖。上面示例中，声明使用`jcenter`依赖库，声明了一个maven库的依赖`com.android.tools.build:gradle:1.3.1`，是指引入gradle集成工具，版本是1.3.1。（关于Android Gradle Plugin版本和Gradle版本关系，[点这里](https://developer.android.com/studio/releases/gradle-plugin.html)）
- `apply plugin`，引用插件，`com.android.application`这个插件用于构建Android工程
- `android {}`，这部分是配置构建Android工程的参数。`compileSdkVersion`和`buildToolsVersion`是必须的

**注意：主工程中只能引用`com.android.application`插件，如果引用`java`插件会报错，[参考这里](http://stackoverflow.com/questions/26861011/android-compile-error-java-plugin-has-been-applied-not-compatible-with-android)，第一个插件说明这是个Android工程，第二个插件说明这是个Java工程，所以只能引用一个。**

**注意：用户可以在local.properties文件中使用`sdk.dir`属性配置本地的Android sdk位置，或者设置一个名为Android_HOME的环境变量，这两种方法没有什么区别。**

示例`local.properties`:

```
sdk.dir=/path/to/Android/Sdk
```

### 2.2 工程结构（Project Structure）
Android工程文件有默认的目录结构。Gradle遵循约定由于配置规则，提供合理的默认值。工程以两个目录为主，一个是工程代码目录，一个是测试代码目录。

- src/main/
- src/androidTest/

在每个目录中都有一些子目录，Java工程和Android工程共有的子目录如下：

- java/
- resources/

Android工程中有一些独有的目录：

- AndroidManifest.xml
- res
- assets
- aidl
- rs
- jni
- jniLibs

所有的java文件都在`src/main/java`目录下，主要的配置文件目录是`src/main/AndroidManifest.xml`。

**src/main/AndroidManifest.xml是自动创建的，不需要手动创建**

#### 2.2.1 配置目录结构（Configuring the Structure）
默认的目录结构并不能完全适配所有情况，用户可以配置目录结构。[点击这里](https://docs.gradle.org/current/userguide/java_plugin.html#N12394)查看Java工程师怎么配置目录结构的。

在Android工程中使用同样的格式，但是因为Android工程中有独有的一些目录，所以配置信息需要写在`android {}`这部分。下面示例中，工程代码使用原来的目录，修改测试代码的目录。

```
android {
    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }

        androidTest.setRoot('tests')
    }
}
```

**注意：因为旧的目录中包含所有的文件（java、AIDL、res等等），所以需要重设所有的目录**

**注意：`setRoot()`重设目录位置，沿用之前的目录结构，这个是Android工程特有的**

### 2.3 构建任务（Build Tasks）
#### 2.3.1 通用任务（General Tasks）
使用插件（包含Java和Android插件）去构建工程会自动创建很多任务，通用的任务如下：

- assemble，打包工程所产出的文件
- check，运营工程中所有的check任务
- build， 执行assemble任务和check任务
- clean，清除工程的产出的文件

`assemble`、`check`、`build`这三个任务实际上并不做任何事，他们只是一个壳任务，实告诉Gradle去执行那些的任务。

不管什么工程，依赖了什么插件，都可以反复去调用同一个任务。例如引用一个`findBugs`插件，会创建一个新的任务，让`check`任务依赖这个新任务，这样每次调用`check`任务时候，新建的任务也会执行。

- 使用`gradle tasks`指令获取工程中所有的可执行任务
- 使用`gradle tasks --all`执行获取工程中所有可执行任务简介以及依赖关系

如果工程中未做任何修改，执行`build`任务，每个任务描述后面都会加上`UP-TO-DATE`，这意味着这个任务不需要真正地执行，因为工程没有改动。这样每个任务都可以依赖其他任务，而且不需要其他任务做构建工作。

#### 2.3.2 Java工程任务（Java project tasks）
引用`Java`插件时候，说明这个工程是个纯Java工程，会额外添加两个壳任务`jar`和`tests`。

- assemble
	- jar 打包工程产出文件
- check
 	- tests 执行所有测试

`jar`任务会直接或者间接的依赖任务`classes`，这个任务会编译java源代码；`tests`任务会依赖任务`testClasses`，但是很少会直接调用这个任务，因为`tests`任务依赖它，直接调用`tests`任务即可。

大体上，用户可能只会调用`assemble`和`check`任务，很少调用其他任务。可以[点击这里](https://docs.gradle.org/current/userguide/java_plugin.html)查看Java工程所有的任务和任务描述。

#### 2.3.3 Android工程任务（Android tasks）
引用`com.android.application`插件，说明这个工程是Android工程，在通用任务基础上会额外添加两个壳任务。

- connectedCheck，查看是否有设备连接
- deviceCheck， 查看是否连接上设备

注意，`build`任务是不依赖`connectedCheck`和`deviceCheck`任务的。

一个Android工程至少有两个构建包，debug apk和release apk。每一个构建包都有自己的壳任务。

- assemble
	- assembleDebug
	- assembleRelease

这两个任务会依赖其他一些任务，要构建出一个安装包，需要执行好多步骤。`assemble`任务依赖这两个任务，所以执行`assemble`任务时候，会产出debug和release两个apk。

**注意：Gradle支持指令简写模式，例如`gradle aR`和`gradle assembleRelease`意义是相同的，只需要保证没有其他任务能简写成`aR`。**

Android工程中check类任务有各自的依赖。

- check
	- lint
- connectedCheck
	- connectedAndroidTest
- deviceCheck
	- 它依赖于那些扩展了`tests`通用任务的任务

最后，Android工程中，也会有对程序安装和卸载的任务。

- installDebug
- installRelease
- uninstallAll
	- uninstallDebug
	- uninstallRelease
	- uninstallDebugAndroidTest

### 2.4 自定义基本构建（Basic Build Customization）

Android的插件提供了领域特定语言（DSL）来帮助用户直接地自定义构建过程。

#### 2.4.1 清单内容（Manifest entries）
通过DSL用户可以设置一些构建参数，可设置内容如下：

- minSdkVersion
- targetSdkVersion
- versionCode
- versionName
- applicationId (最终有效的包名，[点击这里](https://developer.android.com/studio/build/application-id.html)查看细节)
- testApplicationId (用于测试app)
- testInstrumentationRunner

示例如下：

```
android {
    compileSdkVersion 23
    buildToolsVersion "23.0.1"


    defaultConfig { 
        versionCode 12
        versionName "2.0"
        minSdkVersion 16
        targetSdkVersion 23
    }
}
```

[点击这里](http://google.github.io/android-gradle-dsl/current/)查看可以配置的清单参数信息。

可以在.gradle文件中动态配置这些清单信息，例如，动态配置`versionName`参数，示例如下：

```
def computeVersionName() {
    ...
}


android {
    compileSdkVersion 23
    buildToolsVersion "23.0.1"


    defaultConfig {
        versionCode 12 
        versionName computeVersionName()
        minSdkVersion 16
        targetSdkVersion 23
    }
}
```
**注意：自定义时尽量不要使用gettter的方法名，防止冲突，例如，`defaultConfig.getVersionName()`会替换掉自定义的`getVersionName()`方法，也就是说每一个参数都有默认的getter方法**

#### 2.4.2 构建类型（Build Types）

Android工程中默认的会有debug和release两种构建方式，主要区别在于调试程序的能力以及apk签名细节。debug的版本为了防止在构建过程中弹出提示，系统会根据明文的用户名/密码自动创建一个数字证书用于签名，使用debug证书签名的apk是无法上架销售的。release版本在构建过程中不进行签名，将签名放在之后的环节。

在`buildTypes`中配置构建类型信息，默认会创建两种构建方式，debug和release，在Android工程中允许自定义这两种构建方式的具体细节信息。示例如下：

```
android {
    buildTypes {
        debug {
            applicationIdSuffix ".debug"
        }


        jnidebug {
            initWith(buildTypes.debug)
            applicationIdSuffix ".jnidebug"
            jniDebuggable true
        }
    }
}
```

上面代码作用：

- 设置debug构建类型的包名是`<app appliationId>.debug`，这样一台设备上面就可以同时安装debug和release的包，不会出现包名冲突情况
- 新建一个新的构建类型，名为`jnidebug`，`initWith(buildTypes.debug)`表示`buildTypes.debug`构建类型（Build Type）配置信息应用到这个构建中
- 重新设置包名同时设为`jniDebuggable`为true，开启debug模式

在`buildTypes`中新建一个新的构建类型非常方便，可以使用`initWith()`复用其他构建类型的构建参数。[点击这个](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.BuildType.html)查看可配置的构建参数。

除了修改构建参数意外，`buildTypes`中还可以添加特定的代码和资源。每一种构建类型，默认都有一个特定资源目录`src/<build_type_name>/`，例如`src/debug/java`目录，只有构建debug类型apk时候才会用到这个目录下的资源。这就意味着构建类型不能是`main`和`androidTest`，这两个目录是工程的默认目录，参考上面2.2提到的目录结构。

跟上文提到的`sourceSet`一样，每一种构建类型可以重设目录，示例如下：

```
android {
    sourceSets.jnidebug.setRoot('foo/jnidebug')
}
```

另外，对于每一个构建类型，都会有一个新的工程任务被创建，名为`assemble<Build Type Name>`，例如上文提到的`assembleDebug`和`assembleRelease`，	这两个任务也是来源于此。

根据这个规则，上面配置信息就会产生`assembleJnidebug`新任务，`assemble`任务像依赖`assembleDebug`和`assembleRelease`任务一样，也会依赖这个新任务。


可能新建构建类型的场景:

- 某些权限/模式在debug才开启，release版本不开启
- 自定义debug调试实现
- debug模式需要一些额外的资源

构建中设置的代码/资源主要用于以下几点：

- 合并到主清单
- 代码实现替换
- 资源覆盖

#### 2.4.3 签名信息配置（Signing Configurations）
应用签名以下信息是必须的：

- A keystore
- A keystore password
- A key alias name
- A key password
- The store type

[点击这里](https://developer.android.com/studio/publish/app-signing.html)查看Android官方签名细节及具体过程。

[点击这里](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.SigningConfig.html)查看可配置的签名信息，这些信息是在`signingConfigs{}`模块中配置的。

Android工程中，debug构建会用通用的`debug.keysotre`和密码、通用的key和密码，`keystore`文件位于`$HOME/.android/debug.keystore`这个目录。

具体示例如下：

```
android {
    signingConfigs {
        debug {
            storeFile file("debug.keystore")
        }


        myConfig {
            storeFile file("other.keystore")
            storePassword "android"
            keyAlias "androiddebugkey"
            keyPassword "android"
        }
    }


    buildTypes {
        foo {
            signingConfig signingConfigs.myConfig
        }
    }
}
```

上述示例中声明了两个签名类型`signingConfigs.debug`和 `signingConfigs.myConfig`，两者都将设置了keystore位置，位于工程根目录，同时
`myCondig`设置了其他必需信息，`debug`使用通用信息，不用配置。

**注意：只有`debug`类型签名的keystore位于默认位置，系统才会自动创建，如果重设了keystore的位置，就不会自动创建。新建签名类型会自动使用默认的keystore，如果没有，系统会自动创建，也就是说，系统是否自动创建keystore，是跟签名类型的`storeFile`的位置有关系，跟签名类型的名称没有关系。**

**注意：`storeFile`所以的目录在工程的根目录，是个相对目录，当然也可以设置为绝对目录，但是不推荐这样做**

**注意：如果要根据具体情况来控制签名参数，就不能直接将key和密码等信息直接写在`signingConfigs`中，可以在`gradle.properties`文件中设置签名具体细节，然后在`signingConfigs`引用，具体[点击这里](http://stackoverflow.com/questions/18328730/how-to-create-a-release-signed-apk-file-using-gradle)查看**

## 3. 工程依赖/Android库/多工程设置（Dependencies, Android Libraries and Multi-project setup）

Gradle工程可以依赖其他组件，这些组件可能是外部jar包也可能是一个Gradle工程。


### 3.1 依赖jar包（Dependencies on binary packages）
#### 3.1.1 本库jar包（Local packages）
依赖外部jar包，需要在`.gradle`文件中使用`compile`进行配置，示例如下：

```
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
}


android {
    ...
}
```

**注意：`dependencies`属于标准Gradle API的DSL属性，并不是属于`andorid{}`的元素**

`compile`属性用于编译整个工程，`compile`的库都会被添加到编译路径中，最终也会打包到最终的apk中。依赖类型分为以下几种：

- compile，主工程
- androidTestCompile，测试工程
- debugCompile， debug构建类型
- releaseCompile， release构建类型

因为构建一个apk不可能没有构建类型，所以一般至少有两个`compile`类型配置（`compile`和`<buildtype>Compile`）甚至更多。每创建一个新的构建类型，系统都会自动基于构建类型的名称创建一个新的`compile`类型，名为`<buildtype>Compile`。这在构建打包过程非常有用，例如`debug`版本需要某个外部库而`release`版本不需要，又比如不同的构建打包对同一个外部库依赖的版本不同。[点击这里](https://docs.gradle.org/current/userguide/dependency_management.html#sub:version_conflicts)查看解决jar包版本冲突的具体信息。

#### 3.1.2 远程依赖（Remote artifacts）
Gradle支持从Maven/Lvy仓库拉取依赖。首先声明仓库，然后要在声明具体的依赖。示例如下：

```
repositories {
     jcenter()
}


dependencies {
    compile 'com.google.guava:guava:18.0'
}


android {
    ...
}
```

**注意：`jcenter()`是一个仓库URL的缩写。Gradle支持远端和本地仓库。**

**注意：Gradle支持依赖传递，也就是说A工程依赖B，B依赖C，那么A工程也会依赖C，A工程会从仓库中获取C。**

[点击这里](https://docs.gradle.org/current/userguide/artifact_dependencies_tutorial.html)查看更多的依赖设置细节；[点击这里](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.dsl.DependencyHandler.html)查看具体依赖设置语言示例。

### 3.2 多工程设置（Multi project setup）
通过多工程依赖设置，Gradle工程可以依赖其他的Gradle工程。多工程设置是通过将其他被依赖的Gradle工程放入主工程的子目录中，如下结构：

```
MyProject/
 + app/
 + libraries/
    + lib1/
    + lib2/
```

这里面有三个Gradle工程，通过以下的方式能够引用到具体的工程：

- `:app`
- `:libraries:lib1`
- `:libraries:lib2`

每一个工程拥有自己的`build.gradle`文件，配置该工程的构建细节，目录结构如下：

```
MyProject/
 | settings.gradle
 + app/
    | build.gradle
 + libraries/
    + lib1/
       | build.gradle
    + lib2/
       | build.gradle
```

另外，在上面结构中可以看见在主工程的根目录中有一个`settings.gradle`文件，这个文件是用来定义那些目录是Gradle工程，`settings.gradle`示例内容如下，里面定义了三个Gradle工程目录：

```
include ':app', ':libraries:lib1', ':libraries:lib2'
```
主工程`:app`想要依赖其他的Gradle工程，只需要在它自身的`build.gradle`文件中添加依赖关系：

```
dependencies {
     compile project(':libraries:lib1')
}

android {
	  ...
}
```

[点击这里](https://docs.gradle.org/current/userguide/multi_project_builds.html)查看更多多工程设置细节。

### 3.3 库工程(Library projects)


上面所提到的多工程设置，`:libraries:lib1`, `:libraries:lib2`可以是Java工程，`:app`使用它们产生的jar包。但是，如果你想共享那些使用Android APIs或者使用Android-style的资源文件的代码，就不能使用上述的普通的Java工程，必须是Android库工程。

#### 3.3.1 创建库工程（Creating a Library Project）
库工程和平常的Android工程很类似，有一些细小的区别。构建库工程（Library）和构建一个应用工程（Application）是不同的，所以需要引用另一个插件'com.android.library'，和`com.android.application`插件一样，都是由`com.android.tools.build.gradle`jar包提供。下面是库工程`build.gradle`文件的示例。

```
buildscript {
    repositories {
        jcenter()
    }


    dependencies {
        classpath 'com.android.tools.build:gradle:1.3.1'
    }
}


apply plugin: 'com.android.library'


android {
    compileSdkVersion 23
    buildToolsVersion "23.0.1"
}
```

这个库工程使用sdk编译版本是23，`SourceSet`、`buildTypes`和`dependencies`都沿用他们所在的主工程，当然，也可以在库工程自定义这些构建信息。

#### 3.3.2 库工程和应用工程（主工程）的区别（Differences between a Project and a Library Project）
库工程主要产出一个代表Android库的aar包，里面包括编译后的代码（jar包和.so文件）和一些资源文件（manifest、res、assets）；库工程也可以构建出一个测试apk用于测试库工程，这个测试apk是独立于主工程apk的，库工程有`assembleDebug`和`assembleRelease`壳任务，所以用指令构建库工程和构建主工程是没有区别的。其余地方，库功臣和主工程是相同的。他们都有构建类型`buildTypes`和定制版本`product flavors`(后续会讲解)，可以产出多个版本的aar包。注意`buildTypes`中大多数构建参数不适用于库工程，同时，可以通过更改`sourceSet`更改库工程的内容，这取决于库工程是被主工程使用，还是用于测试。

#### 3.3.3 引用库工程（Referencing a Library）
引用库工程示例如下：

```
dependencies {
    compile project(':libraries:lib1')
    compile project(':libraries:lib2')
}

android {
	  ...
}
```

#### 3.3.4 发布库工程（Library Publication）
库工程会默认发布`release`版本，这个版本可以被其他所有的工程引用，与这些工程的构建版本无关。可以通过设置参数控制库工程发布的版本，示例如下：

```
android {
    defaultPublishConfig "debug"
}
```

注意`defaultPublishConfig`内容是构建版本全名，`release`和`debug`是系统默认的构建版本名，我们也可以改成我们自定义的构建版本全名，示例如下：

```
android {
    defaultPublishConfig "flavor1Debug"
}
```

当然，也可以发布所有版本的库工程，示例如下：

```
android {
    publishNonDefault true
}
```

发布多个库工程版本意味着会产生多个aar包，而不是一个aar包包含多个版本，每个aar包都是一个独立的版本。

不同的构建可以依赖同一个库工程的不同版本，示例如下：

```
dependencies {
    flavor1Compile project(path: ':lib1', configuration: 'flavor1Release')
    flavor2Compile project(path: ':lib1', configuration: 'flavor2Release')
}
```

**注意：发布版本的`defaultPublishConfig`变量的内容必须是构建版本的完整名**

**注意：一旦设置了`publishNonDefault true`，会将所有版本的aar包都上传到统一maven仓库，但是，这种做法是不合理的，一个maven仓库目录应该仅对应一个系列版本的aar包，例如`debug`和`release`版本的aar包分别在不同的maven仓库目录中，或者保证不同版本的依赖仅仅发生在工程内部，不上传到maven仓库。**

## 4. 测试（Testing）
可以建立一个测试工程集成到主工程当中，不需要单独新建一个测试工程。

### 4.1 单元测试（Unit testing）
Gradle 1.1版本之后就支持单元测试，[点击这里](https://developer.android.com/training/testing/start/index.html)查看详情。本章所提及的真机测试`instrumentation tests`，是指需要单独构建一个测试apk，运行在真机或者模拟器上的一种测试。

### 4.2 基本配置（Basics and Configuration）
上文中提到Android工程中默认有两个目录`src/main/`、`src/androidTest/`。使用`src/androidTest/`这个目录中的资源会构建一个使用Android测试框架，并且布署到真机（或测试机）上的测试apk来测试应用程序。Android测试框架包含单元测试、真机测试、UI自动化测试。测试apk的清单配置中的`<instrumentation>`节点会自动生成，同时用户也可以在`src/androidTest/AndroidManifest.xml`中添加额外模块用于测试。

下面列出测试apk中可能用到的属性，[点击这里](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.ProductFlavor.html)查看详情：

- testApplicationId
- testInstrumentationRunner
- testHandleProfiling
- testFunctionalTest

这些属性是在`andorid.defaultConfig`中配置的，示例如下：

```
android {
    defaultConfig {
        testApplicationId "com.test.foo"
        testInstrumentationRunner "android.test.InstrumentationTestRunner"
        testHandleProfiling true 
        testFunctionalTest true
    }
}
```

在测试程序的清单配置（manifest）中，`<instrumentation>`节点中的`targetPackage`属性会根据被测试的应用程式包名自动生成，这个属性不受自定义的`defaultConfig`配置和`buildType`配置所影响。这也是manifest文件需要自动生成的一个原因。

另外，`androidTest`可以有自己的依赖配置，默认情况下，应用程序和它的依赖都会自动添加到测试应用的classpath中，也可以通过手动拓展测试的依赖，示例如下：

```
dependencies {
    androidTestCompile 'com.google.guava:guava:11.0.2'
}
```

使用`assembleAndroidTest`任务来构建测试apk，这个任务不依赖于主工程的`assemble`任务，当设置要执行测试时候，这个任务会自动执行。

默认只有一个`buildType`会被测试，`debug`的构建类型，但是可以自定义修改被测试的`buildType`，示例如下：

```
android {
    ...
    testBuildType "staging"
}
```
	
### 4.3 解决冲突（Resolving conflicts between main and test APK）
当启动真机测试的时候，主apk和测试apk会共享同一个classpath，一旦两个apk使用了同一个库，但是使用的是不同版本，gralde构建就会失败。如果Gradle没有捕获这种情况，应用程式在测试和实际使用中可能表现不同（崩溃只是其中一种表现）。

为了促使构建成功，只需要让所有的apk使用同一个版本的库。如果这个冲突是发生在简介依赖中（没有直接在build.gradle中引入的库），只需要在`build.gradle`中引入这个库最新的版本即可，使用`compile`或者`androidTestCompile`。详情查看这里[Gradle's resolution strategy mechanism](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.ResolutionStrategy.html)。可以通过`./gradlew :app:dependencies` and `./gradlew :app:androidDependencies`查看工程的依赖树。

### 4.4 执行测试（Running tests）
上文中提到`check`类的任务（需要连接设备）是通过`connectedCheck`壳任务被唤起的，这个过程依赖`connectedDebugAndroidTest`任务，因此执行测试，需要执行`connectedDebugAndroidTest`任务，它会有以下操作：

- 确认主应用和测试应用都被构建（依赖`assembleDebug`和`assembleDebugAndroidTest`任务）
- 安装主应用和测试应用
- 运行测试
- 卸载主应用和测试应用

如果有多个设备连接，所有的测试会并行在所有设备上运行，其中任何一个设备测试失败，测试就失败。

### 4.5 测试Andorid库（Testing Android Libraries）
测试Andriod库和测试Android程序是相同的。不同的是Android库作为依赖直接继承到测试应用中，这样测试apk不仅包含测试的代码还包含这个库以及这个库的依赖。Android库的清单配置会合并到测试程序的清单配置中。`androidTest`任务改为只安装测试应用（没有其他应用），其他都是相同的。

### 4.6 测试报告（Test reports）
当执行单元测试后，Gradle会生成一份HTML报告方便查看测试结果。Andorid插件是在此基础上扩展了HTML报告，聚合了所有连接设备的测试结果。所有的测试结果以`XML`形式储存在`build/reports/androidTests/`目录下，这个目录也是可配的，示例如下：

```
android {
    ...

    testOptions {
        resultsDir = "${project.buildDir}/foo/results"
    }
}
```

### 4.6.1 多工程测试报告（Multi-projects reports）
在配置了多工程或者多依赖的工程中，当同时运行所有测试时候，针对所有的测试只生成一份测试报告是非常有用的。

为了到达这个目的，需要使用另一个插件，这个插件是Android插件中自带的，示例如下：

```
buildscript {
    repositories {
        jcenter()
    }


    dependencies {
        classpath 'com.android.tools.build:gradle:0.5.6'
    }
}


apply plugin: 'android-reporting'
```

这必须添加到工程的根目录下，例如和`settings.gradle`同目录的`build.gralde`中，然后在根目录中使用使用一下指令运行所有测试，同时合并所有测试报告：

```
gradle deviceCheck mergeAndroidReports --continue
```

**注意：`--continue`是为了保证所有测试都执行，即使其中子项目中任何一个测试失败。如果没有这个选项，当有测试失败时候，整个测试过程就会中断。**

### 4.7 Lint支持（Lint support）

**注：lint是一种检查Android项目的工具**

可以针对某一个构建版本执行lint，例如， `./gradlew lintRelease`，或者针对所有版本`./gradlew lint`，lint会生成一个记录被检查版本问题的报告。可以通过配置`lintOption`来设置lint细节，[点击这里](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.LintOptions.html#com.android.build.gradle.internal.dsl.LintOptions)查看详情，示例如下：

```
android {
    lintOptions {
        // turn off checking the given issue id's
        disable 'TypographyFractions','TypographyQuotes'

        // turn on the given issue id's
        enable 'RtlHardcoded','RtlCompat', 'RtlEnabled'

        // check *only* the given issue id's
        check 'NewApi', 'InlinedApi'
    }
}
```

## 5. 构建版本（Build Variants）
使用新构建工具的目的之一是面对同一个工程，能够编译出不同的版本。

有两个场景会用到：

- 同一个应用有多个版本，例如，免费版本和付费版本
- 同一个应用针对不同的设备有多个版本，比如说手机版和pad版，[点击这里](https://developer.android.com/google/play/publishing/multiple-apks.html)查看详情
- 1和2综合

针对同一个工程，可以编译出不同版本的apks，而不是为了编译出不同版本的apks，使用多个工程。


### 5.1 Product flavors
`product flavor`可自定义应用的版本，同一个工程中可以有多个不同的`product flavor`。

`product flavor`用来告诉构建系统不同版本之间的细小区别，`product flavor`的声明示例如下：

```
android {
    ....

    productFlavors {
        flavor1 {
            ...
        }

        flavor2 {
            ...
        }
    }
}
```

里面创建了两个`product flavor`，分别是`flavor1`和`flavor2`。

**注意：`product flavor`的名称不能和构建类型（buildType）的名称相同，也不能是`androidTest`和`test`。**

### 5.2 Build Type + Product Flavor = Build Variant
> 构建类型 + 定制版本 = 构建版本（应用版本）

就像上文提到的，每一个构建类型（buildType）都可以构建一个apk，同样的，每一个定制版本（productFlavor）都可以构建一个apk，这样的话，构建类型（buildType）和定制版本（productFlavor）结合就会形成一个新的apk，也就是构建版本。默认有两种构建类型`debug`和`release`，再加上上文定义的`flavor1`和`flavor2`，就会形成四种组合，代表四种不同的构建版本：

- Flavor1 - debug
- Flavor1 - release
- Flavor2 - debug
- Flavor2 - release

一个有没有定义`productFlavors`的工程也有构建版本，使用缺省的`productFlavors`，也就是没有`product flavor`名称，那么工程所有的构建版本和构建类型是一样的。

### 5.3 配置定制版本（Product Flavor Configuration）
`productFlavors`配置示例如下：

```
android {
    ...

    defaultConfig {
        minSdkVersion 8
        versionCode 10
    }

    productFlavors {
        flavor1 {
            applicationId "com.example.flavor1"
            versionCode 20
         }

         flavor2 {
             applicationId "com.example.flavor2"
             minSdkVersion 14
         }
    }
}
```

**注意，`android.productFlavors.*`和`android.defaultConfig`中可配置的参数是相同的，也就是说他们共享这些参数。**

`defaultConfig`为所有`productFlavor`提供一些基本配置参数，每一个`productFlavors`自定义一些额外的配置参数，也可以覆盖`defaultConfig`中配置的参数。在上面的示例中，`productFlavors`配置信息如下：

- flavor1
	- applicationId: com.example.flavor1
	- minSdkVersion: 8
	- versionCode: 20
- flavor2
	- applicationId: com.example.flavor2
	- minSdkVersion: 14
	- versionCode: 10

通常，构建类型（buildType）也会修改一些配置信息，比如说`buildType`中的`applicationIdSuffix`变量拼接在`productFlavor`的`applicationId`之后的。但是有些配置参数是以`productFlavor`为主的，比如`signingConfig`，所有的`release`版本的构建都会使用`android.buildTypes.release.signingConfig`中配置的签名信息，如果设置了`productFlavor`，所有的`release`版本的构建都会使用`android.productFlavors.*.signingConfig`中配置的签名信息。

### 5.4 资源目录和依赖（Sourcesets and Dependencies）
和构建类型相似，`productFlavor`也有自己的资源目录，示例如下:

- android.sourceSets.flavor1，资源目录是`src/flavor1/`
- android.sourceSets.flavor2，资源目录是`src/flavor2/`
- android.sourceSets.androidTestFlavor1，资源目录是`src/androidTestFlavor1/`
- android.sourceSets.androidTestFlavor2，资源目录是`src/androidTestFlavor2/`

这些资源目录中的资源会用于构建apk，构建apk资源的来源是`android.sourceSets.main`主工程的资源和构建类型的资源目录（或者`productFlavor`的资源目录）。下面是多个资源目录构建规则：

- 所有的资源代码（src/*/java）最终都会合并到最后输出包中
- 所有的Manifests.xml也会合并，根据`buildTypes`和`productFlavor`，每个不同的构建包apk，会有不同的组件和权限
- 所有的资源（包括res和assets）都会做合并，资源会做合并，资源优先级 `buildType` > `productFlavor` > 主工程
- 每一个构建版本都有唯一的一个R.class，不和其他构建版本共享

最后，和构建类型（buildType）一样，每一个`productFlavor`都有自己的依赖。例如`flavor1`需要依赖广告组件和支付组件，而`flavor2`仅仅依赖广告组件，配置文件如下：

```
dependencies {
    flavor1Compile "ads.sdk"
    flavor1Compile "pay.sdk"
    
    flavor2Compile "ads.sdk"
}
```

另外每一个构建版本都有一个资源目录，示例如下：

- android.sourceSets.flavor1Debug，资源目录`src/flavor1Debug/`
- android.sourceSets.flavor1Release，资源目录`src/flavor1Release/`
- android.sourceSets.flavor2Debug，资源目录`src/flavor2Debug/`
- android.sourceSets.flavor2Release，资源目录`src/flavor2Release/`

构建版本目录资源的优先级高于构建类型资源目录。

现在基本知道，`buildTypes`、`productFlavor`、`buildVariants`都有自己的资源目录，资源优先级是：
> `buildVariants` > `buildType` > `productFlavor` > 主工程。

### 5.5 构建任务（Building and Tasks）

上文中提到，每新建一种构建类型`buildType`，都会自动创建一个名为`assemble<Build Type Name>`的新任务。

而每新建一种`productFlavor`，会自动创建多个新任务：

1. `assemble<Variant Name>`，直接构建一个最终版本的apk，例如`assembleFlavor1Debug`任务
2. `assemble<Build Type Name>`，构建所有`buildType`类型的apks，例如，`assembleDebug`任务会构建出`Flavor1Debug`和`Flavor2Debug`版本的apk
3. `assemble<Product Flavor Name>`，构建所有`productFlavor`的apks，例如，`assembleFlavor1`任务会构建出`Flavor1Debug`和`Flavor1Release`版本的apk.

`assemble`会构建所有版本的apk。

### 5.6 多flavor构建（Multi-flavor variants）

**注：原文中`dimension of Product Flavors`，统一翻译为`productFlavor`类型，在某些文档中也翻译成维度**

在某些场景下，一个应用可能需要基于多个标准创建多个版本。例如，Google Play的multi-apk支持四个不同的过滤器，这些用于创建不同apk的过滤器需要使用多个类型的`ProductFlavor`。

例如，一个游戏有免费版本和付费版本，同时在multi-apk中需要支持ABI过滤器（ABI，二进制接口，可以让编译好的目标代码在所有支持该ABI的系统上运行，而无需对程序进行修改）。一个拥有两个版本和三个ABI过滤器的工程，需要创建六个apks（不考虑构建类型`buildType`），但是它们使用的源代码都是相同的，所以没有必要创建六个`productFlavor`。相反，只需要创建两个类型的`flavor`，就可以构建出所有的可能的版本组合。

使用`flavorDimensions`数组来实现多个类型的`flavor`，每一个`productFlavor`被分到不同的类型，示例如下：

```
android {
    ...


    flavorDimensions "abi", "version"


    productFlavors {
        freeapp {
            dimension "version"
            ...
        }

        paidapp {
            dimension "version"
            ...
        }


        arm {
            dimension "abi"
            ...
        }

        mips {
            dimension "abi"
            ...
        }

        x86 {
            dimension "abi"
            ...
        }
    }
}
```

`android.flavorDimensions`数组按顺序定义了可能使用到的`flavor`类型，每一个`productFlavor`声明自身的`flavor`类型。

上面例子中，将`productFlavor`分为两个类型，`abi`类型[arm, mips, x86]和`version`类型[freeapp, paidapp]，加上默认的[debug, release]构建类型，将会组合出以下这些构建版本（Build Variant）：

- x86-freeapp-debug
- x86-freeapp-release
- arm-freeapp-debug
- arm-freeapp-release
- mips-freeapp-debug
- mips-freeapp-release
- x86-paidapp-debug
- x86-paidapp-release
- arm-paidapp-debug
- arm-paidapp-release
- mips-paidapp-debug
- mips-paidapp-release

`android.flavorDimensions`数组定义的`flavor`类型顺序非常重要。

上述每一个构建版本名称都由以下几个属性构成：

- android.defaultConfig
- abi类型
- version类型

多`flavor`工程也有自身的资源目录，和构建版本目录相似但是目录名称**不包含构建类型**，例如：

- android.sourceSets.x86Freeapp，资源目录是`src/x86Freeapp/`
- android.sourceSets.armPaidapp，资源目录是`src/armPaidapp/`

多`flavor`的资源目录优先级高于`productFlavor`资源目录，但是低于构建类型资源目录优先级。

那么就可以列出整个工程资源优先级，资源优先级是：

> `buildVariants` > `buildType` > 多`flavor` > `productFlavor` > 主工程

### 5.7 测试（Testing）
测试多`flavor`项目和测试一般项目类似。

`androidTest`的目录适用于所有`flavor`的测试，每一个`flavor`也有单独的测试资源目录，例如：

- android.sourceSets.androidTestFlavor1，资源目录`src/androidTestFlavor1/`
- android.sourceSets.androidTestFlavor2，资源目录`src/androidTestFlavor2/`

类似的，每个`flavor`也有自己的依赖配置，示例如下：

```
dependencies {
    androidTestFlavor1Compile "..."
}
```

通过`deviceCheck`任务或者主工程的`androidTest`任务会执行`androidTestFlavor1Compile`任务。

每个`flavor`也有自己任务用于执行测试，`androidTest<VariantName>`，例如：

- androidTestFlavor1Debug
- androidTestFlavor2Debug

类似的，测试apk的构建、安装、卸载任务：

- assembleFlavor1Test
- installFlavor1Debug
- installFlavor1Test
- uninstallFlavor1Debug
- ...

最终，会根据`flavor`生成HTML测试报告，也会生成集成测试报告。测试报告的目录示例如下：

- build/androidTest-results/flavors/<FlavorName>，单个`flavor`测试报告的目录
- build/androidTest-results/all/，合并`flavor`的测试报告
- build/reports/androidTests/flavors<FlavorName>，单个`flavor`测试报告的
- build/reports/androidTests/all/，合并`flavor`的测试报告

即使自定义目录，也只会改变根目录，里面的具体子目录不会改变。

### 5.8 构建配置（BuildConfig）
在编译时，Android Studio会生成一个类`BuildConfig`，这个类包含构建特定版本时用到的一些常量，用户可以根据这些常量执行不同的操作行为。例如：

```
private void javaCode() {
    if (BuildConfig.FLAVOR.equals("paidapp")) {
        doIt();
    else {
        showOnlyInPaidAppDialog();
    }
}
```

下面是`BuildConfig`类包含的一些常量：

- boolean DEBUG – if the build is debuggable.
- int VERSION_CODE
- String VERSION_NAME
- String APPLICATION_ID
- String BUILD_TYPE – 构建类型，例如： "release"
- String FLAVOR – `productFlavor`名称，例如： "paidapp"

如果工程中使用了`flavorDimensions`多类型`flavor`，会自动生成额外的变量。以上述的配置文件为例：

- String FLAVOR = "armFreeapp"
- String FLAVOR_abi = "arm"
- String FLAVOR_version = "freeapp"

### 5.9 过滤构建版本（Filtering Variants）
当添加`productFlavor`或者使用`flavorDimensions`设置多类型`flavor`，可能有些构建版本并不需要。例如，用户定义了两个`productFlavor`，一个是正常版本，另一个仿造数据用于测试。第二个`productFlavor`仅仅是在开发过程中有用，在构建发布包时不需要这个`productFlavor`，可以通过使用`variantFilter`过滤器移除不需要的构建版本。示例如下：

```
android {
    productFlavors {
        realData
        fakeData
    }

    variantFilter { variant ->
        def names = variant.flavors*.name

        if (names.contains("fakeData") && variant.buildType.name == "release") {
            variant.ignore = true
        }
    }
}
```

使用以上配置后，工程就只有以下的构建版本：

- realDataDebug
- realDataRelease
- fakeDataDebug

[点击这里](http://google.github.io/android-gradle-dsl/current/com.android.build.api.variant.VariantFilter.html)查看可以过滤的构建属性。


## 6. 构建定制进阶（Advanced Build Customization）
### 6.1 混淆（Running ProGuard）
`ProGuard`插件是Android插件中自带的，如果构建任务（Build Type）中通过设置`minifyEnabled`为true（意为使用混淆），混淆任务会自动创建。示例如下：

```
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFile getDefaultProguardFile('proguard-android.txt')
        }
    }

    productFlavors {
        flavor1 {
        }
        flavor2 {
            proguardFile 'some-other-rules.txt'
        }
    }
}
```

构建版本同时使用构建类型（Build Type）和`productFlavor`中设置的混淆规则文件。

默认有两个混淆规则文件：

- proguard-android.txt
- proguard-android-optimize.txt

它们位于Android SDK中，使用`getDefaultProguardFile(fileName)`获取它们的绝对路径名，差别是第二个文件会启用优化。

### 6.2 忽略资源（Shrinking Resources）

这个配置设置为true，在构建打包时候，会自动忽略没有被使用到的文件。[点击这里](http://tools.android.com/tech-docs/new-build-system/resource-shrinking)查看详细信息。示例如下：

```
android {
 buildTypes {
        release {
            shrinkResources true
			  ...
        }
    }
}
```

### 6.3 操作任务(Manipulating tasks)
Java工程使用固定的任务一起协作最终打包工程。其中`classes`任务是用来编译Java源代码的，可以在`build.gradle`使用`classes`，它是`project.tasks.classes`的缩写。

在Android工程中，如果要构建打包，可能会复杂一些，因为Android工程中有大量名字相同的任务，而且它们的名字是基于`buildType`和`productFlavor`生成的。

为了解决这个问题，Android对象有两个属性：

- applicationVariants，只适用于`com.android.application`插件
- libraryVariants，只适用于`com.android.library`插件
- testVariants，两种插件都适用

这三个属性分别返回一个ApplicationVariant、LibraryVariant和TestVariant对象的[DomainObjectCollection](https://docs.gradle.org/current/javadoc/org/gradle/api/DomainObjectCollection.html)。

注意，适用这三个collection中的任意一个，都会生成所有相对应的任务，也就是说使用collection后，就不需要再更改配置。

DomainObjectCollection可以直接访问所有对象，或者通过过滤器进行筛选。

```
android.applicationVariants.all { variant ->
   ....
}
```

这三个Variant类共享下面这些属性：

|属性名        |属性类型       |描述    |
|:---         |:---         |:---|
|name         |String       |BuildVariant名称，必须保证唯一|
|description  |String       |BuildVariant的描述说明|
|dirName      |String       |BuildVariant的子文件名，必须是唯一的可能会有多个，例如：debug/flavor1|
|baseName     |String       |BuildVariant构建包的基础名字，必须唯一|
|outputFile   |File         |BuildVariant的输出文件，是个可读写的属性|
|processManifest|ProcessManifest|处理清单Manifest的任务|
|aidlCompile  |AidlCompile  |编译AIDL文件的任务|
|renderscriptCompile|RenderscriptCompile|处理Renderscript文件的任务|
|mergeResources|MergeResources|合并资源的任务|
|mergeAssets  |MergeAssets  |合并assets资源的任务|
|processResources|ProcessAndroidResources|处理和编译资源文件的任务|
|generateBuildConfig|GenerateBuildConfig|生成BuildConfig的任务|
|javaCompile|JavaCompile|编译Java源代码的任务|
|processJavaResources|Copy|处理Java资源的任务|
|assemble|DefaultTask|BuildVariant构建壳任务|

`ApplicationVariant`类还有以下附加属性：

|属性名         |属性类型      |描述                 |
|:---          |:---         |:---                |
|buildType     |BuildType    |BuildVariant的构建类型|
|productFlavors|List<ProductFlavor>| BuildVariant的`productFlavor`，不会为null但可以为空|
|mergedFlavor  |ProductFlavor|合并`android.defaultConfig`和`variant.productFlavors`的任务|
|signingConfig |SigningConfig|BuildVariant使用的签名|
|isSigningReady|boolean      |true表示BuildVariant配置了签名所需要的信息|
|testVariant   |BuildVariant |用于测试这个BuildVariant的BuildVariant|
|dex           |Dex          |将代码打包成dex的任务，如果是库工程，那么这个任务不能为null|
|packageApplication|PackageApplication|打包最终apk的任务，如果是个库工程，这个任务可以为null|
|zipAlign      |ZipAlign     |zip压缩apk的任务，如果是个库工程或者apk不被签名，这个任务可以为null|
|install       |DefaultTask  |安装apk任务，不能为null|
|uninstall     |DefaultTask  |卸载apk任务          |

`LibraryVariant`类有以下附加属性：

|属性名         |属性类型       |描述               |
|:---          |:---          |:---              |
|buildType     |BuildType     |BuildVariant的构建类型|
|mergedFlavor  |ProductFlavor |The defaultConfig values |
|testVariant   |BuildVariant  |用于测试这个BuildVariant的BuildVariant|
|packageLibrary|Zip           |用于打包aar的任务，如果是库工程，不能为null|

`TestVariant`类有以下附加属性：

|属性名         |属性类型       |描述               |
|:---          |:---          |:---              |
|buildType     |BuildType     |BuildVariant的构建类型|
|productFlavors|List<ProductFlavor>| BuildVariant的`productFlavor`，不会为null但可以为空|
|mergedFlavor  |ProductFlavor|合并`android.defaultConfig`和`variant.productFlavors`的任务|
|signingConfig |SigningConfig|BuildVariant使用的签名|
|isSigningReady|boolean      |true表示BuildVariant配置了签名所需要的信息|
|testedVariant |BaseVariant  |被TestVariant测试的BaseVariant |
|dex           |Dex          |将代码打包成dex的任务，如果是库工程，那么这个任务不能为null|
|packageApplication|PackageApplication|打包最终apk的任务，如果是个库工程，这个任务可以为null|
|zipAlign      |ZipAlign     |zip压缩apk的任务，如果是个库工程或者apk不被签名，这个任务可以为null|
|install       |DefaultTask  |安装apk任务，不能为null|
|uninstall     |DefaultTask  |卸载apk任务          |
|connectedAndroidTest|DefaultTask|在连接设备上执行Android测试的任务|
|providerAndroidTest |DefaultTask|使用拓展API执行Android测试的任务|

Android特有任务类型的API：

- ProcessManifest
	- File manifestOutputFile
- AidlCompile
	- File sourceOutputDir
- RenderscriptCompile
	-	File sourceOutputDir
	- File resOutputDir
- MergeResources
	- File outputDir
- MergeAssets
	- File outputDir
- ProcessAndroidResources
	- File manifestFile
	- File resDir
	- File assetsDir
	- File sourceOutputDir
	- File textSymbolOutputDir
	- File packageOutputFile
	- File proguardOutputFile
- GenerateBuildConfig
	- File sourceOutputDir
- Dex
	- File outputFolder
- PackageApplication
	- File resourceFile
	- File dexFile
	- File javaResourceDir
	- File jniDir
	- File outputFile
		- 在Variant对象中修改`outputFile`属性可以改变最终输出的文件夹.
- ZipAlign
	- File inputFile
	- File outputFile
		- 在Variant对象中修改`outputFile`属性可以改变最终输出的文件夹.

每一个任务类型的API由于Gradle的工作方式以及Android插件配置方式而受到限制。首先，Gradle任务只能被配置输入和输出的目录以及一些可能使用到的常量，其次大多数任务的输出都不是固定单一的，一般都混合了sourceSet、Build Type和Product Flavor中的值。这都是为了保证构建文件的简单和可读性，让开发者通过DSL语言去修改构建过程，而不是深入修改任务并改变构建过程。

同时，除了`ZipAlign`任务，其他类型的任务都需要设置私有数据让它们运行，这就意味着无法手动创建这些类型的新任务。

这些API也有可能被更改，目前大部分API都是围绕着给定任务的输入输出来添加外的处理。


### 6.4 配置JDK版本（Setting language level）
默认会根据`compileSdkVersion`来选择JDK版本，可以通过`compileOptions`设置编译时使用的JDK版本。示例如下：

```
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_6
        targetCompatibility JavaVersion.VERSION_1_6
    }
}
```



