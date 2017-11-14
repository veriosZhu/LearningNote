# Android 性能优化

-----------------------

* [概述](#概述)

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