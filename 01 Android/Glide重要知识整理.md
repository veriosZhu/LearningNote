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
    FileDescriptorLoader fileDescriptorLoader; // 从文件加载等图片使用该loader
    FixedLoaderProvider fixLoaderProvider;   
}

 class FixedLoadProvider {
     ModelLoader<A, T> modelLoader;
     ResourceTranscoder<Z, R> transcoder;
     DataLoadProvider<T, Z> dataLoadProvider;
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
这个`Request`对象其实是`GenericRequest`，它的构造函数可以说把我们所以自定义参数都包括其中，例如error图片、placeholder图片，transformation等。