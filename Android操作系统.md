# Android 操作系统

## Android 虚拟机

### Dalvik虚拟机

Dalvik 执行的是Dex字节码，通过解释器执行。它使用的是JIT（just in time )技术来进行代码转译，每次执行应用的时候，Dalvik将程序的代码编译为机器语言执行，即：对执行的代码通过JIT生成本地机器指令来执行。

### ART虚拟机

ART（Android Run Time） 虚拟机，执行的是本地机器指令。它采用AOT（Ahead Of Time）技术，会在应用程序安装时就转换成机器语言，不再在执行时解释，从而优化了应用运行的速度。在内存管理方面，ART也有了较大的改进，对内存分配和回收都做了算法优化，降低了内存碎片化程度，回收时间也得以缩短。

## Android架构

为了能整体上大致了解Android系统涉及的知识层面，先来看一张Google官方提供的经典分层架构图，从下往上依次分为Linux内核、HAL、系统Native库和Android运行时环境、Java框架层以及应用层这5层架构，其中每一层都包含大量的子模块或子系统。

https://developer.android.google.cn/guide/platform/images/android-stack_2x.png

<img src="https://developer.android.google.cn/guide/platform/images/android-stack_2x.png" alt="Android 软件堆栈" style="zoom: 67%;" />

#### 系统启动架构图

Google提供的5层架构图很经典，但为了更进一步透视Android系统架构，本文更多的是以进程的视角，以分层的架构来诠释Android系统的全貌，阐述Android内部的环环相扣的内在联系。

<img src="http://gityuan.com/images/android-arch/android-boot.jpg" alt="process_status" style="zoom:67%;" />

**图解：** Android系统启动过程由上图从下往上的一个过程是由Boot Loader引导开机，然后依次进入 -> `Kernel` -> `Native` -> `Framework` -> `App`，接来下简要说说每个过程：

关于Loader层：

- Boot ROM: 当手机处于关机状态时，长按Power键开机，引导芯片开始从固化在`ROM`里的预设代码开始执行，然后加载引导程序到`RAM`；
- Boot Loader：这是启动Android系统之前的引导程序，主要是检查RAM，初始化硬件参数等功能。

### Linux内核

Android 平台的基础是 Linux内核，例如，Android Runtime（ART）最终调用Linux内核来执行底层功能，例如线程和低层内存管理。Linux 内核的安全机制为Android提供相应的保障

### 硬件抽象层（HAL）

硬件抽象层（HAL）提供标准接口，HAL包含多个库模块，其中每个模块都为特定类型的硬件组件实现一组接口。比如WiFi/蓝牙模块，当框架API请求访问设备硬件时，Android系统将为该硬件加载相应的库模块。

### Android Runtime

每个应用都在其自己的进程中运行，都有自己的虚拟机实例。ART通过执行DEX文件可在设备运行多个虚拟机，DEX文件是一种专为Android设计的字节码格式文件，经过优化，使用内存很少。

ART主要功能包括：

- 预先(AOT)和即时(JIT)编译，
- 优化的垃圾回收(GC)
- 调试相关的支持。

### Native C/C++ Library

许多核心 Android 系统组件和服务（例如 ART 和 HAL）构建自原生代码，需要以 C 和 C++ 编写的原生库。Android 平台提供 Java 框架 API 以向应用显示其中部分原生库的功能。

这里的Naive 系统库主要包括init 孵化来的用户空间的守护进程、HAL层及开机动画等。

启动init进程（pid=1}，是Linux系统的用户进程，**`init进程是所有用户进程的鼻祖。`**

- init进程会孵化出ueventd、logd、healthd、installd、adbd、lmkd 等用户守护进程；
- init 进程还启动`servicemanager`（Binder服务管家，指Native层的ServiceManager（C++））、`bootanim`(开机动画)等重要服务；
- init 进程还孵化出 Zygote 进程
  - **`Zygote 进程是Android系统的第一个Java进程（即虚拟机进程）`**，
  - **`Zygote是所有Java进程的父进程`**，
  - Zygote 进程本身是有init 进程孵化而来的

### Java API Framework

- Zygote 进程，是由init进程通过解析init.rc文件后fork生成，Zygote 进程主要包含：
  - 加载ZygoteInit 类，注册 Zygote Socket 服务端套接字
  - 加载虚拟机
  - 提前加载类 preloadClasses
  - 提前加载资源preloadResources

- System Server 进程，是由Zygote 进程fork而来，**`System Server 是 zygote 孵化出来的第一个进程`**，System Server 负责启动和管理整个 Java Framework，包含：
  - ActivityManager
  - WindowManager
  - PackageManager
  - PowerManager
- Media Server 进程，是由init 进程 fork而来，负责启动和管理整个C++ framework，包含：AudioFlinger、Camera Service 等服务。

### APP系统应用

- Zygote孵化出来的第一个APP进程就是Launcher，这是用户看到的桌面APP
- Zygote 进程还会创建 Browser、Phone、Email等APP进程，每个APP至少运行一个进程。
- 所有的APP进程都是由Zygote进程fork生成的。

### Syscall && JNI

