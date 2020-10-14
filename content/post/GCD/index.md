---
title: ' GCD 的常规用法与底层原理'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-10-14T00:00:00Z"
lastmod: "2020-10-14T00:00:00Z"
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

# GCD 的常规用法与底层原理
以一种非常简洁的记述方法，实现了极为复杂繁琐的多线程编程.

只需要定义想执行的任务并追加到适当的 Dispatch Queue 中，GCD 就能生成必要的线程并执行任务。由于线程管理是作为系统的一部分来实现的，因此可统一管理，也可以执行任务，直接通过 XNU 内核交互，这样就比其它线程更快.

系统常规简单的多线程技术案例：

```
performSelector(onMainThread: <#T##Selector#>, with: <#T##Any?#>, waitUntilDone: <#T##Bool#>)
        
performSelector(inBackground: <#T##Selector#>, with: <#T##Any?#>)
```

存在的问题：

* 数据竞争
* 死锁
* 线程过多消耗大量的内存

## 1. 串行队列

```
        let queue = DispatchQueue(label: "this is serial")
        queue.async {
            print("this is serial 1")
        }
        
        queue.async {
            print("this is serial 2")
        }
        
        queue.async {
            print("this is setial 3")
        }
        
        /// 1 执行完执行 2， 2 执行完执行 3， 共用一个线程
```
使用一个线程对数据访问安全。

一个串行队列会开辟一个线程，同时多个 DispatchQueue(label: "this is serial") 会有多个线程. 因此不能无限次生成串行队列.
## 2. 并行队列

```
        let queue = DispatchQueue(label: "this is concurrent", qos: .default, attributes: .concurrent, autoreleaseFrequency: .workItem, target: .none)
        queue.async {
            print("this is concurrent 1")
            /// 新线程 1
        }
        
        queue.async {
            print("this is concurrent 2")
            /// 新线程 2
        }
        
        queue.async {
            print("this is concurrent 3")
            /// 新线程 3
        }
```
多个线程访问数据造成数据竞争,

对于并行线程不管生成多少， 由 XNU 内核只使用有效管理的线程

## 3. 系统提供的 GCD 创建方式
* 第一种：DispatchQueue(label:) 上面的例子
* 第二种：DispatchQueue.global  系统直接提供 DispatchQueue.global 的例子

### 3.1 dispatch_set_target_queue

```
dispatch_queue_t queue = dispatch_queue_create("this is serial", NULL);
dispatch_queue_t globl = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
dispatch_set_target_queue(queue, globl);
```

把串行队列的优先级改成和另一个线程的优先级一样，使用 `dispatch_set_target_queue`， 在swift中使用：`serialQueue.setTarget(queue: global)`

### 3.2 asyncAfter(deadline: time)

```
let time = DispatchTime.now() + 5
DispatchQueue.main.asyncAfter(deadline: time) {
	print("5秒后执行")
}
```

注意：asyncAfter 函数并不是在指定的时间后执行处理，而是在指定的时间追加处理到 DispatchQueue

因为 Main Dispatch Queue 在主线的 RunLoop 中执行，所以在每隔 1 / 60 秒执行的 RunLoop 中, block 最快在 5 秒后执行，最慢是在 5  + 1 / 60 秒执行，如果在 Main Dispatch Queue 中有大量处理其本身有延迟时，时间会更长.


#### 3.2.1 DispatchTime
DispatchTime 是相对时间

#### 3.2.2 DispatchWallTime
DispatchWallTime 是计算绝对时间

```
    NSTimeInterval interval = [date timeIntervalSince1970];
    double second, subsecond;
    struct timespec time;
    subsecond = modf(interval, &second);
    time.tv_nsec = subsecond * NSEC_PER_SEC;
    time.tv_sec = second;
    dispatch_time_t milestone = dispatch_walltime(&time, 0);
```

### 3.3 DispatchGroup
并发线程全部处理完后得到结束事件一般使用 Group，其获得结束事件的方式有两种：
第一种： notify
第二种： wait

#### 3.3.1 notify
**推荐使用**
使用 notify 检测结束事件，并在指定线程中执行结束代码；
且结束事件的执行是闭包的形式;

