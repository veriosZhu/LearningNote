# Glide重要知识点整理

---------

## 目录

## 最基础使用
``` java
Glide.with(activity).load(url).into(imageView);
```

## Glide.with()解析
`Glide.with()`可以接受多种参数，例如`activity/fragment/context/application`，该方法会返回`RequestManager`，后续的`into`方法其实是这个类的方法。虽然接受的参数类型多样，但其实就是两种，一种是`application`另一种是非`application`，如果是`application`类型的，那么不用做任何事，直接` new RequestManager()`即可，因为`application`不需要监听生命周期。如果是非`application`类型，那么会新建`Fragment`并添加到原来的activity中目的是监听生命周期。最后需要说明的是，如果不是在主线程使用`with()`方法，将一律视为`application`类型。

## Glide.with().load()解析
`load()`方法最终会返回`DrawableTypeRequest`类型，后续等方法例如`error() / priority() / skipMemoryCache() / into() `等都是其父类的方法。
该`Request`同时支持加载bit和gif类型的，如果在该方法后添加类`asBit() / asGif()`则会将类型转换为`BitmapTypeRequest / GifTypeRequest`。
``` java
class DrawableTypeRequest extend DrawableRequestBuilder {
    StreamModelLoader streamModelLoader; // 网络、Resource等会使用该loader加载
    FileDescriptorLoader fileDescriptorLoader; // 从文件加载等图片使用该loader加载
    FixedLoaderProvider fixLoaderProvider;   
}

 class FixedLoadProvider {
     ModelLoader<A, T> modelLoader;   // 包装了StreamModelLoader和FileDescriptorLoader，实例为ImageVideoModelLoader
     ResourceTranscoder<Z, R> transcoder;  // 用于图片转码，如果同时支持git和bit那么该实例GifBitmapWrapperDrawableTranscoder
     DataLoadProvider<T, Z> dataLoadProvider;  // 用于图片编解码，如果同时支持git和bit那么该实例为ImageVideoGifDrawableLoadProvider
 }
 ```

 ## Glide.with().load().into()解析
 最重要也是最复杂的方法就是`into()`，这个方法是在祖父类`GenericRequestBuilder`中实现的。关键句：`into(glide.buildImageViewTarget(view, transcodeClass))`，换言之先转换成`Target`对象在执行`into`。如果同时支持加载bit和gif，那么会转换成`GlideDrawableImageViewTarget`，这里插一句，如果最终资源加载成功后会调用其`setResource()`方法把一个`drawble`对象放入`ImageView`中。`into()`函数主要工作是先构建出`Request`，然后执行它。

 ``` java
 public <Y extends Target<TranscodeType>> Y into(Y target) {
    ...
    Request request = buildRequest(target);   // 关键句1
    target.setRequest(request);
    lifecycle.addListener(target);
    requestTracker.runRequest(request);       // 关键句2
    return target;
}
```

关键句1的`Request`对象其实是`GenericRequest`，它的构造函数可以说把我们所有自定义参数都包括其中，例如error图片、placeholder图片，transformation等。这里会先从一个Request池取，如果池为空才新建。
关键句2最终会调用`Request.begin()`方法，在这个方法会做4件事：
1. 判断`load(url)`中传入的参数是否为空，如果为空的话直接设置error占位符。
2. 如果有override图片宽高那么开启线程加载图片（`onSizeReady`方法）
3. 如果没有设置，那么先测量ImageView的宽高然后再执行第2步
4. 设置placeholder图片。

