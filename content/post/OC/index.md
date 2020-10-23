---
title: 'OC 知识'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-10-24T00:00:00Z"
lastmod: "2020-10-24T00:00:00Z"
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

# OC知识

## 1. 分类
### 1.1. 你用分类都做了哪些事情？

1. 用分类来声明私有方法
2. 用分类来分解体积庞大的类文件 (机器学习相关就是这样做的)
3. 把 Framework 的私有方法公开

### 1.2. 分类的特点：

* 运行时进行决议
当分类完成编译后并没有把相关方法附加到宿主类上，而是在运行时通过 Runtime 把分类的信息附加到宿主上，真实的添加到宿主类上。

这是分类的最大特点，也是和扩展的最大区别.

* 可以为系统类添加分类

### 1.3. 分类中可以添加哪些内容？

1. 添加实例方法
2. 类方法
3. 协议
4. 给分类添加属性
	只是给属性添加了get set 方法，并没有添加实例变量

可以添加实例变量，只不过是通过关联对象的方式进行的。

多个分类有多个同名方法，最终哪一个会生效, 最后编译的文件这个分类会生效。
为什么？ Runtime 源码中对分类的添加是倒序遍历来的。

**”覆盖“**： 当分类中有和宿主类中同名的方法，最新实现的是分类的方法，这是分类添加源码中的描述, 原因是分类的方法最终在数组中是在最前面，而宿主类的方法靠后，当调用某个方法的时候找到就返回了，所以导致同名在数组后面的方法不会执行。间接的呈现出了覆盖的效果。

* 分类添加的方法可以**”覆盖“**原类的方法。
* 同名分类方法谁能生效取决于编译顺序，最后被编译的同名方法最先生效。
* 名字相同的分类会引起编译报错

## 2. 关联对象
能为分类添加成员变量。

我们不能在分类的声明或者定义实现的时候为分类添加成员变量。但是我们可以通过关联对象的方式为分类添加实例变量。从而达到为分类的添加成员变量的效果。


```
id objc_getAssociatedObject(id object, const void *key);

void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);

void objc_removeAssociatedObjects(id object);
```

### 2.1. 通过关联技术实现的关联对象被放到了哪？

关联对象的本质
关联对象是由 AssociationsManager 管理并在 AssociationsHashMap 存储.

它是一个全局的容器，不管为那个类提供了实例成员变量都是放到一个全局的字典中.


| AssociationsHashMap |  |
| --- | --- |
| DISGUISE(Obj)被关联对象的当前指针值 | ObjctAssociationsMap |
↑

| ObjctAssociationsMap |  |
| --- | --- |
| @selecor(text) | ObjcAssociation |
↑

| ObjcAssociation |
| --- |
| OBJC_ASSOCIATION_COPY_NONATOMIC |

### 2.2. 怎么清除被关联对象的关联值呢？
通过Objc_setAssociatedObject 传入 nil 值即可完成擦除.


## 3. 扩展和代理

我们一般用扩展来做什么？

* 声明私有属性
* 声明私有方法
* 声明私有成员变量

### 3.1. 扩展与分类的区别？
1. 扩展是在编译的时候决议，分类是在运行时决议
2. 以声明的显示存在，多数情况下寄生于宿主类的.m中
	分类有声明有实现，而扩展只能声明，其实现是在宿主类中.
3. 不能为系统类添加扩展
	可以为系统类添加分类，不能为系统类添加扩展
	
### 3.2. 代理
1. 是一种软件设计模式
2. iOS 中以 protocol形式出现
3. 传递方式是一对一

#### 3.2.1 协议：
协议中一般可以定义什么？
方法和属性

#### 3.2.2 代理的工作流程是什么样的？
协议、委托方(调用协议的方法，其实是调用代理方的实现)、代理方(具体的实现，方法可能有返回值-返回给委托方)

#### 3.2.3 协议里面的方法和属性代理方一定要实现吗？
不一定，看方法及属性的标记情况，如果是 option(可选的)就不用必须实现

#### 3.2.4 代理方和委托方关系应该怎么建立？会有什么问题？
代理方会强引用委托方，委托方通过 weak 属性弱引用代理方，从而规避循环引用.

## 4. NSNotification 通知原理
发送者 - 通知中心 - 广播给观察者1、2、3、4、5...
### 4.1. 代理和通知之间的区别在哪里？

* 设计模式区别：通知是使用观察者模式进行跨层传递消息的机制. 代理是通过代理模式进行消息传递机制.
* 传递方式：代理是一对一，通知是一对多.

### 4.2. 怎样实现通知机制
在通知中心会维护一个字典，其中key 是通知的 NSNotificationName 监听的名称, 其 value 是一个数组，数组的内容是观察者 Observe, 观察者调用的方法就是其回调方法。

## 5. KVO、KVC 系统的实现机制与原理
### 5.1. 什么是 KVO？

1. KVO 是 key - value - observing 的缩写
2. KVO 是 OC 对观察者模式的又一实现
3. Apple 使用了 isa 混写(isa-swizzling) 来实现 KVO

#### 5.1.1 如何实现 isa 混写？
在观察某一个类的属性的时候，runtime 会在运行时创建这个类的子类（NSNotifying_类名），从而把 isa 指针指向这个子类，子类会重写观察属性的 set 方法，当这个属性值发生变化的时候会调用set 方法，这个时候子类会调用父类然后通知观察者，这样就可以通知观察者内容的变化.

