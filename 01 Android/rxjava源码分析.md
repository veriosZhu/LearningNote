# RxJava源码分析

## 目录

## 概述
RxJava目前在越来越多的项目中应用，它最大的优势在于使代码流程清晰明了。本文假设读者对RxJava的概念、使用方法已有所了解，因此将重点放在源码，分析RxJava具体实现过程。

本文将回答以下问题：
1. 被观察对象(Observable)如何将数据发送给观察者(Observer)？
2. 各种操作符的原理是什么？
3. 线程调度如何实现？
4. 订阅如何取消？
5. 压背如何实现？

源码版本为：RxJava:2.1.16  /  RxAndroid:2.0.2

## 订阅流程解析
首先给出最基本的使用方法：
``` java
Flowable.create(new FlowableOnSubscribe<String>() {
            @Override
            public void subscribe(FlowableEmitter<String> emitter) throws Exception {
                emitter.onNext("one");
                emitter.onNext("two");
                emitter.onComplete();
            }
        }, BackpressureStrategy.MISSING)
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        Log.d(TAG, "onSubscribe");
                    }
                    @Override
                    public void onNext(String s) {
                        Log.d(TAG, "onNext: " + s);
                    }
                    @Override
                    public void onError(Throwable t) {
                        Log.e(TAG, "onError: " + t);
                    }
                    @Override
                    public void onComplete() {
                        Log.d(TAG, "onComplete");
                    }
                });
```
简单来说，这段代码表示用`Flowable.create()`创建了一个`Flowable`，后被自定义的`Subscriber`订阅，最终实现了`onNext`事件的传递。或许有同学有疑问，为什么不直接把`new FlowableOnSubscribe()`对象给自定义的`Subscriber`订阅呢？好，现在打开`Flowable.create()`函数看一看究竟。
``` java
public static <T> Flowable<T> create(FlowableOnSubscribe<T> source, BackpressureStrategy mode) {
        ...
    return RxJavaPlugins.onAssembly(new FlowableCreate<T>(source, mode));
}
```
`RxJavaPlugins.onAssembly()`方法一般都会直接返回参数本身，即`FlowableCreate`。继续打开该类，它有两个成员变量，`source`是上游的`Flowable`也就是程序中自定义的`FlowableOnSubscribe`，`backpressure`是背压策略，下文将详细分析此处先略过，源码如下：
```java
public final class FlowableCreate<T> extends Flowable<T> {
    final FlowableOnSubscribe<T> source;
    final BackpressureStrategy backpressure;
    public FlowableCreate(FlowableOnSubscribe<T> source, BackpressureStrategy backpressure) {
        this.source = source;
        this.backpressure = backpressure;
    }
        ...
}

```
因此`subscribe()`方法的真正调用者就是这里的`FlowableCreate`对象，它是在父类中实现的。切到源码，首先创建出`StrictSubscriber`，然后在`subscribeActual`中调用，该方法的具体实现在`FlowableCreate`中。此处`StrictSubscriber`的作用是为了保证Reactive-Streams的规则，可以将它简单的等同于原来的`Subscriber`，即用户自定义的`Subscriber`。
``` java
// Flowable.java
    public final void subscribe(Subscriber<? super T> s) {
            ...
        subscribe(new StrictSubscriber<T>(s));
    }
    ...
    public final void subscribe(FlowableSubscriber<? super T> s) {
        ...
        // 返回对象本身
        Subscriber<? super T> z = RxJavaPlugins.onSubscribe(this, s);
        // 最终调用处
        subscribeActual(z);
        ...
    }

// FlowableCreate    
        public void subscribeActual(Subscriber<? super T> t) {
        BaseEmitter<T> emitter;
        // 创建和背压测量相关的emitter
        switch (backpressure) {
        case MISSING: {
            emitter = new MissingEmitter<T>(t);
            break;
        }
        case ERROR: {
            emitter = new ErrorAsyncEmitter<T>(t);
            break;
        }
        case DROP: {
            emitter = new DropAsyncEmitter<T>(t);
            break;
        }
        case LATEST: {
            emitter = new LatestAsyncEmitter<T>(t);
            break;
        }
        default: {
            emitter = new BufferAsyncEmitter<T>(t, bufferSize());
            break;
        }
        }
        // 调用onSubscribe
        t.onSubscribe(emitter);
        try {
            // 源Flowable调用emitter
            source.subscribe(emitter);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            emitter.onError(ex);
        }
    }

```
`subscribeActual`方法即是真正实现订阅操作的地方，先回忆一下：`source`是用户自定义的`FlowableOnSubscribe`，`t`可以理解为用户自定义的`Subscriber`。这个函数中首先根据背压策略构造出不同的emitter，我们使用了`MissingEmitter`，它表示不使用背压，其`onNext(), onError(), onComplete()`先判断是否解订阅，然后直接调用了自定义`Subscriber`的`onNext(), onError(), onComplete()`方法。因此`t.onSubcribe(emmitter)`其实调用了自定义`Subscriber`的`OnSubscribe()`。`source.subscribe(emitter)`中会运行自定义`FlowableOnSubscribe#subscribe(emitter)`，参数`emitter`就是刚刚根据背压策略新建的`emitter`，在`emitter#onNext()`调用自定义`Subscriber#onNext()`。至此订阅过程分析完毕。

### 小结