其中1，4都是调用`GlideDrawbleImageViewTarget`对象的方法进行设置，第3步最终是通过`View.getHeight() / View.getWidth()`获取宽高。因此重点分析第2步，这一步中会把参数取出来并转交给`Engine`对象执行。
``` java
// GenericRequest.java 
Engine.load(signature, width, height,  // width和height是onSizeReady传入的
            dataFetcher,               // 从loadProvider的modelLoader中获得，实例为ImageVideoFetcher，它包装了stream型和file型的fetcher，其中stream型实例为HttpUrlFetcher
            loadProvider, transformation, // 这两个参数都是new时传入的
            transcoder,                // 用于图片转码，从loadProvider中获得，实例为GifBitmapWrapperDrawableTranscoder
            priority, isMemoryCacheable, diskCacheStrategy, // 这三个参数也是new时传入的
            this);                    // 回调，资源准备成功后调用

```

现在深入`Engine.load()`分析，它首先从cache中找，如果找到的话直接通过回调调用`onResourceReady`，否则会创建`EngineJob`和`EngineRunable`开启线程加载图片。
关于cache有三点需要说明：
1. cache只包含内存缓存不包括磁盘缓存，换言之如果仅仅缓存了磁盘还是会开启线程加载的。
2. cache有两种，一种是基于`LruCache`的缓存，用于缓存当前已经不显示的图片；另一种是使用了`Map`和`WeakReference`的缓存，用于保存正在显示的图片。
3. 用于查询的key的决定因素非常多，比如原图id、width、height、transformation等等。因此内存缓存只会存转换后的图片

现在来看`EngineJob`和`EngineRunnable`，`EngineJob`仅做一些杂事，例如保存了个线程池、切换到主线程、添加回调等等。最关键的是`EngineRunnable.run()`，现在重点分析。这个方法首先执行`decode()`然后根据其结果调用`EngineJob`的加载成功或者加载失败方法。`decode()`源码如下：
``` java
// EngineRunnable.java
private Resource<?> decode() throws Exception {
    if (isDecodingFromCache()) {
        return decodeFromCache();
    } else {
        return decodeFromSource();
    }
}
```

其中`decodeFromCache()`显然是用于加载磁盘缓存的，它会先加载转换后的资源，如果失败再加载原资源，具体分析在下一节介绍。这里先插一句，其实if判断在首次执行时返回的是true，也就是说就算关闭了磁盘缓存也会执行一次`decodeFromCache`，执行失败后会调用加载失败流程，会再执行一次`run()`，这才会执行`decodeFromSource()`。那这么做的意义何在呢？意义在于两次run所处的线程池不同，前一次位于加载cache的线程池，其核心线程仅为1，后一次位于加载source的线程池，核心线程数为cpu数。

`decodeFromSource()`是用于加载从stream或者file中的资源，这个方法内部又会调用`DecodeJob.decodeFromSource()`。它做了两件事：1.下载原图；2.变形、转码。现在开始逐句分析：

``` java
// DecodeJob.java
public Resource<Z> decodeFromSource() throws Exception {
    Resource<T> decoded = decodeSource();
    return transformEncodeAndTranscode(decoded);
}

private Resource<T> decodeSource() throws Exception {
    ...    
    final A data = fetcher.loadData(priority); // 1
    ...
    return decodeFromSourceData(data);   // 2 
}

private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
    ...
    Resource<T> transformed = transform(decoded);  // 3
    writeTransformedToCache(transformed);  // 4
    Resource<Z> result = transcode(transformed);  // 5
    ...
    return result;
    }
```

1处是下载图片，`fetcher`实例为`ImageVideoFetcher`上文已经提到它其实是包装了两种类型的`fetcher`，因此真正的执行者是`HttpUrlFetchor`，它最终调用了`HttpUrlConnection`来建立连接和下载数据并返回`InputStream`。这里我们终于找到了真正网络请求的地方！建立连接的一些具体参数也可以在这里找到，例如连接超时为2.5秒，缓存关闭，priority等，此外它还支持重定向。所以这里的返回值A的类型其实是`InputStream`。

