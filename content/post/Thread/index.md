---
title: '多线程问题的补充'
subtitle: '多线程问题的补充' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-11-03T00:00:00Z"
lastmod: "2020-11-03T00:00:00Z"
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


# 多线程问题的补充
## 1. GCD
### 1.1 同步、异步， 串行、并发

死锁的原因：
队列引起的循环等待.

### 1.2 dispatch_barrier_async

### 1.3 dispatch_grup


## 2. NSOPeration/NSOPerationQueue
使用 NSOPeration 有哪些优势和特点？

①. 可以添加依赖

②. 任务执行状态的控制
对NSOperation 的状态判断，重写相应的实现。

③. 控制最大并发量
	如果是 1 就是串行队列 
	
### 2.1 任务执行状态的控制

#### 我们可以控制 NSOPeration 的哪些状态？

1. isReady 是否处于就绪状态
2. isExecuting 是否处于正在执行中状态
3. isFinished 是否已完成
4. isCancelled 是否已经取消

#### 状态的控制

如果只重写 main 方法， 底层控制任务变更执行完成状态，以及任务退出.

如果重写了 star 方法， 需要自行控制任务状态. 在合适的时机来修改 状态值.

#### 系统是怎样移除一个 isFinished=YES 的 NSOPeration的？
系统是通过 KVO 的方式来移除 NSOPerationQueue 的 NSOPeration 的. 来达到正常退出销毁 NSOPeration 对象.

## 3. NSThread
启动流程：

第一步: start()
第二步: 创建 pthread
第三步: main()
第四步: 在 main 函数内执行：[target performSelector: selector]
第五步: exit()

开始的时候是执行 start 方法, 然后系统创建一个 pthread, 接着会执行 main 函数，在 main 函数内 执行 [target performSelector: selector], 然后退出。 如果我们想要维护一个常驻线程，那么就需要在 select 方法内进行维护一个常驻的 RunLoop。 

## 4. iOS 系统中有哪些锁？

### 4.1 @synchronized
一般用来创建单利对象, 来保证在多线程的情况下创建是唯一的.

### 4.2 atomic
修饰属性的关键字. 
对被修饰的对象进行原子操作(不负责使用)

对赋值操作是线程安全的，对操作是不安全的. 对于走 set get 方法是线程安全的，否则是不安全的.

### 4.3 OSSpinLock
自旋锁, 
①. 是一个忙等的锁，不断循环等待。 

②. 系统在引用计数表，和弱引用表中使用过。

③. 针对轻量级内存访问。

### 4.4 NSLock
一般解决细粒度的线程同步问题，来保证线程的互斥来进入自己的临界区

```
-(void)methodA {
    _lock = [[NSLock alloc]init];
    
    [_lock lock];
    [self methodB];
    [_lock unlock];
}

-(void)methodB {
    [_lock lock];
    NSLog(@"lock");
    [_lock unlock];
}

```
这会是死锁，对同一把锁进行两次加锁，相当于第一次获取到了这把锁，接着再获取一次会因为重入的问题造成死锁.

这种情况就需要使用递归锁.

### 4.5 NSRecursiveLock
递归锁


```
-(void)methodA {
    _recursiveLock = [[NSRecursiveLock alloc]init];
    
    [_recursiveLock lock];
    [self methodB];
    [_recursiveLock unlock];
}

-(void)methodB {
    [_recursiveLock lock];
    NSLog(@"lock");
    [_recursiveLock unlock];
}

```

递归锁支持重入操作，所以对于重复加锁是没有问题的。

对于上锁和解锁是成对出现的.

### 4.6 dispatch_semaphore_t

信号量

调用信号量阻塞线程的执行是主动行为.

唤醒是被动行为，由释放信号唤醒被阻塞的线程.


```
    func semaphore() {
        ///  Dispatch Semaphore 是持有计数的信号，该计数是多线程编程中的计数类型信号。计数为0时等待，计数为1或者大于1而不等待
        /// 最好处理的是多线程处理其它事情，最终对某一个地址进行写入操作，对地址进行计数信号等等处理
        let semaphore = DispatchSemaphore(value: 1)
        let grup = DispatchGroup()
        let queue = DispatchQueue(label: "这是并行队列", qos: .default, attributes: .concurrent)
        for i in 0...9 {
            queue.async(group: grup) {
                /// 一直等待，或者等待一定的时间
                print("等待前来过")
                _ = semaphore.wait(timeout: .distantFuture)
                /// 执行操作
                self.num += 1
                print("this is: \(i): -- \(Thread.current) -- num: \(self.num)\n")
                semaphore.signal()
            }
        }
        grup.notify(queue: queue) {
            print("done: \(self.num)-- \(Thread.current)\n")
        }
    }
```


#### 4.6.1 DispatchSemaphore 内部实现

```
struct semphore {
    int value;
    List <thread>;
}
```

#### 4.6.2 执行 semaphore.wait(timeout: .distantFuture) 的时候
其内部会 
S.value = S.value - 1;
if S.value < 0 then Blcok(S.List);

当小于 0 就会阻塞，是一个主动行为.

#### 4.6.3 semaphore.signal()
S.value = S.value + 1;
if S.value < 0 then WakUp(S.List); 

唤醒是被动行为，由释放信号唤醒被阻塞的线程.



## 5. 总结

### 5.1 GCD 实现多读单写
使用 dispatch_barrier


```
queue.async(group: nil, qos: .default, flags: .barrier)
```

### 5.2 iOS 系统提供了几种线程，各种特点是怎么样的？

1、GCD
	
一般使用 GCD 来进行简单的线程同步，子线程的分派，多读单写，文件监听，文件读取
	
2、NSOPeration和 NSOPerationQueue
特点是方便控制线程的状态，能够添加依赖移除依赖


3、NSThread
使用常驻线程


### 5.3 NSOPeration 在 Finished 之后怎样从 Queue 中移除的？

KVO 的方式, queue 通过KVO 的方式监听 NSOPeration 相应状态的变化, 以此来达到移除的目的.

### 5.4 你都用过哪些锁，结合实际情况来谈一谈怎么使用的？