```
        let group = DispatchGroup()
        let queue = DispatchQueue.global(qos: .default)
        queue.async(group: group) {
            print("第一个线程: \(Thread.current)")
        }
        queue.async(group: group) {
            print("第二个线程: \(Thread.current)")
        }
        
        group.notify(queue: DispatchQueue.main) {
            print("俩线程执行完了， 这是主线程: \(Thread.current)")
        }
```

#### 3.3.2 group.wait
其事件的结束可以设置超时时间，或者到某个时间还未执行完就超时；
可使用相对时间，或者绝对时间；

```
        let result = group.wait(timeout: DispatchTime.now() + 3)
        switch result {
        case .success:
            print("全部执行完毕")
        case .timedOut:
            print("超时还未执行完毕： \(result)")
        }
```

使用 wait 是在主线程的 RunLoop 的每次循环中检查执行是否结束，从而不消耗多余的等待时间，


### 3.4 dispatch_barrier_async

barrier的使用一般是在并行线程中使用，一般处理并行线程对资源的处理操作。
当 barrier 的 async 使用的时候，当前 DispatchQueue 的其它并行线程都会暂停或者说是挂起. 当 barrier 的 async 执行完后其它的并行线程继续执行。

通常用来处理对数据库或者文件的访问操作；
当并行线程 1，2，3用来执行对某个资源只读操作，4 是写操作，5，6，7是只读操作。那么直接执行的时候会由于多线程对资源的竞争造成数据异常。那么把读与写在同一时间分开处理那么就会解决资源竞争的问题. 这个时候就可以使用 barrier。用 barrier 处理对资源的写操作。当线程执行写操作的时候其它线程就会暂停挂起，等待写操作执行完毕后继续进行。这样就解决了数据竞争的问题。

使用并行线程和 barrier 可以实现高效率的数据库访问和文件访问.

**Swift使用方式:**

```
        let queue = DispatchQueue(label: "this is concurrent", qos: .default, attributes: .concurrent, autoreleaseFrequency: .inherit, target: .none)
        queue.async {
            print("this is thread 0: \(Thread.current)")
        }
        
        queue.async {
            print("this is thread 1: \(Thread.current)")
        }
        
        queue.async {
            print("this is thread 2: \(Thread.current)")
        }
        
        queue.async {
            for _ in 0...5 {
                sleep(1)
                print("this is thread 3: \(Thread.current)")
            }
        }
        
        queue.async(group: nil, qos: .default, flags: .barrier) {
            print("我执行的时候其它都得暂停，在 done 之前不会有输出: \(Thread.current)")
            for _ in 0...5 {
                sleep(1)
            }
            print("我执行完了， done")
        }
        
        queue.async {
            for _ in 0...5 {
                sleep(1)
                print("this is thread 5: \(Thread.current)")
            }
        }
        
        queue.async {
            for _ in 0...5 {
                sleep(1)
                print("this is thread 6: \(Thread.current)")
            }
        }
        
        queue.async {
            print("this is thread 7: \(Thread.current)")
        }
        
        queue.async {
            print("this is thread 8: \(Thread.current)")
        }
```

**OC 中使用：**

```
    dispatch_queue_t queue = dispatch_queue_create("this is queue concurrent", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(queue, ^{
        for (int i = 0; i < 5; i++) {
            sleep(1);
            NSLog(@"this is thread 0");
        }
    });
    
    dispatch_async(queue, ^{
        for (int i = 0; i < 5; i++) {
            sleep(1);
            NSLog(@"this is thread 1");
        }
    });
    
    dispatch_barrier_async(queue, ^{
        NSLog(@"this is barrier begin");
        for (int i = 0; i < 5; i++) {
            sleep(1);
        }
        NSLog(@"this is barrier, done");
    });
    
    dispatch_async(queue, ^{
        for (int i = 0; i < 5; i++) {
            sleep(1);
            NSLog(@"this is thread 2");
        }
    });
    
    dispatch_async(queue, ^{
        NSLog(@"this is thread 3");
    });
```

### 3.5 sync 同步执行
async 是异步执行，sync 是同步执行.

**async** 是不用等到处理完会继续执行；
**sync** 是等到处理完才会继续向下执行；


```
        let queue = DispatchQueue.global()
        queue.sync {
            sleep(3)
            print("1")
        }
        print("2")
        // 打印：1
        // 打印：2
```

dispatch_sync 是简易版的 dispatch_group_wait.

使用 sync 容易造成死锁；