2处将`InputStream`转换成`Resource<GifBitmapWrapper>`，这个类可以理解为包装了`GifDrawble`或者`Bitmap`。2处的具体逻辑也会因有没有开启磁盘缓存而有所区别，先关注未使用磁盘缓存的情况，会调用`GifBitmapWrapperResourceDecoder.decode()`。它会先从流中读取两个字符判断资源类型是gif型还是bit型，如果是bit型那么继续调用`GitBitmapWrapperResourceDecoder.decodeBitmapWrapper()`，然后经过`ImageVideoBitmapDecoder.decode()`、`StreamBitmapDecoder.decode()`最终交给`Downsampler.decode()`真正执行。不出意外，这里最终会还是会调用`BitmapFactory.decodeStream()`方法得到Bitmap。除此之外该方法还处理了图片角度校准等。

3处用于变形，4处将变形后的图片存入磁盘缓存中，这里先略过。

5处的作用是把bitmap适配成`Drawable`对象。但是`GifBitmapWrapper`是无法直接显示到`ImageView`上面的，只有`Bitmap`或者`Drawable`才能显示。5处的入参类型是`Resource<GifBitmapWrapper>`，我们应该还记得它包装了`GifDrawble`或者`Bitmap`，为了统一处理需要把它们都变成`Drawable`。其中`GifDrawable`已经是`Drawable`了，而`Bitmap`不是，因此这里将`Bitmap`转换成`GlideBitmapDrawable`。

现在为止`EngineRunnable.decode()`就分析完了，回到`run()`方法，把加载好的资源给到`EngineJob.onLoadComplete()`，最终切到到主线程并调用了`ViewTarget.onResoueceReady()`，就完成了全部工作。

## 缓存
### 内存缓存
在上一节中已经有提到，内存缓存有两种，包括基于LruCache的缓存及使用WeakReference的缓存。前者缓存已经不显示的图片，后者缓存正在显示的图片。在`Engine.load()`中，首先会依次在lru缓存和WeakReference缓存中找资源，如果找到就不开启线程加载了。

下面谈一下如何保存资源。在`EngineJob.handleResultOnMainThread()`中，首先会构造出`EngineResource`对象，然后回调`Engine.onEngineJobComplete()`这里会把resource加入WeakReference缓存中。至于lru缓存是通过`EngineResource.acquire() - release()`实现，每次acquire计数器+1，每次release计数器-1，发现计数器等于0时回调`Engine.onResourceRelease()`，这里会把resource放入lru缓存。

### 磁盘缓存
磁盘缓存有两类，包括缓存原始图片或者缓存转换后的图片。磁盘缓存也需要在子线程中执行，因此磁盘缓存也是先读取转换后的图片，失败后再读取原始图片，分别对应`decodeJob.decodeResultFromCache()`和`decodeJob.decodeSourceFromCache()`。这两种方法其实都调用了`loadFromCache()`区别在于传入的key不同，前者传入的key和内存缓存的key一样，决定因素有10种之多，后者的key仅含id。得到`Resource`后，前者直接transcode，后者需要先transform再transcode，这些逻辑和上一节相似。

``` java
// DecodeJob
public Resource<Z> decodeResultFromCache() throws Exception {
    if (!diskCacheStrategy.cacheResult()) {
        return null;
    }
    ...
    Resource<T> transformed = loadFromCache(resultKey);
    ...
    return transcode(transformed);
}

public Resource<Z> decodeSourceFromCache() throws Exception {
    if (!diskCacheStrategy.cacheSource()) {
        return null;
    }
    ...
    Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
    return transformEncodeAndTranscode(decoded);
    }
```

在`loadFromCache()`方法中首先会根据key拿到file，然后调用`FileToStreamDecoder`将文件以`InputStream`形式打开，然后和上一节类似用`StreamBitmapDecoder.decode()`解码。

下面来看一下怎么保存成磁盘缓存，上一节分析`DecodeJob.decodeFromSource()`中2处内部将原始文件保存到磁盘中，4处将转换后的图片保存在磁盘中。具体方法为`cacheAndDecodeSourceData()`和`writeTransformedToCache()`。