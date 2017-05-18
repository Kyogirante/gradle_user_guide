# Gradle Plugin User Guide

## 1. 新工具目标
- 能够很简单地复用代码和资源
- 能够很简单地构建几种不同版本参数的应用
- 能够很简单地配置、扩展、自定义构建过程

### 1.1 为什么选择Gradle
Gradle是一款具有优势的构建工具，通过插件可以自定义构建过程。主要优势如下：

- 基于Groovy的领域特定语言（DSL），用于描述和操作构建过程
- 支持maven/lvy的依赖管理
- 非常灵活，并不强迫用户一定要使用最佳的构建方式
- 插件可以暴露自身的语言和接口api给构建文件使用
- 支持IDE集成

## 2. 工程基本配置
Gradle工程默认的配置文件名称是`build.gradle`，在项目工程的根目录下。

### 2.1 配置文件示例
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

- `buildscript{}`，这个部分主要配置在构建过程中的依赖。上面示例中，声明使用`jcenter`依赖库，声明了一个maven库的依赖`com.android.tools.build:gradle:1.3.1`，是指依赖Android的gradle插件，版本是1.3.1。（关于Android Gradle Plugin版本和Gradle版本关系，[点这里](https://developer.android.com/studio/releases/gradle-plugin.html)）
- `apply plugin`，引用插件，`com.android.application`这个插件用于构建Android工程
- `android {}`，这部分是配置构建Android工程的参数。`compileSdkVersion`和`buildToolsVersion`是必须的

**注意：主工程中只能引用`com.android.application`插件，如果引用`java`插件会报错，[参考这里](http://stackoverflow.com/questions/26861011/android-compile-error-java-plugin-has-been-applied-not-compatible-with-android)，第一个插件说明这是个Android工程，第二个插件说明这是个Java工程，所以只能引用一个。**

**注意：用户可以在local.properties文件中使用`sdk.dir`属性配置本地的Android sdk位置，或者设置一个名为Android_HOME的环境变量，这两种方法没有什么区别。**

示例`local.properties`:

```
sdk.dir=/path/to/Android/Sdk
```

### 2.2 工程结构
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

#### 2.2.1 配置目录结构
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

### 2.3 构建任务
#### 2.3.1 通用任务
使用插件（包含Java和Android插件）去构建工程会自动创建很多任务，通用的任务如下：

- assemble，打包工程所产出的文件
- check，运营工程中所有的check任务
- build， 执行assemble任务和check任务
- clean，清除工程的产出的文件

`assemble`、`check`、`build`这三个任务实际上并不做任何事，他们只是一个壳任务，实际上是由所引用的插件去执行具体的任务。

不管什么项目，依赖了什么插件，都可以反复去调用同一个任务。例如引用一个`findBugs`插件，会创建一个新的任务，让`check`任务依赖这个新任务，这样每次调用`check`任务时候，新建的任务也会执行。

- 使用`gradle tasks`指令获取工程中所有的可执行任务
- 使用`gradle tasks --all`执行获取工程中所有可执行任务简介以及依赖关系

如果工程中未做任何修改，执行`build`任务，每个任务描述后面都会加上`UP-TO-DATE`，这意味着这个任务不需要真正地执行，因为工程没有改动。这样每个任务都可以依赖其他任务，而且不需要其他任务做构建工作。

#### 2.3.2 Java工程任务
引用`Java`插件时候，说明这个项目是个纯Java工程，会额外添加两个壳任务`jar`和`tests`。

- assemble
	- jar 打包工程产出文件
- check
 	- tests 执行所有测试

`jar`任务会直接或者间接的依赖任务`classes`，这个任务会编译java源代码；`tests`任务会依赖任务`testClasses`，但是很少会直接调用这个任务，因为`tests`任务依赖它，直接调用`tests`任务即可。

大体上，用户可能只会调用`assemble`和`check`任务，很少调用其他任务。可以[点击这里](https://docs.gradle.org/current/userguide/java_plugin.html)查看Java工程所有的任务和任务描述。

#### 2.3.3 Android工程任务
引用`com.android.application`插件，说明这个项目是Android工程，在通用任务基础上会额外添加两个壳任务。

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

### 2.4 自定义基本构建

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
- 新建一个新的构建类型，名为`jnidebug`，`initWith(buildTypes.debug)`表示`buildTypes.debug`构建类型配置信息应用到这个构建中
- 重新设置包名同时设为`jniDebuggable`为true，开启debug模式

在`buildTypes`中新建一个新的构建类型非常方便，可以使用`initWith()`复用其他构建类型的构建参数。[点击这个](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.BuildType.html)查看可配置的构建参数。

除了修改构建参数意外，`buildTypes`中还可以添加特定的代码和资源。每一种构建类型，默认都有一个特定资源目录`src/<build_type_name>/`，例如`src/debug/java`目录，只有构建debug类型apk时候才会用到这个目录下的资源。这就意味着构建类型不能是`main`和`androidTest`，这两个目录是工程的默认目录，参考上面2.2提到的目录结构。

跟上文提到的`sourceSet`一样，每一种构建类型可以重设目录，示例如下：

```
android {
    sourceSets.jnidebug.setRoot('foo/jnidebug')
}
```

另外，对于每一个构建类型，都会有一个新的工程任务被创建，名为`assemble<Build_Type_Name>`，例如上文提到的`assembleDebug`和`assembleRelease`，	这两个任务也是来源于此。

根据这个规则，上面配置信息就会产生`assembleJnidebug`新任务，`assemble`任务像依赖`assembleDebug`和`assembleRelease`任务一样，也会依赖这个新任务。


可能新建构建类型的场景:

- 某些权限/模式在debug才开启，release版本不开启
- 自定义debug调试实现
- debug模式需要一些额外的资源

构建中设置的代码/资源主要用于以下几点：

- 合并到主清单
- 代码实现替换
- 资源覆盖

#### 2.4.3 签名信息配置
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

## 3. 工程依赖/Android库/多工程设置
Gradle项目可以依赖其他组件，这些组件可能是外部jar包也可能是一个Gradle项目。


### 3.1 依赖jar包
#### 3.1.1 本库jar包
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

**注意：Gradle支持依赖传递，也就是说A项目依赖B，B依赖C，那么A项目也会依赖C，A项目会从仓库中获取C。**

[点击这里](https://docs.gradle.org/current/userguide/artifact_dependencies_tutorial.html)查看更多的依赖设置细节；[点击这里](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.dsl.DependencyHandler.html)查看具体依赖设置语言示例。

### 3.2 多项目设置（Multi project setup）
通过多项目设置，Gradle项目可以依赖其他的Gradle项目。多项目设置是通过将其他被依赖的Gradle项目放入主工程的子目录中，如下结构：

```
MyProject/
 + app/
 + libraries/
    + lib1/
    + lib2/
```

这里面有三个Gradle项目，通过以下的方式能够引用到具体的项目：

- `:app`
- `:libraries:lib1`
- `:libraries:lib2`

每一个项目拥有自己的`build.gradle`文件，配置该项目的构建细节，目录结构如下：

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

另外，在上面结构中可以看见在主项目的根目录中有一个`settings.gradle`文件，这个文件是用来定义那些目录是Gradle工程，`settings.gradle`示例内容如下，里面定义了三个Gradle工程目录：

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

[点击这里](https://docs.gradle.org/current/userguide/multi_project_builds.html)查看更多多项目设置细节。

### 3.3 库工程(Library projects)



