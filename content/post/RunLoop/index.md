---
title: ' RunLoop 梳理'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-09-11T00:00:00Z"
lastmod: "2020-09-17T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
image:
  placement: 2
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

# RunLoop 梳理
了解 **RunLoop** 之前不妨问自己这几个问题：

1. 什么是 `RunLoop` 与 `线程` 有什么关系？
2. 有哪些类型的 `RunLoop` 分别作用是什么？
3. 对外的 `API` 有哪些，其内部逻辑是怎样的？
4. `底层原理` 是什么？
5. iOS 系统的案例有哪些？
6. 第三方的案例有哪些？


-------

## 1. 什么是 `RunLoop` 与 `线程` 有什么关系？
`RunLoop` 其实是一个对象，主要用来管理 `线程` 对CPU 资源的占用控制，使线程在没有任务的时候进入睡眠状态，有任务的时候将其唤醒。从而控制线程以最优程度对系统 `CPU` 资源的占用。

**iOS** 系统不能直接创建 **RunLoop**, 可以通过：

| 两种获取方式： |
| --- |
| 1. CFRunLoopGetMain() |
| 2. CFRunLoopGetCurrent() |

只能在当前线程内部获取当前线程的 RunLoop, 获取主线程的除外；

**RunLoop** 的创建是在第一次获取的时候，当第一次获取 `RunLoop` 的时候，系统会初始化一个全局的字典，并创建主线程的 `RunLoop`, 其中 `key` 是 `pthread_t`, `value` 是 `CFRunloopRef`。

在创建的时候会注册一个回调，回调是在线程结束的时候进行销毁字典中管理该线程的 `RunLoop`。

-------


## 2. 有哪些类型的 `RunLoop` 分别作用是什么？
在了解对外的 `API` 有哪些之前我们先了解 `Core Foundation` 里面关于 `RunLoop` 的类型有哪些：

| 类型有以下 5 种 |
| --- |
| 1. CFRunLoopRef |
| 2. CFRunLoopMode |
| 3. CFRunLoopSourceRef |
| 4. CFRunLoopTimerRef |
| 5. CFRunLoopObserverRef |

-------


### 1. CFRunLoopRef
它是一个结构体, 其成员大致内容如下：

```
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```
根据源码以及通过系统 `lldb` 调试我们这里只了解几个关键的点:
1.     CFMutableSetRef _commonModes;
2.     CFMutableSetRef _commonModeItems;
3.     CFRunLoopModeRef _currentMode;
4.     CFMutableSetRef _modes;

#### _commonModes （common modes）

一个 `RunLoop` 可以把自己标记为 `Common` 的属性, 标记为 `Common` 属性就是把自己的名称添加至 `_commonModes` 中，添加至 `_commonModes` 后，每当 `RunLoop` 的内容发送变化时，系统会自动把 `_commonModeItems` 相关的 `timer/ observer/source` 同步至标记为 `Common` 的 `RunLoop` 中

主线程 RunLoop 里面有两个预置的 Mode:

* **kCFRunLoopDefaultMode**
主线程默认的 mode 平时的状态
* **UITrackingRunLoopMode**
ScrollView 滑动时的状态

这俩 mode 默认都被标记了 common 的属性。

使用：`po RunLoop.current` 即可看到

```
0 : <CFString 0x113a1a640 [0x10c8f7b80]>{contents = "UITrackingRunLoopMode"}
2 : <CFString 0x10ca74848 [0x10c8f7b80]>{contents = "kCFRunLoopDefaultMode"}
```

案例：
当我们平时创建 Timer 的时候，默认是加入了 `kCFRunLoopDefaultMode` 中，但是如果当前页面有 ScrollView 滑动后系统由 `kCFRunLoopDefaultMode` 切换为 `UITrackingRunLoopMode` 这个时候 Timer 就不会回调，

我们常规就是通过把 Timer 加入 `UITrackingRunLoopMode` 来解决.

```
RunLoop.current.add(timer, forMode: .tracking)
```

其实还要一种解决方式：
就是把 Timer 加入到 `_commonModeItems` 中
就是上面说的直接把 Timer 标记为 `Common` 属性


```
RunLoop.current.add(timer, forMode: .common)
```
等效于

```
CFRunLoopAddTimer(CFRunLoopGetMain(), timer, CFRunLoopMode.commonModes)
```

