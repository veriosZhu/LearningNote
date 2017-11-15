# Android 性能优化

-----------------------

* [概述](#概述)
* [数据库性能优化](#数据库性能优化)
* [布局优化](#布局优化)
* [Java(Android)代码优化](#Java(Android)代码优化)
* [移动网络优化](#移动网络优化)

## 概述

### 两个概念

1. 相应时间：
 - 处理时间+网络传输时间+展开时间
2. TPS：
 - 每秒处理的事务数（系统吞吐量）

## 性能调优方式

1. 降低执行时间
-  利用多线程或者分布式提高TPS
- 缓存（包括对象缓存、IO缓存、网络缓存）
- 数据结构优化和算法优化
- 性能更优的底层接口调用，如JNI实现
- 逻辑优化
- 需求优化
 2. 同步改异步
 3. 提前或者延时操作

## 数据库性能优化

### 使用索引

合适的索引可以大大加快数据库查询的效率，经常是一到两个数量级的提升。同时，索引会占用物理空间，并且增删改时需要维护索引。


## 布局优化

### 使用抽象布局标签

#### 1.`<include>`标签

`include`标签用于将布局中公共部分提取出来以供其他layout使用

``` java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
    <ListView
        android:id="@+id/simple_list_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginBottom="@dimen/dp_80" />
    <include layout="@layout/foot.xml" />
</RelativeLayout>
```

#### 2.`<viewstub>`标签

与`include`标签一样，引入一个外部布局。不同的是，`viewstub`引入的布局默认不会扩张，即不会占用显示也不会占用位置。

`viewstub`常用语引入那些默认不会显示，只在特殊情况下显示的布局，如进度布局、网络失败布局、信息出错的提示布局等。

``` java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
 
	……
    <ViewStub
        android:id="@+id/network_error_layout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout="@layout/network_error" />
 
</RelativeLayout>
```

network_error.xml为只有在网络错误时才需要显示的布局，默认不会被解析，示例代码如下：

``` java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
 
    <Button
        android:id="@+id/network_setting"
        android:layout_width="@dimen/dp_160"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="@string/network_setting" />
 
    <Button
        android:id="@+id/network_refresh"
        android:layout_width="@dimen/dp_160"
        android:layout_height="wrap_content"
        android:layout_below="@+id/network_setting"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="@dimen/dp_10"
        android:text="@string/network_refresh" />
 
</RelativeLayout>
```

在代码中，通过`viewstub.inflate()`展开布局。

``` java
// 写法1
ViewStub stub = (ViewStub)findViewById(R.id.network_error_layout);
networkErrorView = stub.inflate();
// 写法2
View viewStub = findViewById(R.id.network_error_layout);
viewStub.setVisibility(View.VISIBLE);   // ViewStub被展开后的布局所替换
networkErrorView =  findViewById(R.id.network_error_layout); // 获取展开后的布局
```

#### 3.`<merge>`标签

使用`include`标签导致布局嵌套过多时，为了删除不必要的layout节点导致解析速度变慢，可用`merge`标签。

`merge`标签常用语两种典型情况：

1. 布局顶节点是`Framelayout`且不需要background或padding等属性。
2. 某布局被其他布局include时，使用`merge`当作顶节点，这样被引入的顶节点会被自动忽略，而将其子节点全部合并到主布局中。

例如
``` java
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >
 
    <Button
        android:id="@+id/button"
        android:layout_width="match_parent"
        android:layout_height="@dimen/dp_40"
        android:layout_above="@+id/text"/>
 
    <TextView
        android:id="@+id/text"
        android:layout_width="match_parent"
        android:layout_height="@dimen/dp_40"
        android:layout_alignParentBottom="true"
        android:text="@string/app_name" />
 
</merge>
```

### 去除不必要的嵌套和View节点

1. 首次不需要使用的节点设置为GONE或使用viewstub
2. 使用RelativeLayout代替LinearLayout

### 减少不必要的infalte

1. 对于`inflate`的布局可以直接缓存，避免下次再次`inflate`
2. `ListView`缓存

### 其他点

#### 使用`SurfaceView`或者`TextureView`代替普通`View`

`SerfaceView`和`TextureView`都是将绘图移动到另一个单独线程上提高性能。`SurfaceView`在常规视图系统之外，所以无法像常规试图一样移动、缩放或旋转一个`SurfaceView`。`TextureView`是Android4.0引入的，除了与`SurfaceView`一样在单独线程绘制外，还可以像常规视图一样被改变。

#### 使用RenderJavascript

RenderScript是Adnroid3.0引进的用来在Android上写高性能代码的一种语言。（没搞明白）

#### 使用OpenGL绘图

一般使用在游戏绘图中。

#### 尽量为所有分辨率创建资源

减少不必要的硬件缩放。可以借助[Android asset studio](https://romannurik.github.io/AndroidAssetStudio/)

### 布局调优工具

**hierarchy viewer**、**lint**

## Java(Android)代码优化

### 降低执行时间

#### 缓存

1. 线程池
2. Android图片缓存，Android图片sd缓存，数据预存缓存
3. 消息缓存
- `handler.sendMessage(handler.obtainMessage(0, object));`
4. `ListView`缓存
5. 网络缓存
6. 文件IO缓存
- 使用`BufferedInputStream`代替`InutStream`；`BufferedReader`代替`Reader`；`BufferedReader`代替`BufferedInputStream`
7. layout缓存

#### 数据结构优化

1. `StringBuilder`代替`String`，如果对字符串长度有大致了解可制定初始大小`new StringBuilder(128)`
2. 使用`SoftReference`、`WeakReference`更有利于系统垃圾回收
3. `LocalBroadcastManager`代替普通`BroadcastReceiver`，效率和安全性都更高

**数据结构选择：**

ArrayList和LinkedList的选择，ArrayList根据index取值更快，LinkedList更占内存、随机插入删除更快速、扩容效率更高。一般推荐ArrayList。

ArrayList、HashMap、LinkedHashMap、HashSet的选择，hash系列数据结构查询速度更优，ArrayList存储有序元素，HashMap为键值对数据结构，LinkedHashMap可以记住加入次序的hashMap，HashSet不允许重复元素。

HashMap、WeakHashMap选择，WeakHashMap中元素可在适当时候被系统垃圾回收器自动回收，所以适合在内存紧张型中使用。

Collections.synchronizedMap和ConcurrentHashMap的选择，ConcurrentHashMap为细分锁，锁粒度更小，并发性能更优。Collections.synchronizedMap为对象锁，自己添加函数进行锁控制更方便。
 
Android也提供了一些性能更优的数据类型，如SparseArray、SparseBooleanArray、SparseIntArray、Pair。
Sparse系列的数据结构是为key为int情况的特殊处理，采用二分查找及简单的数组存储，加上不需要泛型转换的开销，相对Map来说性能更优。

#### 算法优化

尽量不用O(n*n)时间复杂度以上的算法，必要时候可用空间换时间。

#### JNI

（还不懂。。。）

#### 逻辑优化

理清程序逻辑，减少不必要的操作。

#### 需求优化

### 异步、利用多线程提高TPS

将可能造成主线程超时操作放入另外的工作线程中。在工作线程中可以通过handler和主线程交互。

### 提前或延时操作，错开时间段提高TPS

#### 延迟操作

不在Activity、Service、BroadcastReceiver的生命周期等对响应时间敏感函数中执行耗时操作，可适当delay。
Java中延迟操作可使用ScheduledExecutorService，不推荐使用Timer.schedule;

``` java
// 延迟执行代码
ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
scheduledThreadPool.schedule(new Runnable() {

	@Override
	public void run() {
		System.out.println("delay 3 seconds");
	}
}, 3, TimeUnit.SECONDS);

// 定期执行代码
scheduledThreadPool.scheduleAtFixedRate(new Runnable() {

	@Override
	public void run() {
		System.out.println("delay 1 seconds, and excute every 3 seconds");
	}
}, 1, 3, TimeUnit.SECONDS);
```

除此之外还可以使用handler.postDelayed，handler.postAtTime，handler.sendMessageDelayed，View.postDelayed，AlarmManager定时等。

#### 提前操作

对于第一次调用耗时操作，可以统一放到初始化中。

### 网络优化

以下是网络优化中一些客户端和服务器端需要尽量遵守的准则：

1. 图片必须缓存，最好根据机型做图片做图片适配
2. 所有http请求必须添加httptimeout
3. 开启gzip压缩
4. api接口数据以json格式返回，而不是xml或html
5. 根据http头信息中的Cache-Control及expires域确定是否缓存请求结果。
6. 确定网络请求的connection是否keep-alive
7. 减少网络请求次数，服务器端适当做请求合并。
8. 减少重定向次数
9. api接口服务器端响应时间不超过100ms

## 移动网络优化

### 连接服务器优化策略

1. 不用域名，用IP直连
2. 服务器合理部署（至少含三大运营商、南中北三地部署）

### 获取数据优化策略

1. 连接复用
2. 请求合并
3. 减少请求数据大小（例如Gzip压缩）
4. CDN缓存静态资源
5. 减小返回数据大小
- 压缩：使用Gzip
- 精简数据格式：使用JSON代替xml
- 对不同设备不同网络返回不同的内容
- 增量更新
- 大文件下载
6. 数据缓存

### 其他方法

1. 预取
2. 分优先级、延迟部分请求
3. 多连接
4. 监控