**调用父set 前后是这样的：**
willChangeValueForKey
调用父set
didChangeValueForKey

#### 5.1.2. 通过 KVC 设置 value, KVO 能否直接生效？
能生效

##### 5.1.3 为什么？

#### 5.1.4. 通过成员变量赋值, KVO 能否直接生效
不能直接生效

可以通过下面生效.
willChangeValueForKey
_变量 = 赋值
didChangeValueForKey

didChangeValueForKey会触发 KVO 的回调.


* 使用 setter 方法改变值 KVO 才会生效
* 使用 setvalue forKey 改变值才会生效
* 成员变量直接修改需要手段添加（willChangeValueForKey 和 didChangeValueForKey）KVO 才会生效.

### 5.2 什么是KVC 

1. key-value-coding 的缩写，苹果提供的键值编码技术.
2. 通过 valueForKey: NSString*: key 获取和 key 同名或者相似名称的变量值
3. setValue forKey 通过key 设置和 key 同名或者相似名称的变量值.

#### 5.2.1. 我们通过键值编码技术是否会破坏面向对象的编程方法？或者违背面向对象的编程思想？
会破坏，因为KVC 是通过 key 来修改同名属性信息，那么对应类中的私有方法也是可以修改的，那么这样就违背了面向对象的设计.

#### 5.2.2. 系统实现流程？
##### 获取的流程
首先系统会先判断是否有get(Accessor Method） 方法，如果有会直接调用.

不存在：
1. 先断实例变量是否存在
	通过： accessInstanceVariablesDirectly -> Bool 判断

* 存在 - 结束
* 	不存在会调用 valueForUndefinedKey：抛出一个未定义key（NSUndefinedKeyException） 的异常
	
问题：
Accessor Method 访问器判断具体规则是咋么样的？

1. 当类中实现了一些 get 方法，get 方法的名称和 key 相识也会认为存在，返回 true
2. 当类中存在一些实例变量，名称和 key 相识也会返回 true

##### setValue forKey 的调用流程
先判断实例变量是否存在set 方法 (set method is exit?) 存在直接赋值结束

不存在：

先判断是否有实例变量：
通过： accessInstanceVariablesDirectly -> Bool 判断是否存在，如果存在直接赋值结束
	不存在：会调用当前实例变量的 setValue: forUndefinedKey 的方法然后会抛出异常(NSUndefinedKeyException) 结束setValue forKey 的流程.

## 6. Copy, 等相关修饰的使用
### 6.1. 读写权限：
readOnly
readwrite (默认)

### 6.2. 原子性
atomic (默认)
	赋值和获取是线程安全的，是对成员属性直接的获取或者赋值，不是操作和访问.
	比如：对数组进行赋值和获取是线程安全的，但是对数组进行操作，比如添加和移除对象是不安全的.
noatomic

### 6.3 引用计数
retain/ strong
retain(MRC 中使用)
strong(ARC 中使用)
assign(既可以修饰基本数据类型，也可以修饰对象数据类型)
unsafe_unretained (MRC使用频繁，ARC 基本没有)
weak/copy

#### 6.3.1 assign 和 weak 的区别有哪些？

assign：3个特点

* 修饰基本类型，如 int, Bool等
* 修饰对象时不改变其引用计数
* 会产生悬垂指正

weak：2个特点

* 不改变被修饰对象的引用计数
* 所修饰的对象在被释放或者废弃之后会自动设置为 nil

区别：
weak 可以修饰对象，而assign既可以修饰对象也可以修饰基本类型,

#### 6.3.2 copy

浅拷贝：
1. 浅拷贝是对内存地址的复制
2. 浅拷贝会增加被拷贝对象的引用计数

深拷贝：
1. 让目标对象指针和源对象指针指向两个不同的内存空间，但是是内容相同 。
2. 不增加引用计数，增加了内存分屏.

区别：
看是否开辟新的内存空间，是否影响引用计数.


| 源对象类型 | 拷贝方式 | 目标对象类型 | 拷贝类型(深、浅) |
| --- | --- | --- | --- |
| mutable 对象 | copy | 不可变 | 深拷贝 |
| mutable 对象 | mutableCopy | 可变 | 深拷贝 |
| immutable 对象 | copy | 不可变 | 浅拷贝 |
| immutable 对象 | mutableCopy | 可变 | 深拷贝 |


* 可变对象的 copy 和 mutableCopy 都是深拷贝
* 不可变对象的 copy 是浅拷贝，mutableCopy 是深拷贝
* copy 返回的都是不可变对象


## 总结
MRC 下如何重写 retain 修饰下的变量的set方法？

在 set 方法中判断是否是原来对象，如果不是进行 release 操作，然后进行 retain 操作再赋值给变量。

为什么判断是否相等？
如果不判断，如果是当前对象，直接 release的话，会把当前传递过来的变量直接释放了


请简述分类的实现原理？
1. 运行时决议的，不同分类中含有同名方法谁最终生效取决于谁最后完成编译，最后编译的同名方法最终生效。分类中的方法和宿主类的方法一致的时候，分类的方法最终会”覆盖“掉宿主内的方法。这里的覆盖是指消息查找优先查找数组中靠前的元素。找到了同名方法就最先调用。

2.KVO 的实现原理是什么样的？
KVO 是系统关于观察者模式的一种实现
KVO 是系统通过ISA 混写的实现.（参考前面）

3.能否为分类添加实例变量？
通过关联对象可以为分类添加对象.（参考前面）