### 2. CFRunLoopMode
其本质也是一个结构体

```
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};

```

结合结构体成员变量我们可以知道：
一个 RunLoop 包含若干 mode， 一个 mode又包含若干 timer/ source / observe. 每次调用主函数的时候只能指定其中一个 mode, 也就是 current mode. 如果要切换只能退出 RunLoop 从新指定新的 mode 进入。这样做主要是分开不同的 timer/observe/sources, 让其相互之间不影响.

### 3. CFRunLoopSource

**source 0**
包含一个回调的函数指针，它不能主动的触发事件，需要手动去处理，先把它通过 CFRunLoopSourceSignal(`CFRunLoopSource`) 标记为待处理，然后使用 CFRunLoopWakeUp(`CFRunLoop`) 将其唤醒处理.

**source 1**
包含一个回调函数指针和 **mach_port**, 主要是用来通过内核和其它线程发消息，能够主动唤醒 RunLoop 线程。

### 4. CFRunLoopTimer
包含一个回调函数指针和一个时间长度，加入 RunLoop 系统会自动注册一个倒计时时间，当时间到了会唤醒 RunLoop 执行回调.

### 5. CFRunLoopObserver
包含一个回调函数指针，当 RunLoop 的状态发生变化的时候，观察者能够通过回调知道这个变化.

```
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

| 类别 | 作用描述 |
| --- | --- |
| kCFRunLoopEntry | 即将进入 |
| kCFRunLoopBeforeTimers | 即将处理 Timer |
| kCFRunLoopBeforeSources | 即将处理 Sources |
| kCFRunLoopBeforeWaiting | 即将进入 休眠 |
| kCFRunLoopAfterWaiting | 刚才休眠中 唤醒 |
| kCFRunLoopExit | 即将退出 |
| kCFRunLoopAllActivities | 所有状态的改版 |


```
        let observe = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, CFRunLoopActivity.allActivities.rawValue, true, 0) { observer, activity in
            switch activity {
            case .entry:
                print("进入 entry")
            case .beforeTimers:
                print("即将处理 Timer")
            case .beforeSources:
                print("即将处理 Source")
            case .beforeWaiting:
                print("即将进入睡眠")
            case .afterWaiting:
                print("刚才睡眠中唤醒")
            case .exit:
                print("退出")
            case .allActivities:
                print("所有状态的改版")
            default:
                print("其它: \(activity)")
            }
        }
        
        CFRunLoopAddObserver(CFRunLoopGetCurrent(), observe!, .defaultMode)
