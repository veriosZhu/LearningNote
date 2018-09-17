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


static final class MissingEmitter<T> extends BaseEmitter<T> {

        MissingEmitter(Subscriber<? super T> actual) {
            super(actual); // actual指的是下游的subcriber
        }

        @Override
        public void onNext(T t) {
            if (isCancelled()) {
                return;
            }
            if (t != null) {
                actual.onNext(t);   // actual指的是下游的subcriber
            } else {
               ...
                return;
            }
            ...
        }
    // onComplete 和 onError类似
    }
```
`subscribeActual`方法即是真正实现订阅操作的地方，先回忆一下：`source`是用户自定义的`FlowableOnSubscribe`，`t`可以理解为用户自定义的`Subscriber`。这个函数中首先根据背压策略构造出不同的emitter，我们使用了`MissingEmitter`，它表示不使用背压，其`onNext(), onError(), onComplete()`先判断是否解订阅，然后直接调用了自定义`Subscriber`的`onNext(), onError(), onComplete()`方法。因此`t.onSubcribe(emmitter)`其实调用了自定义`Subscriber`的`OnSubscribe()`。`source.subscribe(emitter)`中会运行自定义`FlowableOnSubscribe#subscribe(emitter)`，参数`emitter`就是刚刚根据背压策略新建的`emitter`，在`emitter#onNext()`调用自定义`Subscriber#onNext()`。至此订阅过程分析完毕。

### 小结
用传入的`FlowableOnSubscribe`构造出`FlowableCreate`，用`Subscriber`构造出`FlowableEmitter`，在`FlowableCreate#subscribe()`中调用`FlowableOnSubscribe.subscribe(emitter)`，实现了订阅过程。

## 操作符的原理
这里例举几个常用的操作符分析其原理。所有操作符都是大同小异的，主要看它的`subscribeActual()`。这里选择map, filter, concat, take, flatmap, just, error做介绍。
### map
**基本用法**
```java
Flowale.create(...)
    .map(new Function<T, R>() {
        @Override
        public R apply(T s) throws Exception {
            ...
        }
    })
    .subscribe()
```
`map()`的语义是把原元素做一个变换，其源码如下。经过map操作后得到的是`FlowableMap`对象，再看它的`subscribeActual()`，这个方法会把原`Subscriber`包装成`MapSubscriber`。因此在这种情况下，`Emitter.actual`实际对象就是这个`MapSubscriber`，我们重点看它的`onNext()`方法。该方法中的`mapper`也就是我们重写的方法，传入的参数t是从上游的`Subscriber`传过来的，也就是从`Emitter`传入的，其实就是在`FlowableOnSubscribe#subsribe()`写的。t参数被`mapper`转换秤了U类型，最后在调用下游的`Subscriber#onNext`。
``` java
// Flowable
public final <R> Flowable<R> map(Function<? super T, ? extends R> mapper) {
    return RxJavaPlugins.onAssembly(new FlowableMap<T, R>(this, mapper));
}

// FlowableMap
protected void subscribeActual(Subscriber<? super U> s) {
    if (s instanceof ConditionalSubscriber) {
        ...
    } else {
        source.subscribe(new MapSubscriber<T, U>(s, mapper));
    }
}

// MapSubscriber
static final class MapSubscriber<T, U> extends BasicFuseableSubscriber<T, U> {
    final Function<? super T, ? extends U> mapper;

    MapSubscriber(Subscriber<? super U> actual, Function<? super T, ? extends U> mapper) {
        super(actual);
        this.mapper = mapper;
    }

    @Override
    public void onNext(T t) {
        ...

        U v;

        try {
            v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
        } catch (Throwable ex) {
            fail(ex);
            return;
        }
        actual.onNext(v);
    }
}
```

### flatmap
```java
Flowale.create(...)
    .map(new Function<T, R>() {
        @Override
        public R apply(T s) throws Exception {
            ...
        }
    })
    .subscribe()
```

## 线程切换
使用Rxjava很方便的地方就在于线程切换简单，只需要一句话就能完成切换
``` java
...
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
```
### subscribeOn(Schedulers.io())
``` java
// Flowable
public final Flowable<T> subscribeOn(@NonNull Scheduler scheduler, boolean requestOn) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new FlowableSubscribeOn<T>(this, scheduler, requestOn));
}

// FlowableSubscribeOn.java
public final class FlowableSubscribeOn<T> extends AbstractFlowableWithUpstream<T , T> {
    public FlowableSubscribeOn(Flowable<T> source, Scheduler scheduler, boolean nonScheduledRequests) {
        super(source);
        this.scheduler = scheduler;
        this.nonScheduledRequests = nonScheduledRequests;
    }
 public void subscribeActual(final Subscriber<? super T> s) {
        Scheduler.Worker w = scheduler.createWorker();
        final SubscribeOnSubscriber<T> sos = new SubscribeOnSubscriber<T>(s, w, source, nonScheduledRequests);
        s.onSubscribe(sos);

        w.schedule(sos);
    }
}
```
在经过`subscribeOn()`操作后，会转换成`FlowableSubscribeOn`，关注其`subscribeActual()`。这个方法与普通的操作符相差较大，普通操作符一般都是直接调用`source.subscribe(new xxSubscriber())`，但是此处立马调用了`s.onSubscibe()`，换句话说此处没切换线程，仅运行在创建`Flowable`的线程。然后调用`Scheduler.Worker#schedule(sos)`，这里的`worker`是从`Schedulers.io()`创建的。`SubscribeOnSubscriber`其实是一个`Runnable()`，就是在`run()`方法中完成了订阅的动作。因此这个`Subscriber`的`onNext(), onComplete()`等都是在新线程的，也就是说在`subscribeOn()`下一个操作符的`onNext`是运行在这个新线程的。

``` java
 static final class SubscribeOnSubscriber<T> extends AtomicReference<Thread>
    implements FlowableSubscriber<T>, Subscription, Runnable {
    @Override
    public void run() {
        lazySet(Thread.currentThread());
        Publisher<T> src = source;
        source = null;
        src.subscribe(this);
    }

    @Override
        public void onNext(T t) {
            actual.onNext(t);
        }
}
```
下面再来看`Scheduler.Worker`对象，当使用`Schedulers.io()`时，使用的是`IOScheduler`，因此这里的`Worker`是指`EventLoopWorder`。在它的`schedule()`中又会调用`NewthreadWorker#scheduleActual`，在该方法内部完成了线程切换。因此可以看到`subscribeOn()`其实是把它所有上游的`Flowable`都放到了指定的线程中。
``` java
public class IOScheduler {
    public Worker createWorker() {
        return new EventLoopWorker(pool.get());
    }

    static final class EventLoopWorker extends Scheduler.Worker {
        public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
            ...
            return threadWorker.scheduleActual(action, delayTime, unit, tasks);
        }
    }
}

public class NewthreadWorker {
    public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        ...
        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
        ...
        Future<?> f;
        try {
            if (delayTime <= 0) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delayTime, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            ...
        }
        return sr;
    }
}

```

### observeOn(AndroidSchedulers.mainThread())



