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
Gradle工程默认的配置文件名称是`build.gradle`，在主工程的根目录下。

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

不管什么工程，依赖了什么插件，都可以反复去调用同一个任务。例如引用一个`findBugs`插件，会创建一个新的任务，让`check`任务依赖这个新任务，这样每次调用`check`任务时候，新建的任务也会执行。

- 使用`gradle tasks`指令获取工程中所有的可执行任务
- 使用`gradle tasks --all`执行获取工程中所有可执行任务简介以及依赖关系

如果工程中未做任何修改，执行`build`任务，每个任务描述后面都会加上`UP-TO-DATE`，这意味着这个任务不需要真正地执行，因为工程没有改动。这样每个任务都可以依赖其他任务，而且不需要其他任务做构建工作。

#### 2.3.2 Java工程任务
引用`Java`插件时候，说明这个工程是个纯Java工程，会额外添加两个壳任务`jar`和`tests`。

- assemble
	- jar 打包工程产出文件
- check
 	- tests 执行所有测试

`jar`任务会直接或者间接的依赖任务`classes`，这个任务会编译java源代码；`tests`任务会依赖任务`testClasses`，但是很少会直接调用这个任务，因为`tests`任务依赖它，直接调用`tests`任务即可。

大体上，用户可能只会调用`assemble`和`check`任务，很少调用其他任务。可以[点击这里](https://docs.gradle.org/current/userguide/java_plugin.html)查看Java工程所有的任务和任务描述。

#### 2.3.3 Android工程任务
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

## 3. 工程依赖/Android库/多工程设置
Gradle工程可以依赖其他组件，这些组件可能是外部jar包也可能是一个Gradle工程。


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

#### 3.3.1 创建库工程
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

#### 3.3.2 库工程和应用工程（主工程）的区别
库工程主要产出一个代表Android库的aar包，里面包括编译后的代码（jar包和.so文件）和一些资源文件（manifest、res、assets）；库工程也可以构建出一个测试apk用于测试库工程，这个测试apk是独立于主工程apk的，库工程有`assembleDebug`和`assembleRelease`壳任务，所以用指令构建库工程和构建主工程是没有区别的。其余地方，库功臣和主工程是相同的。他们都有构建类型`buildTypes`和定制版本`product flavors`(后续会讲解)，可以产出多个版本的aar包。注意`buildTypes`中大多数构建参数不适用于库工程，同时，可以通过更改`sourceSet`更改库工程的内容，这取决于库工程是被主工程使用，还是用于测试。

#### 3.3.3 引用库工程
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

#### 3.3.4 发布库工程
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

**注意：一旦设置了`publishNonDefault true`，会将所有版本的aar包都上传到统一maven仓库，这种做法是不合理的，一个maven仓库目录应该仅对应一个系列版本的aar包，例如`debug`和`release`版本的aar包分别在不同的maven仓库目录中，或者保证不同版本的依赖仅仅发生在工程内部，不上传到maven仓库。**

## 4. 测试（待翻译）
### 4.1 Unit testing
### 4.2 Basics and Configuration
### 4.3 Resolving conflicts between main and test APK
### 4.4 Running tests
### 4.5 Testing Android Libraries
### 4.6 Test reports
### 4.6.1 Multi-projects reports
### 4.7 Lint support

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

就像上文提到的，每一个构建类型（buildType）都可以构建一个apk，同样的，每一个定制版本（productFlavor）都可以构建一个apk，这样的话，构建类型（buildType）和定制版本（productFlavor）结合就会形成一个新的apk，也就是构建版本（Build Variant）。默认有两种构建类型`debug`和`release`，再加上上文定义的`flavor1`和`flavor2`，就会形成四种组合，代表四种不同的构建版本（应用版本）：

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

现在基本知道，`buildTypes`、`productFlavor`、`buildVariants`都有自己的资源目录，资源优先级是`buildVariants` > `buildType` > `productFlavor` > 主工程。

### 5.5 构建任务

上文中提到，每新建一种构建类型`buildType`，都会自动创建一个名为`assemble<Build Type Name>`的新任务。

而每新建一种`productFlavor`，会自动创建多个新任务：

1. `assemble<Variant Name>`，直接构建一个最终版本的apk，例如`assembleFlavor1Debug`任务
2. `assemble<Build Type Name>`，构建所有`buildType`类型的apks，例如，`assembleDebug`任务会构建出`Flavor1Debug`和`Flavor2Debug`版本的apk
3. `assemble<Product Flavor Name>`，构建所有`productFlavor`的apks，例如，`assembleFlavor1`任务会构建出`Flavor1Debug`和`Flavor1Release`版本的apk.

`assemble`会构建所有版本的apk。

### 5.6 多flavor构建


