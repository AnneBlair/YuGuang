---
title: ' UI 视图知识回顾'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-10-22T00:00:00Z"
lastmod: "2020-10-22T00:00:00Z"
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

# UI 视图知识回顾
## 1. TableView
### 1.1. 重用机制
* 当前队列
* 重用队列（重用池）

### 1.2. 数据展示Load
①. 如何解决TableView 在多线程的情况下对数据的修改和访问问题？

* 1. 并发访问，数据拷贝的解决方案

主线程的数据 Copy 一份给子线程，当主线程进行删除操作的时候把删除的行为记录下来，当子线程数据组装完毕后访问 **删除的行为记录**，进行删除操作，之后再在主线程刷新数据，这样数据就可以准确的表达在 UI 上.

直接操作数据会造成多线程数据竞争

带来的问题是：copy 会增加内存

* 2. 串行队列的解决方案

主线程 A， GCD串行队列B，异步数据请求C
异步数据请求后执行 B的一个 **Block** 进行数据排版，主线程执行删除操作在串行队列B中执行,B需要执行完数据排版后进行删除操作的Block，然后在主线程中刷新UI，这个时候主线程的删除展示需要得到 C的请求是否完成以及数据排版是否完成.

问题：网络延迟会造成删除的UI展示有延迟效果.


### 1.3. 滑动优化
* CPU
* GPU 

## 2. 事件传递&视图响应
### 2.1. UIView 与 CALayer 的关系与区别
UIView 里面属性有 Layer 和 backgroundColor
其中 Layer 对应的是 CALayer, backgroundColor 是对 CALayer 同名属性的包装， UIView 的显示是由 CALayer 的 contents 来决定的. contents 对应的是 backing store(它实际是bit map 的位图), 最终显示在视图上的控件其实都是位图.

**区别：**

* UIView 负责为 CALayer 提供内容，以及负责处理触摸事件，参与响应链. 其是 UIResponder 的子类。

* CALayer 负责显示内容 contens

#### 2.1.1. 为什么 UIView 负责为 CALayer 提供内容，以及负责处理触摸事件，参与响应链 而 CALayer 只负责显示内容 contens
主要是系统模式的设计，①. 讲究单一执行原则 ②. 事件关注点分离职责分工. ③. 耦合性.

### 2.2
* 1. hitTest withEvent 返回 UIView 

	是哪个视图响应就返回哪个视图
	
* 2. pointInside withEvent

	判断点击的位置是否在某个视图内，在的话会返回 YES
	
#### 	2.2.1. 事件分发
**APP 外：**
点击屏幕 -> IO.KIT 产生一个IOEvent -> Storeboard -> 相关进程(APP)

**APP内**
Application -> UIWindow -> pointInside withEvent 判断是否在某个视图内 -> hitTest withEvent 返回响应的视图

**hitTest withEvent** 
是一个逆序遍历过程，最先添加到当前视图的View 最后遍历掉.

在进行事件分发的时候首先会进行判断视图是否有效
①. hidden 是否显示
②. userInteraction 是否可以交互
③. alpha 是否大于 0.01

事件响应
当前 View -> superView -> Controller -> UIWindow

* touchesBegin
* touchesMoved
* touchesEnded

## 3. 图像显示原理
* CPU 生成位图
* GPU 对位图进行渲染，纹理合成

### 3.1. CPU 工作
分为四步骤
1. Layout UI布局，文本大小的计算
2. Display 绘制， drawRect
3. Prepare 准备阶段：如果有图片，则图片的编解码在这一步
4. Commit 提交位图

### 3.2. GPU 的渲染管线 (其实也是 OpenGL 的渲染管线)

GPU 做以下5步处理
1. 顶点着色： 对位图进行处理
2. 图元装配：
3. 光栅化
4. 片段着色
5. 片段处理

处理完后会把像素点提交到 FrameBuffer(帧缓冲存储区)， 接着由视频控制器在 Vsync 信号到来之前去帧缓存存储区去取相应的内容展示到屏幕上.

## 4. UI 卡顿、掉帧
页面滑动的流畅度是 60FPS, 也就是说1秒钟会有60帧画面，那么每一帧画面就是 1/60 秒， 16.7ms 会产生一帧画面。在 16.7 毫秒之内 CPU（文本布局，UI计算，视图绘制，图片解码 -> 位图） + GPU（位图图层的合成，管理的渲染） 相互协同参数这一帧的内容; 在下一帧 VSync 信号到来之前进行显示. 

如果 CPU 在做这些事情的时候耗费时间较长，那么再加上 GPU 的处理事件超出了16.7 ms，就会出现掉帧，卡顿的现象.

总结：在规定的时间16.7 毫秒之内也就是下一个 Vsync 信号到来之前，CPU 与 GPU 没有共同完成下一帧画面的合成就会造成卡顿。

### 4.1. 滑动优化方案
基于 tableView 以及 scrollView 有哪些滑动优化方案？你是如何做的？

#### 4.1.1 减轻 CPU 处理的时长：