#### 3.5.1 死锁案例1: 主线程 sync

```
        let mainQueue = DispatchQueue.main
        mainQueue.sync {
            print("打印")
        }
        print("永远来不了")
```

#### 3.5.2 案例 2：主线程先 async 在 sync

```
        let mainQueue = DispatchQueue.main
        mainQueue.async {
            mainQueue.sync {
                print("死锁")
            }
        }
```

#### 3.5.3 案例 3：串行队列的死锁

```
        let seriaQueue = DispatchQueue(label: "this is seial")
        seriaQueue.async {
            seriaQueue.sync {
                print("死锁")
            }
        }
``` 

### 3.6 dispatch_apply 
指定次数将指定的 Block 追加到 queue 中；
但是它跟 sync 一样会等等处理完才会继续执行，所以一般是：

```
		   let queue = DispatchQueue.global()
        queue.async {
            DispatchQueue.concurrentPerform(iterations: 5) { num in
                print("num:\(num) thread: \(Thread.current)")
            }
            print("done")
            DispatchQueue.main.async {
                print("执行完毕");
            }
        }
```

打印：

```
num:1 thread: <NSThread: 0x6000019c4b80>{number = 5, name = (null)}
num:4 thread: <NSThread: 0x6000019c0280>{number = 6, name = (null)}
num:2 thread: <NSThread: 0x600001994e40>{number = 3, name = (null)}
num:0 thread: <NSThread: 0x600001998540>{number = 4, name = (null)}
num:3 thread: <NSThread: 0x600001991f00>{number = 7, name = (null)}
done
执行完毕
```

#### 3.6.1 OC 中的用法:

```
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_apply(10, queue, ^(size_t num) {
        NSLog(@"num: %z, thread: %@", num, [NSThread currentThread]);
    });
```
打印：

```
2020-10-14 11:39:38.710840+0800 GCD-Next[3644:171705] num: 3, thread: <NSThread: 0x600002964e80>{number = 5, name = (null)}
2020-10-14 11:39:38.710848+0800 GCD-Next[3644:171704] num: 6, thread: <NSThread: 0x600002969180>{number = 8, name = (null)}
2020-10-14 11:39:38.710849+0800 GCD-Next[3644:171708] num: 5, thread: <NSThread: 0x60000294c080>{number = 3, name = (null)}
2020-10-14 11:39:38.710852+0800 GCD-Next[3644:171709] num: 1, thread: <NSThread: 0x6000029481c0>{number = 4, name = (null)}
2020-10-14 11:39:38.710853+0800 GCD-Next[3644:171715] num: 8, thread: <NSThread: 0x60000294c100>{number = 10, name = (null)}
2020-10-14 11:39:38.710853+0800 GCD-Next[3644:171703] num: 0, thread: <NSThread: 0x600002925000>{number = 6, name = (null)}
2020-10-14 11:39:38.710854+0800 GCD-Next[3644:171588] num: 7, thread: <NSThread: 0x600002928900>{number = 1, name = main}
2020-10-14 11:39:38.710854+0800 GCD-Next[3644:171713] num: 4, thread: <NSThread: 0x600002955440>{number = 9, name = (null)}
2020-10-14 11:39:38.710857+0800 GCD-Next[3644:171706] num: 2, thread: <NSThread: 0x600002925040>{number = 7, name = (null)}
2020-10-14 11:39:38.710871+0800 GCD-Next[3644:171716] num: 9, thread: <NSThread: 0x600002962bc0>{number = 11, name = (null)}

```

#### 3.6.2 Swift 中的用法
与OC 的不同地方是 Swift 会自动管理线程;

```
DispatchQueue.concurrentPerform(iterations: 5) { num in
	print("num:\(num) thread: \(Thread.current)")
}
```

打印:

```
num:0 thread: <NSThread: 0x600001ff4900>{number = 1, name = main}
num:3 thread: <NSThread: 0x600001fb3680>{number = 6, name = (null)}
num:2 thread: <NSThread: 0x600001fba040>{number = 5, name = (null)}
num:4 thread: <NSThread: 0x600001fad580>{number = 4, name = (null)}
num:1 thread: <NSThread: 0x600001fa81c0>{number = 3, name = (null)}
```


### 3.7 dispatch_suspend 挂起 dispatch_resume 恢复
挂起后已追加到 Queue 中已执行的处理没有影响，已追加到 queue 中尚未执行的处理会停止。恢复后继续执行。

