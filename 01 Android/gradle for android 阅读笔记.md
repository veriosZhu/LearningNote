# Gradle for Android 笔记

----------------------------------

## 第四章：创建构建Variant

### 构建类型

``` gradle
android{
    buildTypes {
        release {
            minifyEnabled false
        }
    }
}
```

构建类型大家一定有注意过，在```build.gradle```中的```release```即构建类型，常见构建类型有```debug,release```两种，当然也可以自定义构建类型，例如：

``` gradle
android{
    buildTypes {
        staging {
            staging.initWith(buildTypes.debug)
            staging{
                applicationIdSuffix ".staging"
                versionNameSuffix "-staging"
                debuggable = false
            }
        }
    }
}
```

其中的```initWith()```方法创建了一个新的构建类型，并复制了一个已有类型的所有属性到新类型中。

### product flavor

放在```productFlavors```中，例子：

``` gradle
android{
    productFlavors {
        red {
            applicationId 'com.gradleandroid.red'
            versionCode 3
        }
        blue {
            applicationId 'com.gradleandroid.blue'
            versionCode 1
        }
    }
}
```

#### 高级用法：flavor维度
例如客户A和客户B都在他们的APP中想要付费版和免费版，那么一共需要四个版本。如果使用不同的flavor意味着会有很多重复的代码，因此可以使用以下的方法：

``` gradle
android {
    flavorDimensions "color", "price"

    productFlavors {
        red {
            flavorDimension "color"
        }
        blue {
            flavorDimension "color"
        }
        free {
            flavorDimension "price"
        }
        paid {
            flavorDimension "price"
        }
    }
}
```

### 构建variant

构建```variant```是构建类型和```product flavor```的结果。以下方法会创建出四个不同的构建variant
``` gradle
android {
    buildTypes {
        debug {
            buildConfigField "String", "URL", "\"http://test.example.com/api\""
        }
        staging.initWith(android.buildTypes.debug)
        staging {
             buildConfigField "String", "URL", "\"http://staging.example.com/api\""
             applicationIdSuffix ".staging"
        }
    }
    pruductFlavors {
        red {
            applicationId 'com.gradleandroid.red'
            resValue "color", "flavor_color", "#ff0000"
        }
        blue {
             applicationId 'com.gradleandroid.blue'
            resValue "color", "flavor_color", "#0000ff"
        }
    }
}

#### 高级用法：variant过滤器
使用```variantFilter```可以过滤某些variant，以下代码过滤了```release```的```blue```版本

``` gradle
android.variantFilter {
    if(variant.buildType.name.equal('release')) {
        variant.getFlavors().each() { flavor ->
            if (flavor.name.equals('blue')) {
                variant.setIgnore(true)
            }
        }
    }
}
```

## 第六章：运行测试

### 单元测试

**JUnit**：他是一个非常受欢迎的单元测试依赖库，使用以下方法添加依赖：

``` gradle
dependencies {
    testCompile 'junit:junit:4.12'
}
```

之后只需运行```gradlew test```就可以通过gradle运行所有的测试了，如果只想在某个构建variant上测试，运行命令：`gradlewtestDebug`。

**Robolectric**：不需要运行设备或者模拟器即可在测试用例中使用Android资源。

### 功能测试

功能测试用于测试应用中的几个组件是否可以一起正常工作。例如测试某一个按钮是否能打开一个新的Activity。
**Espresso**使用方法：

``` gradle
defaultConfig {
    testInstrumentationRunner
      "android.support.test.runner.AndroidJUnitRunner"
}
dependencies {
    androidTestCompile 'com.android.support.test.espresso:espresso-core:2.2'
    androidTestCompile 'com.android.support.test.espresso:espresso-contrib:2.2'
}
```

### 测试覆盖率

用于获知你的代码被测试用例覆盖了多少。

**Jacoco**：使用方法非常简单

``` gradle
buildTypes {
    debug {
        testCoverageEnabled = true
    }
}
```

之后执行`gradlew connectedCheck`以自动创建覆盖率报告。

## 第七章：创建任务和插件

### Groovy

#### 语法

Groovy是一种动态语言

``` groovy
println 'Hello, World!'
def name = 'Andy'
def greeting = "Hello, $name!"
def name_size = "Your name is ${name.size()} characters long."

def method = 'toString'
new Date()."$method"()
```

**类和成员变量：**

``` groovy
class MyGroovyClass {
    String greeting
    String getStreeting() {
        return 'Hello!'
    }
}

def instance = new MyGroovyClass()
```

**方法**

``` groovy
def square(def num) {
    num*num
}
square 4

// 另外一种定义
def square = { num -> num * num}
square 8
```

**Closures**

``` groovy
Closure square = {
    it * it
}
square 16
```

**集合**

``` groovy
// list
List list = [1, 2, 3, 4, 5]
list.each() { element ->
  println element
}
// 另一种写法
list.each() {
    println it
}
// map
Map pizzaPrices = [margherita:10, pepperono:12]
pizzaPrices.get('pepperono')
pizzaPrices['pepperoni']
```

### 任务入门

``` groovy
// 配置阶段运行
task hello {
    println 'Configuration'
}
// 执行阶段运行
task hello << {
    println 'Execution'
}
// 以上方法等同于doFirst
task hello {
    println 'Configuration'

    doLast {
        println 'Goodbye'
    }

    doFirst {
        println 'Hello'
    }
}
```

**指定发送顺序**

``` groovy
task task1 << {
    println 'task1'
}
task task2 << {
    println 'task2'
}
// task2必须在task1之前执行
task2.mustRunAfter task1
// 和上面语句的区别在于试图只执行task2而不执行task1时，也只能同时运行
task2.dependOn task1
```

#### 使用本体keystore

``` groovy
task getReleasePassword << {
    def password = ''
    if (rootProject.file('private.properties').exists()) {
        Properties properties = new Properties();
        properties.load(rootProject.file('private.properties').newDataInputStream())
        password = properties.getProperty('release.password')
    }
    if (!password.trim()) {
        password = new String(System.console().readPassword(" \nWhat's the secret password?"))
    }
    android.signingConfigs.release.storePassword = password
    android.signingConfigs.release.keyPassword = password
}

tasks.whenTaskAdded { theTask ->
    if (theTash.name.equals("packageRelease")) {
        theTask.dependsOn "getReleasePassword"
    }
}
```

### Hook 到Android插件

#### 自动重命名apk

``` groovy
android.applicationVariants.all { variant ->
    variant.output.each {output ->
        def file = output.outputFile
        ouput.outputFile = new File(file.parent, file.name.replace(".apk", "-${variant.versionName}.apk"))
    }
}
```

#### 动态创建新任务

这个任务可以在完成安装apk后自动运行该apk

``` groovy
android.applicationVeriants.all {variant ->
    if (variant.install) {
        task.create(name: "run$variant.name.capitalize()}", dependsOn: variant.install) {
            description "Installs the ${vaiant.deescription} and runs the main laucher activity."
        }
    }
}
```

### 创建自己的插件

#### demo

要创建一个插件首先要创建一个实现Plugin接口的类。

``` groovy
class RunPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.android.applicationVariants.all { variant ->
            if (variant.install) {
                project.tasks.create(name:
                    "run${variant.name.capitalize()}",
                    dependsOn: variant.install) {
                        // Task definition
                    }
            }
        }
    }
}
```

使用时，确保在`build.gradle`中加入`apply plugin: RunPlugin`

#### 分发插件

``` groovy
apply plugin: 'groovy'

dependencies {
    compile gradleApi()
    compile localGroovy()
}
```