* CPU 的处理优化, 对象的创建, 调整, 销毁放到子线程去做节省CPU 处理时间.
* 预排版（UI布局的计算，文本计算）全放到子线程中去做, 主线程有更多的实际响应用户的交互
* 预渲染（文本的异步绘制，图片的编解码）

#### 4.1.2 GPU 的优化
* 纹理的渲染， 当触发了离屏渲染，产生了layer 的圆角， maskToBounds 的设置，包括阴影蒙层都会触发GPU 层面的离屏渲染。这些都会增加 GPU 做纹理渲染的工作量。尽量避免离屏渲染。同时依托 CPU 的异步绘制功能减轻 GPU 的压力.

* 视图的混合： 当视图层级比较复杂，CPG需要做每一个视图的合成，需要大量的计算像素点。 当减轻视图层级的复杂性，也可以减轻 GPU 的压力. 也可以通过 CPU 的异步绘制来达到提交的位图层级本身就比较少，这样也可以减轻 GPU 的绘制压力。

## 5. UI绘制原理、异步绘制
当执行UIView 的 setNeedsDisplay 并没有立刻产生当前视图的绘制工作，而是在之后的某一时机才进行当前视图的真正绘制。

执行 UIView 的 setNeedsDisplay方法接下来只是执行 Layer 的同名属性 setNeedsDisplay 方法。 相当于给 Layer 打上脏标记。然后会在当前 RunLoop 将要结束的时候调用 CALayer 的 display 方法，然后进入当前视图的真正绘制工作.

其 CALayer 内部 delegate 首先会判断是否响应 displayLayer 的方法，如果不响应会进入系统的绘制流程。 如果响应就会提供异步绘制的入口。

### 5.1 系统的绘制流程

在CALayer 内部会创建 **backing store** (CGContextRef) 
位于栈顶的 Backing Store或者说是 Context 或者说是上下文。


接下来会判断 layer 是否有代理

* 如果没有代理会触发 CALayer 的 drawInContext。
* 如果有，会执行代理的 drawLayer: inContext  做当前视图的绘制工作，这一步骤发生在系统的内部; 然后会在一个合适的时机执行回调：UIView 的 drawRect, 这就是给我们提供接口能够在系统的绘制之上做一些其他的绘制操作。

不管是那种方式都会由CALayer 上传 backing store (位图)给 GPU做进一步处理. 结束了系统的默认绘制流程。

### 5.2 异步绘制
如何实现异步绘制？
如果 CALayer 的 delegate 响应 displayLayer, 那么就可以进行异步绘制.

* 在异步绘制过程中由代理来响应负责生产的位图(bitMap )
* 设置 bitMap 作为 Layer 的contents 属性。

流程：

1. 执行 UIView 的 setNeedsDisplay
2. 接着会在RunLoop 的下一次循环中执行 UIView 的CALayer 的 Display
3. 接着判断是否响应 CALayer 内部的 delegate 方法
4. 如果响应接着在异步并发队列： 创建 
	CGBitMapContextCreat() // 创建上下文
	CoreGraphic 的API      /// 执行绘制操作
	CGBitMapContextCreatImage  /// 生产位图
	
5. 接着回到主队列把 位图设置给 CALayer 的 setContents

这样就完成了一个 UI 控件的异步绘制过程.
	

## 6. 离屏渲染
什么是离屏渲染？6.1 6.2


### 6.1 什么是在屏渲染（On-Screen-Rendering）：
意味当前屏幕渲染， 指的是GPU的渲染操作是在当前屏幕显示的缓冲区中进行.

### 6.2 什么是离屏渲染 (off-screen-rendering)
GPU 在当前的屏幕缓冲区以外开辟一个新的缓冲区进行渲染操作

何时触发离屏渲染？
1.设置圆角，设置圆角的同时需要设置 maskToBound 为 true, 才会触发离屏渲染
2. 设置图层蒙版
3. 设置阴影
4. 光栅化的设置

### 6.3. 如何避免离屏渲染？
为什么要避免？
离屏渲染触发了 GPU 也就是 OpenGL 渲染管道的额外开销.
在触发离屏渲染的时候会额外增加 GPU 的工作量，而 GPU 增加的工作量有可能会导致 CPU + GPU 的处理时间会超过 16.7ms, 会导致卡顿掉帧的现象，所以我们要避免离屏渲染.



## 7. 问题总结：
1. 系统UI事件传递机制是怎样的？
2. 使 UITableView 滚动更流畅的方案有哪些？
	* 	CPU 
		①. 在子线程进行对象的创建，修改，销毁
		②. 进行预排版，图片的解码，
		③. 采用异步绘制方案
	* 	GPU
		①. 避免离屏渲染
		②. 减少图像的层级
3. 什么是离屏渲染？
	
	离屏渲染的概念起源于 GPU， 就是在当前的缓冲区以外开辟一个新的缓冲区进行渲染操作。这就是离屏渲染.
	
4. UIView 和 CALayer 的关系咋么样？

	UIView 主要是为 CALayer 提供UI内容，包括布局信息等，同时参与事件的传递. 是UIResponse 的子类.
	CALyer 负责通过 contents 显示内容
	
	用到了6大设计原则的，单一原则
