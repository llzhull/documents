# Android 操作系统

## Android 虚拟机

### Dalvik虚拟机

Dalvik 执行的是Dex字节码，通过解释器执行。它使用的是JIT（just in time )技术来进行代码转译，每次执行应用的时候，Dalvik将程序的代码编译为机器语言执行，即：对执行的代码通过JIT生成本地机器指令来执行。

### ART虚拟机

ART（Android Run Time） 虚拟机，执行的是本地机器指令。它采用AOT（Ahead Of Time）技术，会在应用程序安装时就转换成机器语言，不再在执行时解释，从而优化了应用运行的速度。在内存管理方面，ART也有了较大的改进，对内存分配和回收都做了算法优化，降低了内存碎片化程度，回收时间也得以缩短。

## Android架构

为了能整体上大致了解Android系统涉及的知识层面，先来看一张Google官方提供的经典分层架构图，从下往上依次分为Linux内核、HAL、系统Native库和Android运行时环境、Java框架层以及应用层这5层架构，其中每一层都包含大量的子模块或子系统。

https://developer.android.google.cn/guide/platform/images/android-stack_2x.png

![Android 软件堆栈](https://developer.android.google.cn/guide/platform/images/android-stack_2x.png)

#### 系统启动架构图

Google提供的5层架构图很经典，但为了更进一步透视Android系统架构，本文更多的是以进程的视角，以分层的架构来诠释Android系统的全貌，阐述Android内部的环环相扣的内在联系。

![process_status](http://gityuan.com/images/android-arch/android-boot.jpg)

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
- Zygote 进程还会创建 Browser、Phone、Email等APP进程，每隔APP至少运行一个进程。
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

  无须复制，共享缓冲区直接付附加到进程虚拟地址空间，速度快；但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决；

- 套接字

  作为更通用的接口，传输效率低，主要用于不通机器或跨网络的通信；

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



### 系统服务

### 内存和存储