```

我们可以监视视 RunLoop 具体的某个状态来处理一些其它事情：`CFRunLoopActivity.allActivities.rawValue` 以达到我们的使用的其它目的；

比如: 我们添加 `CADisplayLink` 到 `RunLoop` 上 展示 **FPS**

```
let caDis = CADisplayLink(target: self, selector: #selector(isTime))
caDis.add(to: RunLoop.current, forMode: .common)
/// 处理 timestamp
```


## 3. 对外的 `API` 有哪些，其内部逻辑是怎样的？

**CFRunLoopAddCommonMode(`CFRunLoop`, `CFRunLoopMode`)**
把 RunLoop 标记为某个属性，或者为某个 RunLoop 创建对应的 Mode

**CFRunLoopRunInMode(`CFRunLoopMode`, `CFTimeInterval`, `Bool`)**
用于指定的 Mode 启动，设置循环时间，执行完是否退出


**CFRunLoopAddTimer(`CFRunLoop`, `CFRunLoopTimer`, `CFRunLoopMode`)**

**CFRunLoopAddSource(`CFRunLoop`, `CFRunLoopSource`, `CFRunLoopMode`)**

**CFRunLoopAddObserver(`CFRunLoop`, `CFRunLoopObserver`, `CFRunLoopMode`)**

**CFRunLoopRemoveTimer(`CFRunLoop`, `CFRunLoopTimer`, `CFRunLoopMode`)**

**CFRunLoopRemoveSource(`CFRunLoop`, `CFRunLoopSource`, `CFRunLoopMode`)**

**CFRunLoopRemoveObserver(`CFRunLoop`, `CFRunLoopObserver`, `CFRunLoopMode`)**


只能通过 mode name 来操作内部的 model, 当传入新的 mode name 但是 RunLoop 内部没有对应的 mode 时， RunLoop 会自动帮你创建对应的 `CFRunLoopModeRef`， 对于一个 RunLoop 其内部的 Mode 只能增加不能删除.

| 苹果提供的 mode 有类型有:  ||
| --- | --- |
| 1. kCFRunLoopDefaultMode (公开)  |系统默认 mdoe |
| 2. UITrackingRunLoopMode (公开) | ScrollVoew 滑动 View|
| 3. kCFRunLoopCommonModes (公开) |操作 Common Items，或标记一个 Mode 为 Common|
| 4. UIInitializationRunLoopMode |APP 启动后第一个 mode, 启动完后就不再使用|
| 5. GSEventReceiveRunLoopMode |接受系统事件内部的 mode|

### 其内部逻辑
RunLoop 就是一个 while 循环，当调用 RunLoop 时，线程会一直在这个循环里面，直到超时或者手动停止，函数才会返回。


## 4. 底层原理是什么？

RunLoop 的核心是基于 mach port， 调用 `mach_msg()` 函数


| 应用层 | Spotlight，Aqua，SpringBoard |
| --- | --- |
| 应用框架层 | Cocoa 系框架 |
| 核心框架层 | 各种核心框架、OpenGL 等 |
| Darwin | 操作系统核心，内核、驱动、Shell(均开源) |
 
### Darwin：
硬件层面上通过：Mach、BSD、IOKit 来组成 XUN 内核成分之一；

* XNU 内核的内环被称作 Mach，其作为一个微内核，仅提供了诸如处理器调度、IPC (进程间通信)等非常少量的基础服务。
* BSD 层可以看作围绕 Mach 层的一个外环，其提供了诸如进程管理、文件系统和网络等功能。
* IOKit 层是为设备驱动提供了一个面向对象(C++)的一个框架

在 Mach 中，所有的东西都是通过自己的对象实现的，进程、线程和虚拟内存都被称为”对象”。和其他架构不同， Mach 的对象间不能直接调用，只能通过消息传递的方式实现对象间的通信。”消息”是 Mach 中最基础的概念，消息在两个端口 (port) 之间传递，这就是 Mach 的 IPC (进程间通信) 的核心。

为了实现消息的发送和接收，mach_msg() 函数实际上是调用了函数mach_msg_trap()。当在用户态调用 mach_msg_trap() 时会触发陷阱机制，切换到内核态；内核态中内核实现的 mach_msg() 函数会完成实际的工作。

RunLoop 的核心就是 mach_msg()
RunLoop 调用这个函数去接收消息，如果没有别人发送 port 消息过来，内核会将线程置于等待状态。

## 5. iOS 系统的案例有哪些？
### 1. AutoreleasePool 自动释放池
APP在启动后会在 RunLoop 中注册两个 Observer

#### 1. Observer 监测 entry
回调函数是：
`_wrapRunLoopWithAutoreleasePoolHandler()`
回调内会调用：`_objc_autoreleasePoolPush()`
创建释放池，其优先级最高，保证创建释放池在其它回调之前.

order = -2147483647 (-2^31)

#### 2. Observer 监测：
* beforeWaiting
回调函数是：`_wrapRunLoopWithAutoreleasePoolHandler()`
回调会调用 `_objc_autoreleasePoolPop()` 释放旧的释放池 和 `_objc_autoreleasePoolPush()` 创建新的释放池
* exit
回调函数：`_wrapRunLoopWithAutoreleasePoolHandler()`
退出的时候调用 `_objc_autoreleasePoolPop()` 来释放自动释放池，优先级最低，保证其调用发生在所有的回调之后
order = -2147483647 (2^31)


### 2. 事件的响应
#### Source1
iOS 注册了一个 Source1 其包含一个回调函数指针和 mach port, 它的回调函数是：`__IOHIDEventSystemClientQueueCallback()`

当有：触摸、锁屏、摇晃 事件的产生的时候，首先会有  `IOKit.framework` 生成一个 **IOHIDEvent** 事件, 这个事件会由 `SpringBoard` 接收：

| 接收事件 |
| --- |
| 按键 （锁屏/静音） |
| 触摸 |
| 加速 |
| 传感器 |
| ... |

收到 Event 用会用 mach port 转发给需要的 App 进程, 随后 Source1 就会触发回调：`_UIApplicationHandleEventQueue()` 进行进一步的分发.

流程大致如下:

`_UIApplicationHandleEventQueue()` 会把 `IOHIDEvent` 处理并包装成 `UIEvent` 进行处理或分发其中包括识别 `UIGesture/处理屏幕旋转/发送给 UIWindow `等。

通常事件 `UIButton` 的点击、`touchesBegin/Move/End/Cancel` 事件都是在这个回调中完成的。

### 3. 手势识别
当 `_UIApplicationHandleEventQueue()` 识别并产生一个手势的事件的时候，其首先会调用 `Cancel` 将当前的 `touchesBegin/Move/End` 系列回调打断。随后系统将对应的 `UIGestureRecognizer` 标记为待处理。

通过 `lldb` 调试可以看到：
iOS 会注册一个 关于 `BeforeWaiting` 的 `Observer`， 它的回调函数是：`_UIGestureRecognizerUpdateObserver()`其内部会获取所有刚被标记为待处理的 `GestureRecognizer`，并执行 `GestureRecognizer` 的回调。

当有 `UIGestureRecognizer` 的变化(创建、销毁、状态改变)时，这个回调都会进行相应处理。

### 4. 界面更新
在操作UI 的时候, 比如修改了 UIView, CALayer, 或者调用了：`setNeedsLayout()`  `setNeedsDisplay()` 时系统会把这个 View 标记为待处理，并被提交到一个全局的容器中。

iOS 会注册一个：**Observe**  用来监听：监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件

回调去执行一个很长的函数：
`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()`。这个函数里会遍历所有待处理的 `UIView/CAlayer` 以执行实际的绘制和调整，并更新 UI 界面。
(在AutoLayout 补充)

### 5. 定时器
Timer 其实就是 `CFRunLoopTimer`, 只是 Timer 的计时颗粒度比较大，当 Timer 被注册到 RunLoop 中后会在 RunLoop 中注册相应的回调时间点. RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 `Tolerance` (宽容度)，标示了当时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。

再说一下: CADisplayLink 
是一个和屏幕刷新率一致的定时器, 如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去造成界面卡顿就可以计算出来

### 6. PerformSelecter

* `perform(#selector(action), with: nil, afterDelay: 3)`

实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

* `performSelector(onMainThread: #selector(action), with: nil, waitUntilDone: true)`

实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。


### 7. GCD
`DispatchQueue.main.async`

当调用 `DispatchQueue.main.async {}` 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 `__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()` 里执行这个 block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

### 8. 网络请求

| 层级 | 作用 |第三方|
| --- | --- | --- |
| CFSocket | 最底层的接口，只负责 socket 通信 |
| CFNetwork | 基于 CFSocket 等接口的上层封装，ASIHttpRequest 工作于这一层 |ASIHttpRequest |
| NSURLConnection |是基于 CFNetwork 的更高层的封装，提供面向对象的接口，AFNetworking 工作于这一层| AFNetworking |
| NSURLSession |是 iOS7 中新增的接口，表面上是和 NSURLConnection 并列的，但底层仍然用到了 NSURLConnection 的部分功能 |AFNetworking2, Alamofire |


**NSURLConnection**
通常使用 `NSURLConnection` 时，你会传入一个 `Delegate`，当调用了 [tast start] 后，这个 Delegate 就会不停收到事件回调。实际上，start 这个函数的内部会会获取 `current RunLoop`，然后在其中的 `DefaultMode` 添加了4个 `Source0` 。`CFMultiplexerSource` 是负责各种 Delegate 回调的，`CFHTTPCookieStorage` 是处理各种 Cookie 的。

进行网络请求的时候会有下面两个线程：

`com.apple.CFSocket.private`
其中 CFSocket 线程是处理底层 socket 连接的

`com.apple.NSURLConnectionLoader`
这个线程内部会使用 RunLoop 来接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate。

线程中的 `RunLoop` 通过一些基于 `mach port` 的 `Source` 接收来自底层 `CFSocket` 的通知。当收到通知后，其会在合适的时机向 `CFMultiplexerSource` 等 `Source0` 发送通知，同时唤醒 `Delegate` 线程的 `RunLoop` 来让其处理这些通知。`CFMultiplexerSource` 会在 `Delegate` 线程的 `RunLoop` 对 `Delegate` 执行实际的回调。

## 6. 第三方案例有哪些？
AFNetworking

AsyncDisplayKit

[内容学习处](https://blog.ibireme.com/2015/05/18/runloop/)


