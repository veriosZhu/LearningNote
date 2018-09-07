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
![start_service2]


https://hawksjamesf.github.io/blog/2018-03-15/framework-service-ams-component