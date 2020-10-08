---
title: ' 关于启动优化需要知道的一些事'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-10-08T00:00:00Z"
lastmod: "2020-10-08T00:00:00Z"
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

# 关于启动优化需要知道的一些事

## 1. 时间构成
* **T1** 点击 APP 到执行 `main` 函数.

* **T2** 从 main 函数开始到 `appDelegate` 的 `didFinishLaunchingWithOptions`、

* **T3** 执行完 `didFinishLaunchingWithOptions` 后App还需要做一些初始化工作，然后经历定位、首页请求、首页渲染等过程后，用户才能真正看到数据内容并开始使用，我们认为这个时候冷启动才算完成。这个过程定义为 **T3** 。

**冷启动时间: T = T1 + T2 + T3**
启动优化在这三个阶段都可以施展拳脚.

### 1.1. 影响因素：
**T1 阶段**

* 执行了大量的 load() 方法
* 加载了无用的类及方法

**T2 阶段**

* 执行了大量的启动项任务
* 同步 I/0 操作
* 一些隐晦的耗时操作

**T2 阶段**

* 定位，请求，渲染

### 1.2. 发展历程
一般在 APP 早起开发阶段冷启动不会有明显的问题, 冷启动问题也不是在某个版本出现的，是长期一个量变到质变的过程。

而是随着版本迭代，App功能越来越复杂，启动任务越来越多，冷启动时间也一点点延长。最后注意到并想要优化它的时候，这个问题已经变得很棘手了。

## 2. 优化思路及方向


| 问题思考方方向 |   |
| --- | --- |
| 1. 解决存量问题 | 优化当前性能瓶颈点，优化启动流程，缩短冷启动时间。 |
| 2. 管控增量问题 | 冷启动流程规范化，通过代码范式和文档指导后续冷启动过程代码的维护，控制时间增量 |
| 3. 完善监控策略 | 完善冷启动性能指标监控，收集更详细的数据，及时发现性能问题 |

### 2.1. T1 的时间过程有哪些？
T1 也就是 main() 函数之前

main()之前操作系统所做的工作就是把可执行文件（**Mach-O**格式）加载到内存空间，然后加载动态链接库 **dyld**，再执行一系列动态链接操作和初始化操作的过程（加载、绑定、及初始化方法。

**如何优化：**

* staticlib

* 二进制重排

查看时间：
```Edit Scheme -> Run Dubug -> Arguments -> Environment Variables
``` 添加 
```DYLD_PRINT_STATISTICS
``` 值为 1.

DYLD_PRINT_STATISTICS



```
Total pre-main time: 1.1 seconds (100.0%)
         dylib loading time: 729.70 milliseconds (63.6%)
        rebase/binding time:  65.69 milliseconds (5.7%)
            ObjC setup time: 155.19 milliseconds (13.5%)
           initializer time: 196.28 milliseconds (17.1%)
           slowest intializers :
             libSystem.B.dylib :   6.89 milliseconds (0.6%)
    libMainThreadChecker.dylib :  49.65 milliseconds (4.3%)
                   MenuItemKit :  87.14 milliseconds (7.5%)
```

**具体在 main 函数之前做的是：**

真正的加载过程从 `exec()` 函数开始，`exec()` 是一个系统调用。操作系统首先为进程分配一段内存空间，然后把可执行文件 Mach-O格式加载到内存，接着加载 Dyld(动态链接器), 再接着由动态链接器加载动态库，

#### 2.1.1. exec()
真正的加载过程从exec()函数开始，exec()是一个系统调用。操作系统首先为进程分配一段内存空间。


#### 2.1.2. 加载 Mach-O 格式的可执行文件
| 组成成分 |  |
| --- | --- |
| Header  | Header中包含的是可执行文件的CPU架构，Load Commands的数量和占用空间 |
| Load Commands | Load Commands中包含的是Segment的Header与内存分布，以及依赖动态库的版本和Path等 |
| Segment Data | Segment Data就是Segment汇编代码的实现，每段Segment的内存占用大小都是分页页数的整数倍 |

#### 2.1.3. 加载 dyld(动态链接器)
#### 2.1.4. dyld 加载动态库 dylib

dyld读取完 Mach-O的Header和Load Commands后，就会找到可执行文件的依赖动态库。接着dyld会将所依赖的动态库加载到内存中。这是一个递归的过程，依赖的动态库可能还会依赖别的动态库，所以dyld会递归每个动态库，直至所有的依赖库都被加载完毕。

加载后的动态库会被缓存到dyld shared cache中，提高读取效率。

#### 2.1.5. Rebase & Bind
- Rebase在Image内部调整指针的指向。把动态库加载到指定地址，由于地址空间布局是随机的，需要在原来的地址根据随机的偏移量做一下修正 （可以理解为地址翻译）
- Bind是把指针正确地指向Image外部的内容。这些指向外部的指针被符号(symbol)名称绑定，dyld需要去符号表里查找，找到symbol对应的实现

#### 2.1.6. Objc setup
- 注册Objc类 (class registration)
- 把category的定义插入方法列表 (category registration)
- 保证每一个selector唯一 (selector uniquing)

因为Objective C的动态特性，所以在Main函数执行之前，需要把类信息注册到一个全局Table中。同时，Category的方法也会被注册到对应类中，Category中的同名方法实现，会根据编译顺序，被最后一个编译的Category实现所覆盖。同时还会做Selector的唯一性检测。


#### 2.1.7. Initializers
- Objc的+load()函数
- C++的构造函数属性函数
- 非基本类型的C++静态全局变量的创建(通常是类或结构体)

这里区分下+load方法与+Initialize方法，前者是在类加载时调用的，后者是在类第一次收到message之前调用的。    


