# LeakCanary简述

1.通过Application监听Activity的生命周期
2.在Activity的Destroy时，进行内存泄漏分析。
3.利用弱应用的特性使用一个引用队列保存Activity的引用，如果onDestroy后引用队列中存在该Activity的实例则说明成功回收。
4.若不存在，则手动利用Runtime.getRuntime().gc()方法手动触发GC，执行完后再进行一次判断。
5.若此时还没有在队列中存在，说明没有被回收，则认定此时发生内存泄漏。
6.异步执行Haha库进行引用链分析，然后通知Service发出通知。


## 参考资料
1. [LeakCanary源码解析-很值得我们学习的一款框架](https://www.jianshu.com/p/02614aa6162e)