挂起只是在当前动作执行的时候如果有已追加到 queue 尚未执行的会停止，其它情况不会停止； 


### 3.8 DispatchSemaphore
使用信号量处理多线程中对资源的竞争操作。


```
        /// 多线程并行队列对数据竞争的处理一般可以设计
        /// 1. 串行队列
        /// 2. 使用 dispatch barrier
        /// 3. semaphore
        /// 计数类的信号量 计数为0 的时候等待，计数为1或者大于1时减去1不等待
        
        let semaphore = DispatchSemaphore(value: 1)
        let queue = DispatchQueue.global()
        
        for _ in 0...10 {
            queue.async {
                /// 一直等待，直到大于或者等于 1, 
                semaphore.wait(timeout: .distantFuture)  // 会进行减一操作
    //            安全的处理不用担心竞争的问题
                ///  排它控制结束，计数加1
                semaphore.signal()
            }
        }
```

### 3.9 Dispatch I/O
**提高文件读取速度**

1. 先创建 Dispatch I/O
2. 设定文件一次的读取大小
3. dispatch_io_read 函数使用 Global 进行并列读取
4. 读取后将每次的data 传递给指定的回调 block. 
5. 回调用的 Block 分析传递过来的 Dispatch Data 进行结合处理.


## GCD 的底层实现

多线程的底层是 XNU 内核方面的操作。GCD 是直接能够接触 XNU 内核。 其性能方面是内核级实现的。所以效率极高.


| 组件名称及层面 | 提供技术 |
| --- | --- |
| libdispatch | Dispatch Queue |
| Libc (pthreads) | pthread_workqueue |
| XNU 内核 | workqueue |


我们接触的 GCD API 都在 libdispatch 库中的 C语言函数。

Dispatch Queue 通过结构体和链表，被实现为 FIFO 队列。 FIFO 队列管理 dispatch_async 函数所追加的 Block.

Block 并不是直接加入 FIFO 队列，而是先加入 Dispatch Continuation 的 dispatch_continuation_t 的结构体中。然后再加入 FIFO 队列。

这个 Dispatch_continuation 主要是用来记录 Block 所属的 Dispatch Group 和一些其他信息。 

### 执行过程：

1. 执行 Block 时，libdispatch 从 Global Dispatch Queue 自身的 FIFO 队列中取出 Dispatch Continuation. 
2. 取出信息后调用 pthread_workqueue 组件中的 pthread_workqueue_additem_n 函数。将自身、符合其优先级的 workqueue 信息以及为执行 Dispatch Continuation 的回调函数传递给参数。
3. pthread_workqueue_additem_n 函数会调用 work_kernreturn 系统调用。通知 workqueue 增加应当执行的的类目。
4. 这个时候 XNU 内核会基于系统状态去判断是否要生成线程。 如果是 Overcommit 优先级的 Global Dispathc Queue, workqueue 会始终生成线程。
5. workqueue 的线程执行 pthread_workqueue 函数，该函数调用 libdispatch 的回调函数。在回调函数中执行加入 Dispatch Continuation 的 Block.
6. Block 执行结束后，进行通知 Dispatch Group 结束，释放 Dispatch Continuation 等处理，开始执行下一个加入 Dispatch Queue 中的Block.


主线程在 RunLoop 中执行 Block.

### Glob Dispath Queue 有 8 种优先级

| Global Dispatch Queue (high Priority) |  |
| --- | --- |
| Global Dispatch Queue (Default Priority) |  |
| Global Dispatch Queue (Low Priority) |  |
| Global Dispatch Queue (Background Priority) |  |
| Global Dispatch Queue (high Overcommit Priority) | 串行队列 |
| Global Dispatch Queue (Default Overcommit Priority) | 串行队列 |
| Global Dispatch Queue (Low Overcommit Priority) | 串行队列 |
| Global Dispatch Queue (Background Overcommit Priority) | 串行队列 |

串行队列不管系统如何，都会强制生成线程


### XNU 内核持有 4中 workqueue

* WORKQUEUE_HIGH_PRIOQUEUE
* WORKQUEUE_DEFAULT_PRIOQUEUE
* WORKQUEUE_LOW_PRIOQUEUE
* WORKQUEUE_BG_PRIOQUEUE





