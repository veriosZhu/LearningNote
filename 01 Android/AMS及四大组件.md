# MAS及四大组件

## 目录

## Activity

![launch1](src/launch-activity-1.png)
启动流程：

1. launcher通过startActivity开启一个新应用
2. 调用Instrumentation#startActivity
3. 调用ActivityManager代理远程调用AMS方法
![launch2](src/launch-activity-2.png)
4. 经过ActivityStart / ActivityStackSupervisor / ActivityStack，最后调用ActivityStackSupervisor#startSpecificActivityLocked方法
5. 该方法会做判断，如果是打开新app转6，否则转11
6. AMS创建新进程，使用Socket通知Zygote进程fork出一个进程，用来承载即将启动的Activity，指定main函数的类为ActivityThread
7. main函数中开启Looper，并执行ActivityThread#attach
8. 继续跨进程调用AMS#attachApplication
9. AMS根据进程记录将传递给ApplicationThread#bindApplication详见10，然后再调用mStackSupervisor#attachApplicationLocked启动activiy，详见11
10. 转到主线程，创建Context、Instrumentation、application，并调用application#oncreate
11. 调用ApplicationThread#scheduleLaunchActivity，之后切回本地进程
12. 在ActivityThread#handleLaunchActivity中依次调用performLaunchActivity，handleResumeActivity。
13. performLaunchActivity中首先创建activity，然后依次调用Activity的attach()->setTheme()->performCreate()->performStart()->performRestoreInstanceState()->onPostCreate()
14. 在handleResumeActivity中performResume()->onPostResume()->WindowManager#addView()
![launch3](src/launch-activity-3.png)

### 说明
ActivityStarter：决定了intent和flag如何打开Activity和关联task、stack

ActivityStack：管理者Activity

ActivityStack管理Activity生命周期相关的几个方法。

startActivityLocked()
resumeTopActivityLocked()
completeResumeLocked()
startPausingLocked()
completePauseLocked()
stopActivityLocked()
activityPausedLocked()
finishActivityLocked()
activityDestroyedLocked()
ActivityStackSupervisor：由于多屏功能的出现，就需要ActivityStackSupervisor这么一个类来管理ActivityStack。
![launch4](src/launch-activity-4.png)

## Service

### startService
![start_service1](src/launch-service-start1.png)
1. 通过ContextImpl发起startService操作，之后跨进程调用AMS的startService方法
2. 在ActiveService#realStartServiceLocked中跨进程调用ApplicationThread#scheduleCreateService(详见3)，再调用ApplicationThread#scheduleServiceArgs(详见4)
3. ApplicationThread#scheduleCreateService：切换到主线程，构造出service, ContextImpl, 执行Service#attach，Service#onCreate
4. ApplicationThread#scheduleServiceArgs：切换到主线程，调用Service#onStartCommand
![start_service2](src/launch-service-start2.png)

### bindService
![bind_service1](src/launch-service-bind1.png)

1. 通过ContextImpl发起bindService操作，由于涉及到跨进程调用，因此把ServiceConnection对象封装成IBinder类型的InnerConnection对象。
2. 跨进程调用AMS#bindService，然后和startService类似，交给ActiveService实际操作。
3. 在ActiveService#bindServiceLocked中会调用ApplicationThread#scheduleCareteService和ApplicationThread#scheduleBindService。前一个方法与startService类似
4. ActivityThread切换到主线程，然后再跨进程调用AMS#publishService
5. AMS仍旧交给ActiveService执行publishServiceLocked()，然后会调用第1步中构造的InnerConnection#connect
6. InnerConnection是能拿到最初的ServiceConnection对象的，因此只需要切换到主线程并调用ServiceConnection#onServiceConnected即可。

![bind_service2](src/launch-service-bind2.png)

## BroadcastReceiver
广播的工作流程主要分为两方面，一是广播的注册，二是广播的发送与接收。
注册又分为动态注册和静态注册，静态注册写在manifest中，由PMS完成注册，而动态注册是在代码中完成，这里只分析动态注册。
### 注册广播(动态注册)
1. 通过ContextImpl发起registerReceiver操作，然后与service绑定过程类似，将receiver封装成一个用于跨进程通信的IBinder(IIntentReceiver)，然后调用AMS#registerReceiver
2. AMS中的核心是将IntentReceiver作为key，ReceiverList（DeathRecipient子类）作为value保存到一个map中。最后把该ReceiverList和IntentFilter封装成BroadcastFilter并加入mReceiverResolver中。
### 发送广播
1. 由ContextImpl发起sendBroadcast操作，跨进程调用AMS#broadcastIntent
2. 将信息封装成BroadcastRecord然后加入BroadQueue中，并调用BroadcastQueue#scheduleBroadcastsLocked
3. BroadCastQueue中先切换到主线程（AMS的主线程），然后取出所有所有普通广播的BroadcastRecord
4. 针对每一个BroadcastRecord都发送给各自所有都BroadcastFilter，然后会调用ApplicationThread#scheduleRegisteredReceiver
5. 这里会调用IIntentReceiver#performReceive然后会切换到主线程，调用BroadcastReceiver#onReceive(mContext, intent)，至此完成
![broadcast1](src/launch-broadcastreceiver-1.png)
![broadcast2](src/launch-broadcastreceiver-2.png)

## ContentProvider
ContentProvider提供了一个外部访问本应用数据的接口。实现时需要继承ContentProvider并重写QURD方法。

当需要访问外部程序的数据时，通过getContentResolver().query()/insert()等操作其他程序的ContentProvider。
![contentprovider](src/launch-contentprovider.png)
1. 通过ContextImpl#getContentResolver发起操作，这时获取的实际对象为ApplicationContentResolver
2. 调用ApplicationContentResolver的query()等方法时，会调用ActivityThread#acquireProvider，它会拿到IContentProvider类型的对象，很明显这个对象也是用于跨进程调用的。因此最终query操作会交给这个IContentProvider做。
3. 在ActivityThread#acquireProvider中，首先会在本地查找是否已经注册，如果没有则调用AMS#getContentProvider，然后再安装到本地。(这里查找其实就是在mProviderMap中根据authority和userId找，安装的本质也就是放入该map中)
4. 在AMS#getContentProvider中分两种情况，如果目标provider所在的应用已经启动那么会在AMS中的ProviderMap中找到，否则需要先启动目标应用(注，这个ProviderMap不是本地的mProviderMap，两者完全不同)
5. 如果目标应用未启动，那么同样会从zygort fork一个新应用，然后调用新应用的ActivityThread#main()
6. 之后与启动新app相同，在ActivityThread#handlebindapplication()中会调用installContentProvider()，该放在在new Application()之后，在Application#onCreate()之前
7. 在ActivityThread#installContentProvider()中完成两件事：安装Provider和将Provider发布至AMS
8. 安装Provier包括：1)通过反射创建provider 2)attach info，例如是否exported等，并调用onCreate 3)建立一条记录ProviderClientRecord并放入mProviderMap 4)返回holder
9. 发布至AMS主要工作是存入AMS的ProviderMap
10. 因此，最终的query()会调用IContentProvider#query()，这个IConentProvider的真实对象是ContentProvider.Transport，它的qery()会调用ContentProvider#query()，这也就是我们重写的方法了。