- Native与Kernel之间有一层系统调用(SysCall)层，见[Linux系统调用(Syscall)原理](http://gityuan.com/2016/05/21/syscall/);
- Java层与Native(C/C++)层之间的纽带JNI，见[Android JNI原理分析](http://gityuan.com/2016/05/28/android-jni/)。



## 通信方式

### Linux 进程间通信方式

IPC（Inter-Process Communication，进程间通信），Linux有以下几种机制：

- 管道

  在创建时分配一个page大小的内存，缓存区大小比较有限；

- 消息队列

  信息复制两次，额外的CPU消耗；不合适频繁或信息量大的通信；

- 共享内存

  无须复制，共享缓冲区直接附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决；

- 套接字

  作为更通用的接口，传输效率低，主要用于不同机器或跨网络的通信；

- 信号量

  常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。

- 信号

  不适用于信息交换，更适用于进程中断控制，比如非法内存访问，杀死某个进程等；

### Android进程间通信方式

Android额外还有 Binder IPC机制，

- Android OS 中的Zygote进程的IPC采用的是 Socket机制
- 上层 System Server、Media Server 以及上层APP之间更多的是采用 Binder IPC方式来完成跨进程之间的通信。
- 杀进程 Process.killProcess() 采用的是 signal方式

下面来说一下 Binder 和 Socket

#### Binder

Binder 作为Android系统提供的一种IPC机制，无论从系统开发还是应用开发，都是Android系统中最重要的组成。

##### IPC原理

从进程角度来看IPC机制

![binder_interprocess_communication](http://gityuan.com/images/binder/prepare/binder_interprocess_communication.png)



每个Android的进程，只能运行在自己进程所拥有的虚拟地址空间。对应一个4GB的虚拟地址空间，其中3GB是用户空间，1GB是内核空间，当然内核空间的大小是可以通过参数配置调整的。对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。Client进程向Server进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，Client端与Server端进程往往采用ioctl等方法跟内核空间的驱动进行交互。



**ioctl**(input/output control)是一个专用于设备输入输出操作的系统调用,该调用传入一个跟设备有关的请求码，系统调用的功能完全取决于请求码。

举个例子，CD-ROM驱动程序可以弹出光驱，它就提供了一个对应的**Ioctl**请求码。设备无关的请求码则提供了内核调用权限。



##### Binder 原理

Binder通信采用 C/S 架构，从组件视角来说，包含 Client、Server、ServiceManager 以及Binder驱动，其中ServiceManager 用于管理系统中的各种服务。

![ServiceManager](http://gityuan.com/images/android-arch/IPC-Binder.jpg)



可以看出无论是注册服务和获取服务的过程都需要ServiceManager，需要注意的是此处的Service Manager是指Native层的ServiceManager（C++），并非指framework层的ServiceManager(Java)。ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程，要掌握Binder机制，首先需要了解系统是如何首次[启动Service Manager](http://gityuan.com/2015/11/07/binder-start-sm/)。当Service Manager启动之后，Client端和Server端通信时都需要先[获取Service Manager](http://gityuan.com/2015/11/08/binder-get-sm/)接口，才能开始通信服务。

图中Client/Server/ServiceManage之间的相互通信都是基于Binder机制。既然基于Binder机制通信，那么同样也是C/S架构，则图中的3大步骤都有相应的Client端与Server端。

1. **[注册服务(addService)](http://gityuan.com/2015/11/14/binder-add-service/)**：Server进程要先注册Service到ServiceManager。该过程：Server是客户端，ServiceManager是服务端。
2. **[获取服务(getService)](http://gityuan.com/2015/11/15/binder-get-service/)**：Client进程使用某个Service前，须先向ServiceManager中获取相应的Service。该过程：Client是客户端，ServiceManager是服务端。
3. **使用服务**：Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互。该过程：client是客户端，server是服务端。

图中的Client，Server，Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与[Binder驱动](http://gityuan.com/2015/11/01/binder-driver/)进行交互的，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client，Server，Service Manager位于用户空间。Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Server是Android的应用层，开发人员只需自定义实现client、Server端，借助Android的基本平台架构便可以直接进行IPC通信。

##### C/S 模式

BpBinder(客户端)和BBinder(服务端)都是Android中Binder通信相关的代表，它们都从IBinder类中派生而来，关系图如下：

![Binder关系图](http://gityuan.com/images/binder/prepare/Ibinder_classes.jpg)

- client端：BpBinder.transact()来发送事务请求；
- server端：BBinder.onTransact()会接收到相应事务。



#### Socket 套接字

Socket 通信方式也是C/S 架构的，比Binder简单很多。在Android系统中采用Socket通信方式的主要有：

- zygote：用于孵化进程，system_server创建进程是通过socket向zygote进程发起请求；
- installd：用于安装App的守护进程，上层PackageManagerService很多实现最终都是交给它来完成；
- lmkd：lowmemorykiller的守护进程，Java层的LowMemoryKiller最终都是由lmkd来完成；
- adbd：这个也不用说，用于服务adb；
- logcatd:这个不用说，用于服务logcat；
- vold：即volume Daemon，是存储类的守护进程，用于负责如USB、Sdcard等存储设备的事件处理。

等等还有很多，这里不一一列举，Socket方式更多的用于Android framework层与native层之间的通信。

来看看守护进程(也就是进程名一般以d为后缀，比如logd，此处d是指daemon的简称)

#### 为什么Android要采用Binder作为IPC机制？



## Android系统

### 系统启动

### Android进程

#### Android进程创建流程

对于系统工程师或者高级开发者，还是有很必要了解Android系统是如何一步步地创建出一个进程的。先来看一张进程创建过程的简要图：

![start_app_process](http://gityuan.com/images/android-process/start_app_process.jpg)

图解：

1. **App发起进程**：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；

2. **system_server进程**：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；

3. **zygote进程**：在执行`ZygoteInit.main()`后便进入`runSelectLoop()`循环体内，当有客户端连接时便会执行ZygoteConnection.runOnce()方法，再经过层层调用后fork出新的应用进程；

4. **新进程**：执行handleChildProc方法，最后调用ActivityThread.main()方法。

   

**详细流程：**

Process.start()方法是阻塞操作，等待直到进程创建完成并返回相应的新进程pid，才完成该方法。

当App第一次启动时或者启动远程Service，即AndroidManifest.xml文件中定义了process:remote属性时，都需要创建进程。

比如当用户点击桌面的某个App图标，桌面本身是一个app（即Launcher App），那么Launcher所在进程便是这次创建新进程的发起进程，

该通过binder发送消息给system_server进程，该进程承载着整个java framework的核心服务。system_server进程从Process.start开始，执行创建进程，流程图（以进程的视角）如下：

![process-create](http://gityuan.com/images/android-process/process-create.jpg)

上图中，`system_server`进程通过socket IPC通道向`zygote`进程通信，`zygote`在fork出新进程后由于fork**调用一次，返回两次**，即在zygote进程中调用一次，在zygote进程和子进程中各返回一次，从而能进入子进程来执行代码。该调用流程图的过程：

1. **system_server进程**（`即流程1~3`）：通过Process.start()方法发起创建新进程请求，会先收集各种新进程uid、gid、nice-name等相关的参数，然后通过socket通道发送给zygote进程；
2. zygote进程（`即流程4~12`）：接收到system_server进程发送过来的参数后封装成Arguments对象，图中绿色框forkAndSpecialize()方法是进程创建过程中最为核心的一个环节（`详见流程6`），其具体工作是依次执行下面的3个方法：
   - preFork()：先停止Zygote的4个Daemon子线程（java堆内存整理线程、对线下引用队列线程、析构线程以及监控线程）的运行以及初始化gc堆；
   - nativeForkAndSpecialize()：调用linux的fork()出新进程，创建Java堆处理的线程池，重置gc性能数据，设置进程的信号处理函数，启动JDWP线程；
   - postForkCommon()：在启动之前被暂停的4个Daemon子线程。
3. **新进程**（`即流程13~15`）：进入handleChildProc()方法，设置进程名，打开binder驱动，启动新的binder线程；然后设置art虚拟机参数，再反射调用目标类的main()方法，即Activity.main()方法。

再之后的流程，如果是startActivity则将要进入Activity的onCreate/onStart/onResume等生命周期；

如果是startService则将要进入Service的onCreate等生命周期。

system_server进程等待zygote返回进程创建完成(ZygoteConnection.handleParentProc)，一旦Zygote.forkAndSpecialize()方法执行完成，那么分道扬镳，zygote告知system_server进程进程已创建，而子进程继续执行后续的handleChildProc操作.

### 四大组件

#### Activity

##### Activity启动流程



![start_activity_process](http://gityuan.com/images/activity/start_activity_process.jpg)

**启动流程：**

1. 点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；
2. system_server进程接收到请求后，向zygote进程发送创建进程的请求；
3. Zygote进程fork出新的子进程，即App进程；
4. App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
5. system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；
6. App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；
7. 主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。

到此，App便正式启动，开始进入Activity生命周期，执行完onCreate/onStart/onResume方法，UI渲染结束后便可以看到App的主界面。

#### Service

##### Service 启动流程

![start_service_process](http://gityuan.com/images/android-service/start_service/start_service_processes.jpg)

图中涉及3种IPC通信方式：`Binder`、`Socket`以及`Handler`，在图中分别用3种不同的颜色来代表这3种通信方式。一般来说，同一进程内的线程间通信采用的是 [Handler消息队列机制](http://gityuan.com/2015/12/26/handler-message/)，不同进程间的通信采用的是[binder机制](http://gityuan.com/2015/10/31/binder-prepare/)，另外与Zygote进程通信采用的`Socket`。

启动流程：

1. Process A进程采用Binder IPC向system_server进程发起startService请求；
2. system_server进程接收到请求后，向zygote进程发送创建进程的请求；
3. zygote进程fork出新的子进程Remote Service进程；
4. Remote Service进程，通过Binder IPC向sytem_server进程发起attachApplication请求；
5. system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向remote Service进程发送scheduleCreateService请求；
6. Remote Service进程的binder线程在收到请求后，通过handler向主线程发送CREATE_SERVICE消息；
7. 主线程在收到Message后，通过发射机制创建目标Service，并回调Service.onCreate()方法。



##### Service 引起ANR

Service启动过程出现ANR，"executing service [发送超时serviceRecord信息]”， 这往往是service的onCreate()回调方法执行时间过长。

所以在Service 的 onCreate 方法中，不能执行耗时任务。

核心代码 在` ActivityThread.handleCreateService()`方法中

在bumpServiceExecutingLocked会发送一个延迟处理的消息SERVICE_TIMEOUT_MSG。

在方法scheduleCreateService执行完成，也就是onCreate回调执行完成之后，便会remove掉该消息。

但是如果没能在延时时间之内remove该消息，则会进入执行service timeout流程。

发送延时消息SERVICE_TIMEOUT_MSG,延时时长：

- 对于前台服务，则超时为SERVICE_TIMEOUT，即timeout=20s；
- 对于后台服务，则超时为SERVICE_BACKGROUND_TIMEOUT，即timeout=200s；

#### ContentProvider

##### ContentProvider 启动流程

**ContentProvider的作用是跨进程通信，实现数据共享**

我们在使用ContentProvider的时候，需要下面两步

- 新建一个类，继承ContentProvider，并实现方法
- 在AndroidManifest中注册这个ContentProvider

AndroidManifest 是 ActivityManagerService 进行处理的，AndroidManifest会在当前应用创建时被处理，ContentProvider 是伴随着进程的启动被创建的

![img](https://s1.ax1x.com/2020/08/19/d125lt.png)

ContentProvider是伴随着进程的启动而被创建的，进程之间的通信是通过Binder来实现的

- 应用进程创建后的代码入口是ActivityThread的main函数，然后他通过Binder机制把ApplicationThread发送给AMS进行缓存，以后AMS要访问该进程就是通过ApplicationThread来访问的。
- AMS做处理之后，通过ApplicationThread来让ActivityThread进行初始化。
- 在ActivityThread初始化中会创建ContextImp，Application和ContentProvider。
- 然后ActivityThread把ContentProvider发布到AMS中。以后如果有别的进程想要读取数据可以直接从AMS中获取到对应的ContentProvider。

##### 一个进程如何访问另外一个进程中的ContentProvider ?

涉及到跨进程通信，不可能直接以ContentProvider来通信，而是以IContentProvider来通信

![img](https://s1.ax1x.com/2020/08/19/d1rYZj.png)

整体的思路就是：获取到对应URI的IContentProvider，然后跨进程访问数据，所以重点在获取IContentProvider。

- 首先会查询本地是否有和URI对应的IContentProvider，有的话直接使用，没有的话去AMS获取。
- 如果ContentProvider对应的进程已启动，那么AMS一般是有该IContentProvider的，但是如果应用尚未启动，那么需要AMS去启动该进程。
- 进程B启动后会自然把IContentProvider发布给AMS，AMS先把他缓存起来，然后再返回给进程A
- 进程A拿到IContentProvider之后，就是直接跨进程访问ContentProvider了。

#### BroadcastReceiver

广播接收器的注册过程

<img src="https://s1.ax1x.com/2020/08/17/dnSOMQ.png" alt="Broadcast注册源码流程.png" style="zoom: 80%;" />



```java
//LoadedApk.java

    final IIntentReceiver.Stub mIIntentReceiver;
   
        ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
         
            mIIntentReceiver = new InnerReceiver(this, !registered);
            
        }

        IIntentReceiver getIIntentReceiver() {
            return mIIntentReceiver;
        }

//ContextImpl.java
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            //调用AMS的registerReceiverWithFeature（）rd就是 mIIntentReceiver
            final Intent intent = ActivityManager.getService().registerReceiverWithFeature(
                    mMainThread.getApplicationThread(), mBasePackageName, getAttributionTag(), rd,
                    filter, broadcastPermission, userId, flags);
           
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }


//AndroidManagerService.java
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId,
            int flags) {
        return registerReceiverWithFeature(caller, callerPackage, null, receiver, filter,
                permission, userId, flags);
    }


//IIntentReceiver 就是 InnerReceiver
	public Intent registerReceiverWithFeature(IIntentReceiver receiver){

        //......
        synchronized (this) {
            
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    final int totalReceiversForApp = rl.app.receivers.size();
                    if (totalReceiversForApp >= MAX_RECEIVERS_ALLOWED_PER_APP) {
                        throw new IllegalStateException("Too many receivers, total of "
                                + totalReceiversForApp + ", registered for pid: "
                                + rl.pid + ", callerPackage: " + callerPackage);
                    }
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } 
            //....

            return sticky;
        }
    }
```



Receiver是没有跨进程通信能力的，而广播需要AMS的调控，所以必须有一个可以跟AMS沟通的对象，这个对象是InnerReceiver，而ReceiverDispatcher就是负责维护他们两个的联系，如下图：

![img](https://s1.ax1x.com/2020/10/11/0gkmW9.png)

```java
//LoadApk.java
    static final class ReceiverDispatcher {

        //通过InnerReceiver实现跨进程通信（和AMS进行通信）
        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }

            @Override
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                final LoadedApk.ReceiverDispatcher rd;
                if (intent == null) {
                    Log.wtf(TAG, "Null intent received");
                    rd = null;
                } else {
                    rd = mDispatcher.get();
                }
               
                if (rd != null) {
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                }
                //....
            }
        }


        public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            final Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);

            //执行args.getRunnable()
            if (intent == null || !mActivityThread.post(args.getRunnable())) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManager.getService();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing sync broadcast to " + mReceiver);
                    args.sendFinished(mgr);
                }
            }
        }
        
        
        final class Args extends BroadcastReceiver.PendingResult {

            public final Runnable getRunnable() {
                return () -> {
                    final BroadcastReceiver receiver = mReceiver;

                    //调用receiver.onReceive()
                    try {
                        ClassLoader cl = mReceiver.getClass().getClassLoader();
                        intent.setExtrasClassLoader(cl);
                        intent.prepareToEnterProcess();
                        setExtrasClassLoader(cl);
                        receiver.setPendingResult(this);
                        receiver.onReceive(mContext, intent);
                    } catch (Exception e) {
                       
                    }

                    if (receiver.getPendingResult() != null) {
                        finish();
                    }
                   
                };
            }
        }



    }
```



### 系统服务

### 内存和存储

### ANR

ANR(Application Not responding)，是指应用程序未响应，Android系统对于一些事件需要在一定的时间范围内完成，如果超过预定时间能未能得到有效响应或者响应时间过长，都会造成ANR。

当应用程序一段时间无法及时响应，则会弹出ANR对话框，让用户选择继续等待，还是强制关闭。

#### ANR触发机制

ANR是一套监控Android应用响应是否及时的机制，可以把发生ANR比作是引爆炸弹，那么整个流程包含三部分组成：

1. 埋定时炸弹：中控系统(system_server进程)启动倒计时，在规定时间内如果目标(应用进程)没有干完所有的活，则中控系统会定向炸毁(杀进程)目标。
2. 拆炸弹：在规定的时间内干完工地的所有活，并及时向中控系统报告完成，请求解除定时炸弹，则幸免于难。
3. 引爆炸弹：中控系统立即封装现场，抓取快照，搜集目标执行慢的罪证(traces)，便于后续的案件侦破(调试分析)，最后是炸毁目标。

常见的ANR有service、broadcast、provider以及input

##### Service 超时机制

下面来看看埋炸弹与拆炸弹在整个服务启动(startService)过程所处的环节。

<img src="http://gityuan.com/images/android-anr/service_anr.jpg" alt="service_anr" style="zoom:80%;" />

图解1：

1. 客户端(App进程)向中控系统(system_server进程)发起启动服务的请求
2. 中控系统派出一名空闲的通信员(binder_1线程)接收该请求，紧接着向组件管家(ActivityManager线程)发送消息，埋下定时炸弹
3. 通讯员1号(binder_1)通知工地(service所在进程)的通信员准备开始干活
4. 通讯员3号(binder_3)收到任务后转交给包工头(main主线程)，加入包工头的任务队列(MessageQueue)
5. 包工头经过一番努力干完活(完成service启动的生命周期)，然后等待SharedPreferences(简称SP)的持久化；
6. 包工头在SP执行完成后，立刻向中控系统汇报工作已完成
7. 中控系统的通讯员2号(binder_2)收到包工头的完工汇报后，立刻拆除炸弹。如果在炸弹倒计时结束前拆除炸弹则相安无事，否则会引发爆炸(触发ANR)

##### Broadcast 超时机制

broadcast跟service超时机制大抵相同，对于静态注册的广播在超时检测过程需要检测SP，如下图所示。

<img src="http://gityuan.com/images/android-anr/broadcast_anr.jpg" alt="broadcast_anr" style="zoom:80%;" />

图解2：

1. 客户端(App进程)向中控系统(system_server进程)发起发送广播的请求
2. 中控系统派出一名空闲的通信员(binder_1)接收该请求转交给组件管家(ActivityManager线程)
3. 组件管家执行任务(processNextBroadcast方法)的过程埋下定时炸弹
4. 组件管家通知工地(receiver所在进程)的通信员准备开始干活
5. 通讯员3号(binder_3)收到任务后转交给包工头(main主线程)，加入包工头的任务队列(MessageQueue)
6. 包工头经过一番努力干完活(完成receiver启动的生命周期)，发现当前进程还有SP正在执行写入文件的操作，便将向中控系统汇报的任务交给SP工人(queued-work-looper线程)
7. SP工人历经艰辛终于完成SP数据的持久化工作，便可以向中控系统汇报工作完成
8. 中控系统的通讯员2号(binder_2)收到包工头的完工汇报后，立刻拆除炸弹。如果在倒计时结束前拆除炸弹则相安无事，否则会引发爆炸(触发ANR)

（说明：SP从8.0开始采用名叫“queued-work-looper”的handler线程，在老版本采用newSingleThreadExecutor创建的单线程的线程池）

如果是动态广播，或者静态广播没有正在执行持久化操作的SP任务，则不需要经过“queued-work-looper”线程中转，而是直接向中控系统汇报，流程更为简单，如下图所示：

<img src="http://gityuan.com/images/android-anr/broadcast_anr_2.jpg" alt="broadcast_anr_2" style="zoom:80%;" />

可见，只有XML静态注册的广播超时检测过程会考虑是否有SP尚未完成，动态广播并不受其影响。

SP的apply将修改的数据项更新到内存，然后再异步同步数据到磁盘文件，因此很多地方会推荐在主线程调用采用apply方式，避免阻塞主线程，但静态广播超时检测过程需要SP全部持久化到磁盘，如果过度使用apply会增大应用ANR的概率，更多细节详见http://gityuan.com/2017/06/18/SharedPreferences

Google这样设计的初衷是针对静态广播的场景下，保障进程被杀之前一定能完成SP的数据持久化。因为在向中控系统汇报广播接收者工作执行完成前，该进程的优先级为Foreground级别，高优先级下进程不但不会被杀，而且能分配到更多的CPU时间片，加速完成SP持久化。

更多细节详见Android Broadcast广播机制分析，http://gityuan.com/2016/06/04/broadcast-receiver

##### Provider超时机制

provider的超时是在provider进程首次启动的时候才会检测，当provider进程已启动的场景，再次请求provider并不会触发provider超时。

<img src="http://gityuan.com/images/android-anr/provider_anr.jpg" alt="provider_anr" style="zoom:80%;" />

图解3：

1. 客户端(App进程)向中控系统(system_server进程)发起获取内容提供者的请求
2. 中控系统派出一名空闲的通信员(binder_1)接收该请求，检测到内容提供者尚未启动，则先通过zygote孵化新进程
3. 新孵化的provider进程向中控系统注册自己的存在
4. 中控系统的通信员2号接收到该信息后，向组件管家(ActivityManager线程)发送消息，埋下炸弹
5. 通信员2号通知工地(provider进程)的通信员准备开始干活
6. 通讯员4号(binder_4)收到任务后转交给包工头(main主线程)，加入包工头的任务队列(MessageQueue)
7. 包工头经过一番努力干完活(完成provider的安装工作)后向中控系统汇报工作已完成
8. 中控系统的通讯员3号(binder_3)收到包工头的完工汇报后，立刻拆除炸弹。如果在倒计时结束前拆除炸弹则相安无事，否则会引发爆炸(触发ANR)

##### input 超时机制

input的超时检测机制跟service、broadcast、provider截然不同，为了更好的理解input过程先来介绍两个重要线程的相关工作：

- InputReader线程负责通过EventHub(监听目录/dev/input)读取输入事件，一旦监听到输入事件则放入到InputDispatcher的mInBoundQueue队列，并通知其处理该事件；
- InputDispatcher线程负责将接收到的输入事件分发给目标应用窗口，分发过程使用到3个事件队列：
  - mInBoundQueue用于记录InputReader发送过来的输入事件；
  - outBoundQueue用于记录即将分发给目标应用窗口的输入事件；
  - waitQueue用于记录已分发给目标应用，且应用尚未处理完成的输入事件；

input的超时机制并非时间到了一定就会爆炸，而是处理后续上报事件的过程才会去检测是否该爆炸，所以更像是扫雷的过程，具体如下图所示。

<img src="http://gityuan.com/images/android-anr/input_anr.jpg" alt="input_anr" style="zoom:80%;" />

图解4：

1. InputReader线程通过EventHub监听底层上报的输入事件，一旦收到输入事件则将其放至mInBoundQueue队列，并唤醒InputDispatcher线程
2. InputDispatcher开始分发输入事件，设置埋雷的起点时间。先检测是否有正在处理的事件(mPendingEvent)，如果没有则取出mInBoundQueue队头的事件，并将其赋值给mPendingEvent，且重置ANR的timeout；否则不会从mInBoundQueue中取出事件，也不会重置timeout。然后检查窗口是否就绪(checkWindowReadyForMoreInputLocked)，满足以下任一情况，则会进入扫雷状态(检测前一个正在处理的事件是否超时)，终止本轮事件分发，否则继续执行步骤3。
   - 对于按键类型的输入事件，则outboundQueue或者waitQueue不为空，
   - 对于非按键的输入事件，则waitQueue不为空，且等待队头时间超时500ms
3. 当应用窗口准备就绪，则将mPendingEvent转移到outBoundQueue队列
4. 当outBoundQueue不为空，且应用管道对端连接状态正常，则将数据从outboundQueue中取出事件，放入waitQueue队列
5. InputDispatcher通过socket告知目标应用所在进程可以准备开始干活
6. App在初始化时默认已创建跟中控系统双向通信的socketpair，此时App的包工头(main线程)收到输入事件后，会层层转发到目标窗口来处理
7. 包工头完成工作后，会通过socket向中控系统汇报工作完成，则中控系统会将该事件从waitQueue队列中移除。

input超时机制为什么是扫雷，而非定时爆炸呢？是由于对于input来说即便某次事件执行时间超过timeout时长，只要用户后续在没有再生成输入事件，则不会触发ANR。 这里的扫雷是指当前输入系统中正在处理着某个耗时事件的前提下，后续的每一次input事件都会检测前一个正在处理的事件是否超时（进入扫雷状态），检测当前的时间距离上次输入事件分发时间点是否超过timeout时长。如果前一个输入事件，则会重置ANR的timeout，从而不会爆炸。

##### ANR超时阈值

不同组件的超时阈值各有不同，关于service、broadcast、contentprovider以及input的超时阈值如下表：

<img src="http://gityuan.com/images/android-anr/anr_timeout.jpg" alt="anr_timeout" style="zoom:80%;" />

##### 前台服务与后台服务的区别

系统对前台服务启动的超时为20s，而后台服务超时为200s，那么系统是如何区别前台还是后台服务呢？来看看ActiveServices的核心逻辑：

```java
    ComponentName startServiceLocked(...) throws TransactionTooLargeException {
        final boolean callerFg;
        if (caller != null) {
            final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
            if (callerApp == null) {
                throw new SecurityException(
                        "Unable to find app for caller " + caller
                        + " (pid=" + callingPid
                        + ") when starting service " + service);
            }
            callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        } else {
            callerFg = true;
        }
        //...
        ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
        return cmp;
    }
```

在startService过程根据发起方进程callerApp所属的进程调度组来决定被启动的服务是属于前台还是后台。当发起方进程不等于ProcessList.SCHED_GROUP_BACKGROUND(后台进程组)则认为是前台服务，否则为后台服务，并标记在ServiceRecord的成员变量createdFromFg。

什么进程属于SCHED_GROUP_BACKGROUND调度组呢？进程调度组大体可分为TOP、前台、后台，进程优先级（Adj）和进程调度组（SCHED_GROUP）算法较为复杂，其对应关系可粗略理解为Adj等于0的进程属于Top进程组，Adj等于100或者200的进程属于前台进程组，Adj大于200的进程属于后台进程组。关于Adj的含义见下表，简单来说就是Adj>200的进程对用户来说基本是无感知，主要是做一些后台工作，故后台服务拥有更长的超时阈值，同时后台服务属于后台进程调度组，相比前台服务属于前台进程调度组，分配更少的CPU时间片。

![adj](http://gityuan.com/images/android-anr/adj.png)

`前台服务准确来说，是指由处于前台进程调度组的进程发起的服务`。

这跟常说的fg-service服务有所不同，fg-service是指挂有前台通知的服务。

##### 前台广播和后台广播超时

前台广播超时为10s，后台广播超时为60s，那么如何区分前台和后台广播呢？来看看AMS的核心逻辑：

```java
    BroadcastQueue broadcastQueueForIntent(Intent intent) {
        if (isOnOffloadQueue(intent.getFlags())) {
            return mOffloadBroadcastQueue;
        }

        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }
    
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", foreConstants, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", backConstants, true);
```



根据发送广播sendBroadcast(Intent intent)中的intent的flags是否包含`FLAG_RECEIVER_FOREGROUND`来决定把该广播是放入前台广播队列或者后台广播队列，前台广播队列的超时为10s，后台广播队列的超时为60s，默认情况下广播是放入后台广播队列，除非指明加上`FLAG_RECEIVER_FOREGROUND`标识。

后台广播比前台广播拥有更长的超时阈值，同时在广播分发过程遇到后台service的启动(mDelayBehindServices)会延迟分发广播，等待service的完成，因为等待service而导致的广播ANR会被忽略掉；后台广播属于后台进程调度组，而前台广播属于前台进程调度组。简而言之，后台广播更不容易发生ANR，同时执行的速度也会更慢。

另外，只有串行处理的广播才有超时机制，因为接收者是串行处理的，前一个receiver处理慢，会影响后一个receiver；并行广播通过一个循环一次性向所有的receiver分发广播事件，所以不存在彼此影响的问题，则没有广播超时。

`前台广播准确来说，是指位于前台广播队列的广播`。

##### 前台和后台ANR

除了前台服务，前台广播，还有前台ANR可能会让你云里雾里的，来看看其中核心逻辑：

```java
//ProcessRecord.java
void appNotResponding(String activityShortComponentName, ApplicationInfo aInfo,
            String parentShortComponentName, WindowProcessController parentProcess,
            boolean aboveSystem, String annotation, boolean onlyDumpSelf) {
       
        final boolean isSilentAnr;
        synchronized (mService) {
            // PowerManager.reboot() can block for a long time, so ignore ANRs while shutting down.
            if (mService.mAtmInternal.isShuttingDown()) {
                Slog.i(TAG, "During shutdown skipping ANR: " + this + " " + annotation);
                return;
            } else if (isNotResponding()) {
                Slog.i(TAG, "Skipping duplicate ANR: " + this + " " + annotation);
                return;
            } else if (isCrashing()) {
                Slog.i(TAG, "Crashing app skipping ANR: " + this + " " + annotation);
                return;
            } else if (killedByAm) {
                Slog.i(TAG, "App already killed by AM skipping ANR: " + this + " " + annotation);
                return;
            } else if (killed) {
                Slog.i(TAG, "Skipping died app ANR: " + this + " " + annotation);
                return;
            }


            //......

            //弹出ANR对话框
            if (mService.mUiHandler != null) {
                // Bring up the infamous App Not Responding dialog
                Message msg = Message.obtain();
                msg.what = ActivityManagerService.SHOW_NOT_RESPONDING_UI_MSG;
                msg.obj = new AppNotRespondingDialog.Data(this, aInfo, aboveSystem);

                mService.mUiHandler.sendMessage(msg);
            }
        }
    }

```

决定是前台或者后台ANR取决于该应用发生ANR时对用户是否可感知，比如拥有当前前台可见的activity的进程，或者拥有前台通知的fg-service的进程，这些是用户可感知的场景，发生ANR对用户体验影响比较大，故需要弹框让用户决定是否退出还是等待，如果直接杀掉这类应用会给用户造成莫名其妙的闪退。

后台ANR相比前台ANR，只抓取发生无响应进程的trace，也不会收集CPU信息，并且会在后台直接杀掉该无响应的进程，不会弹框提示用户。

`前台ANR准确来说，是指对用户可感知的进程发生的ANR`。

##### ANR收集信息

对于service、broadcast、provider、input发生ANR后，中控系统会马上去抓取现场的信息，用于调试分析。收集的信息包括如下：

- 将am_anr信息输出到EventLog，也就是说ANR触发的时间点最接近的就是EventLog中输出的am_anr信息
- 收集以下重要进程的各个线程调用栈trace信息，保存在data/anr/traces.txt文件
  - 当前发生ANR的进程，system_server进程以及所有persistent进程
  - audioserver, cameraserver, mediaserver, surfaceflinger等重要的native进程
  - CPU使用率排名前5的进程
- 将发生ANR的reason以及CPU使用情况信息输出到main log
- 将traces文件和CPU使用情况信息保存到dropbox，即data/system/dropbox目录
- 对用户可感知的进程则弹出ANR对话框告知用户，对用户不可感知的进程发生ANR则直接杀掉

整个ANR信息收集过程比较耗时，其中抓取进程的trace信息，每抓取一个等待200ms，可见persistent越多，等待时间越长。关于抓取trace命令，对于Java进程可通过在adb shell环境下执行kill -3 [pid]可抓取相应pid的调用栈；对于Native进程在adb shell环境下执行debuggerd -b [pid]可抓取相应pid的调用栈。对于ANR问题发生后的蛛丝马迹(trace)在traces.txt和dropbox目录中保存记录。更多细节详见理解Android ANR的信息收集过程，http://gityuan.com/2016/12/02/app-not-response。

有了现场信息，可以调试分析，先定位发生ANR时间点，然后查看trace信息，接着分析是否有耗时的message、binder调用，锁的竞争，CPU资源的抢占，以及结合具体场景的上下文来分析，调试手段就需要针对前面说到的message、binder、锁等资源从系统角度细化更多debug信息，这里不再展开，后续再以ANR案例来讲解。

作为应用开发者应让主线程尽量只做UI相关的操作，避免耗时操作，比如过度复杂的UI绘制，网络操作，文件IO操作；避免主线程跟工作线程发生锁的竞争，减少系统耗时binder的调用，谨慎使用sharePreference，注意主线程执行provider query操作。简而言之，尽可能减少主线程的负载，让其空闲待命，以期可随时响应用户的操作。

## APK 打包流程

![20a3e5cd6be0ec23de8a0d42a985a6eb.png](https://img-blog.csdnimg.cn/img_convert/20a3e5cd6be0ec23de8a0d42a985a6eb.png)



Android包文件apk分为两个部分，代码文件和资源文件，所以分别将代码文件打包、资源文件打包

- 通过aapt工具，将资源文件打包（AndroidManifest.xml 文件，各种xml资源文件等）的打包，生层R.java 文件
- 通过aidl工具处理AIDL文件，生成相应的Java文件
- 通过javac工具编译项目源码，生成.class文件
- 通过dex工具将所有的.class文件和架包转换成dex文件，该过程主要完成Java字节码转换成Dalvik字节码，才能给Android虚拟机使用
- 通过apkbuilder 工具，将资源文件和dex文件打包生成未签名的apk文件
- 通过Jarsigner 工具对apk进行签名
- 通过zipalign 工具进行对齐处理，对齐的过程就是将apk中的资源文件距离文件的起始位置都偏移四个字节的整数倍。这样通过内存映射访问apk的速度会更快。



## Android 类加载机制

我们知道Android系统也是仿照java搞了一个虚拟机，不过它不叫JVM，它叫Dalvik/ART VM。

Dalvik/ART VM 虚拟机加载类和资源也是要用到ClassLoader，不过Jvm通过ClassLoader加载的class字节码，而Dalvik/ART VM通过ClassLoader加载则是dex。

Android的类加载器分为两种：PathClassLoader和DexClassLoader，两者都继承自BaseDexClassLoader

### Dalvik虚拟机类加载流程

![img](https://upload-images.jianshu.io/upload_images/12972541-b2f0739d340718d7?imageMogr2/auto-orient/strip|imageView2/2/w/784/format/webp)



## Dalvik虚拟机类加载器源码分析

Android的类加载器主要有两个PathClassLoader和DexClassLoader，其中PathClassLoader是默认的类加载器，下面我们就来说说两者的区别与联系。

- **PathClassLoader**：支持加载DEX或者已经安装的APK（因为存在缓存的DEX）。
- **DexClassLoader**：支持加载APK、DEX和JAR，也可以从SD卡进行加载。

DexClassLoader和PathClassLoader都属于符合双亲委派模型的类加载器（因为它们没有重载loadClass方法）。也就是说，它们在加载一个类之前，会去检查自己以及自己以上的类加载器是否已经加载了这个类。如果已经加载过了，就会直接将之返回，而不会重复加载。

```
libcore/dalvik/src/main/java/dalvik/system/
    - PathClassLoader.java
    - DexClassLoader.java
    - BaseDexClassLoader.java
    - DexPathList.java
    - DexFile.java

art/runtime/native/dalvik_system_DexFile.cc

libcore/ojluni/src/main/java/java/lang/ClassLoader.java
```

#### ClassLoader

```Java
//ClassLoader.class
public abstract class ClassLoader {

    public Class<?> loadClass(String className) throws ClassNotFoundException {
        return loadClass(className, false);
    }

    protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        //判断当前类加载器是否已经加载过指定类，若已加载则直接返回
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) { 
            //如果没有加载过，则调用parent的类加载递归加载该类，若已加载则直接返回
            clazz = parent.loadClass(className, false);
            
            if (clazz == null) {
                //还没加载，则调用当前类加载器来加载
                clazz = findClass(className);
            }
        }
        return clazz;
    }
}
```

该方法的加载流程如下：

1. 判断当前类加载器是否已经加载过指定类，若已加载则直接返回，否则继续执行；
2. 调用parent的类加载递归加载该类，检测是否加载，若已加载则直接返回，否则继续执行；
3. 调用当前类加载器，通过findClass加载。

```java
//ClassLoader.class
protected final Class<?> findLoadedClass(String name) {
    ClassLoader loader;
    if (this == BootClassLoader.getInstance())
        loader = null;
    else
        loader = this;
    return VMClassLoader.findLoadedClass(loader, name);
}
```

#### BaseDexClassLoader

```java
  public class BaseDexClassLoader extends ClassLoader { 
      private final DexPathList pathList;
      protected final ClassLoader[] sharedLibraryLoaders;
      
       @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            // First, check whether the class is present in our shared libraries.
            if (sharedLibraryLoaders != null) {
                for (ClassLoader loader : sharedLibraryLoaders) {
                    try {
                        return loader.loadClass(name);
                    } catch (ClassNotFoundException ignored) {
                    }
                }
            }
            // Check whether the class in question is present in the dexPath that
            // this classloader operates on.
            List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
            Class c = pathList.findClass(name, suppressedExceptions);
            if (c == null) {
                ClassNotFoundException cnfe = new ClassNotFoundException(
                        "Didn't find class \"" + name + "\" on path: " + pathList);
                for (Throwable t : suppressedExceptions) {
                    cnfe.addSuppressed(t);
                }
                throw cnfe;
            }
            return c;
        }
      
      
      public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;
            if (dex != null) {
                //找到目标类，则直接返回
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        return null;
		}
    }
```

#### PathClassLoader 加载类的过程

此处以PathClassLoader为例来说明类的加载过程，先初始化，然后执行loadClass()方法来加载相应的类。 例如：

```java
new PathClassLoader("/system/framework/tcmclient.jar", ClassLoader.getSystemClassLoader());
```

##### 初始化

```java
public class PathClassLoader extends BaseDexClassLoader {
   
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }

    /**
     * @hide
     */
    @libcore.api.CorePlatformApi
    public PathClassLoader(
            String dexPath, String librarySearchPath, ClassLoader parent,
            ClassLoader[] sharedLibraryLoaders) {
        super(dexPath, librarySearchPath, parent, sharedLibraryLoaders);
    }
}

public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList;
    
        public BaseDexClassLoader(String dexPath,
            String librarySearchPath, ClassLoader parent, ClassLoader[] sharedLibraryLoaders,
            boolean isTrusted) {
        super(parent);
        // Setup shared libraries before creating the path list. ART relies on the class loader
        // hierarchy being finalized before loading dex files.
        this.sharedLibraryLoaders = sharedLibraryLoaders == null
                ? null
                : Arrays.copyOf(sharedLibraryLoaders, sharedLibraryLoaders.length);
            //收集dex文件和Native动态库
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);

        reportClassLoaderChain();
    }
}

public abstract class ClassLoader {
    private ClassLoader parent;  //父类加载器

    protected ClassLoader(ClassLoader parent) {
        this(checkCreateClassLoader(), parent);
    }

    private ClassLoader(Void unused, ClassLoader parent) {
        this.parent = parent;
    }
}
```

##### DexPathList

```java
public final class DexPathList {
    private Element[] dexElements;
        /** List of application native library directories. */
    private final List<File> nativeLibraryDirectories;

    /** List of system native library directories. */
    private final List<File> systemNativeLibraryDirectories;
    
     DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
       

        this.definingContext = definingContext;

        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        ////记录所有的dexFile文件
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);

        // Native libraries may exist in both the system and
        // application library paths, and we use this search order:
        //
        //   1. This class loader's library path for application libraries (librarySearchPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        //
        // This order was reversed prior to Gingerbread; see http://b/2933456.
         //app目录的native库
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
         //系统目录的native库
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
         //记录所有的Native动态库
        this.nativeLibraryPathElements = makePathElements(getAllNativeLibraryDirectories());

    }
    
}
```

DexPathList初始化过程,主要功能是收集以下两个变量信息:

1. dexElements: 根据多路径的分隔符“;”将dexPath转换成File列表，记录所有的dexFile
2. nativeLibraryPathElements: 记录所有的Native动态库, 包括app目录的native库和系统目录的native库。

##### makeDexElements

该方法的主要功能是创建Element数组

```java
public final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";

    private Element[] dexElements;
	private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
            List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
      Element[] elements = new Element[files.size()];
      int elementsPos = 0;
      /*
       * Open all files and load the (direct or contained) dex files up front.
       */
      for (File file : files) {
          if (file.isDirectory()) {
              // We support directories for looking up resources. Looking up resources in
              // directories is useful for running libcore tests.
              elements[elementsPos++] = new Element(file);
          } else if (file.isFile()) {
              String name = file.getName();

              DexFile dex = null;
              //匹配以.dex为后缀的文件
              if (name.endsWith(DEX_SUFFIX)) {
                  // Raw dex file (not inside a zip/jar).
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                      if (dex != null) {
                          elements[elementsPos++] = new Element(dex, null);
                      }
                  } catch (IOException suppressed) {
                      System.logE("Unable to load dex file: " + file, suppressed);
                      suppressedExceptions.add(suppressed);
                  }
              } else {
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                  } catch (IOException suppressed) {
                      suppressedExceptions.add(suppressed);
                  }

                  if (dex == null) {
                      elements[elementsPos++] = new Element(file);
                  } else {
                      elements[elementsPos++] = new Element(dex, file);
                  }
              }
              if (dex != null && isTrusted) {
                dex.setTrusted();
              }
          } else {
              System.logW("ClassLoader referenced unknown path: " + file);
          }
      }
      if (elementsPos != elements.length) {
          elements = Arrays.copyOf(elements, elementsPos);
      }
      return elements;
    }

```

PathClassLoader创建完成后，就已经拥有了目标程序的文件路径，native lib路径，以及parent类加载器对象。接下来开始执行loadClass()来加载相应的类。

##### loadClass()

```java
public abstract class ClassLoader {
    
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
    
	protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
    
        protected final Class<?> findLoadedClass(String name) {
        ClassLoader loader;
        if (this == BootClassLoader.getInstance())
            loader = null;
        else
            loader = this;
        return VMClassLoader.findLoadedClass(loader, name);
    }
    
}
```

该方法的加载流程如下：

1. 判断当前类加载器是否已经加载过指定类，若已加载则直接返回，否则继续执行；
2. 调用parent的类加载递归加载该类，检测是否加载，若已加载则直接返回，否则继续执行；
3. 调用当前类加载器，通过findClass加载。

##### DexPathList.findClass()

```java
//BaseDexClassLoader.java
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // First, check whether the class is present in our shared libraries.
        if (sharedLibraryLoaders != null) {
            for (ClassLoader loader : sharedLibraryLoaders) {
                try {
                    return loader.loadClass(name);
                } catch (ClassNotFoundException ignored) {
                }
            }
        }
        // Check whether the class in question is present in the dexPath that
        // this classloader operates on.
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException(
                    "Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }

//DexPathList.java
    public Class<?> findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            Class<?> clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }

        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
```

**功能说明：**

这里是核心逻辑，一个Classloader可以包含多个dex文件，每个dex文件被封装到一个Element对象，这些Element对象排列成有序的数组 dexElements。当查找某个类时，会遍历所有的dex文件，如果找到则直接返回，不再继续遍历dexElements。也就是说当两个类不同的dex中出现，会优先处理排在前面的dex文件，这便是热修复的核心精髓，将需要修复的类所打包的dex文件插入到dexElements前面。



几种类加载器：

- PathClassLoader: 主要用于系统和app的类加载器,其中optimizedDirectory为null, 采用默认目录/data/dalvik-cache/
- DexClassLoader: 可以从包含classes.dex的jar或者apk中，加载类的类加载器, 可用于执行动态加载,但必须是app私有可写目录来缓存odex文件. 能够加载系统没有安装的apk或者jar文件， 因此很多插件化方案都是采用DexClassLoader;
- BaseDexClassLoader: 比较基础的类加载器, PathClassLoader和DexClassLoader都只是在构造函数上对其简单封装而已.
- BootClassLoader: 作为父类的类构造器。

热修复核心逻辑：在DexPathList.findClass()过程，一个Classloader可以包含多个dex文件，每个dex文件被封装到一个Element对象，这些Element对象排列成有序的数组dexElements。当查找某个类时，会遍历所有的dex文件，如果找到则直接返回，不再继续遍历dexElements。也就是说当两个类不同的dex中出现，会优先处理排在前面的dex文件，这便是热修复的核心精髓，将需要修复的类所打包的dex文件插入到dexElements前面。

