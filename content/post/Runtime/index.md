---
title: 'RunTime 基础知识'
subtitle: 'RunTime 基础知识' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-10-26T00:00:00Z"
lastmod: "2020-10-26T00:00:00Z"
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


# RunTime 基础知识

## 1. 数据结构
### 1.1. objc_object
objc_object 是结构体
我们实际使用的对象都是 id 类型的，id 类型的对象对应到 runtime 当中实际就是 objc_object

##### 1.1.1 objc_object 主要包含：

1. isa_t: 共用体
2. 关于 isa 操作相关的一些方法：通过 objc_object 结构体来获取它的isa 指针所指向的类对象，包括通过类对象获取他的isa 指向的元类对象,一些遍历的方法
3. 弱引用相关的方法： 标记一个对象获取它曾经是否有过弱引用指针
4. 关联对象一些相关的方法：一个对象我们为它设置了关联属性相关方法也提现这里。
5. 内存管理的相关方法实现：
	
	MRC 下的 retain/ release
	MRC/ARC 下的@AutoRelease 自动释放池相关

### 1.2. objc_class
Class = objc_class

objc_class 是继承至 objc_object

class 这个类是否是一个对象呢？
是的

#### 1.2.1 objc_class 主要包含：

1. Class superClass: 指向的类型也是一个 class， 如果是一个类对象那么指向的是它父类对象，
2. cache_t cache: 表达了方法缓存的一个结构
3. class_data_bits_t bits: 关于一个类的变量、属性、方法 都在bits 的成员结构当中.

###1.3. isa指针
isa_t: C++ 中的一个共用体

目前的机器上是 64位的一个数字

##### 1.3.1 两种类型：

①. 指针类型 isa
	isa 64位的值代表 Class 地址
②. 非指针类型的 isa
	isa 的部分(33或者44)位代表 Class 的地址，这样做的目的是在寻址的过程44位就可以完成，多出来的部分存储其它相关内容，达到节省内存的目的.
	
isa 指针的含义？
有指针型的isa, 和非指针型的isa.

##### 1.3.2 isa 指向

①. 关于对象，其指向类对象
	实例(id 类型)isa指针指向的是 Class(类对象)
②. 关于类对象，指向元类对象.
	Class 的isa 指针指向的是 MetaClass 元类对象
	
### 1.4. cache_t

1.快速查找方法执行函数的一个结构.

调用一个方法，如果有缓存我们就不用去方法调用列表中去遍历，查找方法的逐一实现，调高我们方法的调用速度，或者说是消息传递的速度。

2.可增量扩展的哈希表结构
	
* 	增量扩展：当我们结构存储量增大的过程中，它也会增大的扩充内存空间,来支持更多的缓存。
* 	哈希表：哈希表的使用是提高查找效率

3.计算机局部性原理的最佳应用。
	当我们执行某几个方法比较频繁的时候，会把这几个方法放到缓存里面去，下次执行的命中率会更高。

#### 1.4.1 cache_t 数据结构类似数组：

| bucket_t | bucket_t | bucket_t | bucket_t | ...  |
| --- | --- | --- | --- | --- |

bucket_t 是结构体， 其组成成分是两个：
	①. key: selector  
	②. IMP: 无类型的函数指针
	
### 1.5. class_data_bits_t
 
*  class_data_bits_t 主要是对 class_rw_t 类的封装
*  class_rw_t 代表了类相关的读写信息、对 class_ro_t 的封装
*  class_ro_t 代表了类相关的只读信息

#### 1.5.1 class_rw_t 内容是：

一般是分类的方法相关信息

1. 	class_ro_t
2. 	protocols： 协议
3. 	properties：属性
4. 	methods：方法

我们对分类添加的方法、属性、协议都在 2，3，4中.
2，3，4 都是：list_arry_tt 一个二维数组

#### 1.5.2 class_ro_t

一般是宿主类的方法信息

1. name: 类的类名可以通过 NSClassFromString 反射出来类
2. ivars: 声明定义的一些类的成员变量
3. properties： 类的属性
4. protocol： 类遵循的一些协议
5. methodLists: 我们添加的一些方法

ivars、protocol、properties、methodLists 都是一维数组.
 

methodLists 对应的是:
 
| method_t | method_t | method_t | method_t | ... |
| --- | --- | --- | --- | --- |


###1.6. method_t

method_t 数据结构是对函数四要素的一个封装；

函数的四要素

1. 名称： SEL: name;
2. 返回值: const chart *types;
3. 参数:	const chart *types;
4. 函数体： IMP imp;

#### 1.6.1 Type Encodings

const chart *types

| 返回值 | 参数1 | 参数2 | ... | 参数n |
| --- | --- | --- | --- | --- |


-------

## 2. 对象，类对象，元类对象

### 2.1 类对象和原类对象分别是什么？有什么区别和联系？

* 类对象： 存储实例方法列表等信息.
* 元类对象： 存储类方法列表等信息.

实例对象通过 isa 指针能够找到类对象，类对象通过 isa 指针能找到元类对象。
类对象和元类对象都是 objc_class 数据结构的, 因为 objc_class 是继承 objc_object 所以他们都有 isa 指针。 

元类对象的isa 指针指向根元类对象，根元类对象的 isa 指针指向自己。根元类的 superClass 指向根类对象。

当我们调用的类方法在元类查找不到的时候就会去根类方法里面查找同名的实例方法。

### 2.2. 当我们调用类方法而原类方法中没有实现的时候而实例方法有实现，会不会发送崩溃？会不会产生实际的调用？ 

因为根元类对象的 super class 指向了根类，所以当元类方法中没有查到的时候就会去查找根类对象的同名实例方法。所以不会崩溃。

## 3. 消息传递机制

### 3.1 void objc_msgSend(void /* id self, SEL op */)

[self class] == objc_msgSend(self, @selector(class))

### 3.2 void objc_mesSendSuper(void /* struct objc_super * super, SEL op,  */)


```
struct objc_super {
	_unsafe_unretained id receiver;
}
```

receiver 其实是当前对象; receiver == self

[super class] == objc_msgSend(super, @selector(class))


### 3.3 消息传递流程
调用一个方法的时候

1.缓存：先查找缓存是否命中，缓存查找到就结束。
	缓存其实是局部变量最快命中的最佳应用。 缓存是一个 hash 表， 通过方法 name 去查找缓存里面的 bucket_t, bucket_t 里面包含 IMP 函数指针。	
2. 查找当前类对象中的方法列表：缓存没有命中就会去当前类对象中的方法列表中查找, 找到就结束。
	①.当前类对象中方法已排好序就用二分查找
	②.当前类对象中的方法没有排好序，就实际遍历查找
	
3.父级类逐级查找是否命中
	查找当前类对象的方法列表中没有，就会去当前类的父类中去查找，先查找缓存是否命中，再查找方法列表中是否命中(同样已排好序的用二分查找，未排序的逐一遍历查找)，找到就结束，未找到就向当前的 superClass 继续查找，直到super Class 为 nil.
	
4. 如果父类查找中为 nil, 没有找到，那么就会进入消息转发流程.

### 3.4 缓存查找具体是怎样的流程和步骤？

给定 SEL, 目标值是 bucket_t 中的 IMP

cache_key_t ---f(key)--->	 bucket_t

其实就是 hash 查找，

f(key) = key & mask  = 索引位置

### 3.5 当前类中查找

* 已排序好的方法列表，会采用二分查找的的算法查找方法对应的执行函数。
* 对于没有排序好的列表，采用一般遍历查找方法对于的执行函数。

### 3.6 父类逐级查找的过程

第一步： curClass = curClass -> superclass
第二步：判断 curClass 是否为 nil, 为 nil 结束进入消息转发机制
第三步：查找 curClass 的 缓存是否命中？ 没有命中就会去当前方法列表中去查找，没有找到就会去当前父类中去查找，也就是执行 第二步；

### 3.7 消息转发流程

1. 第一步.  + (BOOL)resolveInstanceMethod:(SEL)sel
	返回YES 的话就结束，否则进入下一步：
	
2. 第二步.  - (id)forwardingTargetForSelector:(SEL)aSelector
	 返回一个转发目标，由转发目标来处理。如果没有返回转发目标就是返回 nil， 就会进入第三步：
3. 第三步.  - (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
	返回一个类，返回一个方法签名的话，系统会继续调用, 如果返回 nil 则消息无法处理- crash。 
	
```
    if (aSelector == @selector(showMessage)) {
        NSLog(@"methodSignatureForSelector");
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    } else {
        return [super methodSignatureForSelector:aSelector];
    }
```

	v@:  v 表示：void  @表示：第一个参数类型时id, 即self, :表示：代理第二个人参数SEL
	
4. 第四步. - (void)forwardInvocation:(NSInvocation *)anInvocation
	如果还无法处理就会- 进入消息无法处理- crash

## 4. Method-Swizzling

交换前：
select 1 -> IMP1
select 2 -> IMP2

交换后：

select 1 -> IMP2
select 2 -> IMP1

### 4.1. 方法的动态替换：

method_exchangeImplementations(Method  _Nonnull m1, Method  _Nonnull m2)

一般使用替换可以进行代码注入的操作：
比如：

我们有个方法：

```
- (void)showOther {
    NSLog(@"showOther");
}
```

我们需要在这个方法执行内一个额外的事情比如：打印：doOtherWork，我们可以使用 runtime 对两个方法进行替换：

```
- (void)doOtherWork {
    NSLog(@"doOtherWork");
    [self doOtherWork];
}

```

这样执行： showOther 的时候的结果就是：

```
doOtherWork
showOther
```

### 4.2. 动态添加方法：

class_addMethod(Class  _Nullable __unsafe_unretained cls, SEL  _Nonnull name, IMP  _Nonnull imp, const char * _Nullable types)

1. 参数1： Class  _Nullable __unsafe_unretained cls
是为哪个类动态添加.

参数2：SEL  _Nonnull name 添加的选择器名称是什么？@selector(方法)

参数3：IMP  _Nonnull imp 方法的具体实现，就是把它添加上去.

参数4：方法的返回类型，参数个数。

参数2，3，4 是method_t 的成员变量.


#### 4.2.1 面试：你是否使用过 performSelector:(SEL)

一个类在编译时没有这个方法，在运行时才有这个方法。那么在使用的时候就需要用到 performSelector。
那么在运行是添加这个方法就需要用到 Runtime 的动态添加,


#### 4.2.2 动态方法解析

@dynamic 编译器关键字
当我们声明的属性在实现的时候标记为 @dynamic 的时候，相当于 get set 方法是在运行时添加不是在编译时实现。

* 动态运行的语言将函数的决议推迟到运行时。

* 编译时语言是在编译期确定了函数的决议.

## 5. Runtime 总结

### 5.1. [obj foo] 与 objc_msgSend() 函数之间有什么关系？

向 obj 对象发送 foo这条消息，实际在编译后就处理成了 objc_msgSend(obj, foo). 接下来就是 Runtime 的消息传递过程


### 5.2. RunTime 是如何通过 Selector 找到对应的 IMP 地址？

消息传递机制的描述：

首先查找实例对象的 isa, 找到实例的类对象, 接着就查找类对象的缓存，缓存命中没有查找到就查找当前类的方法列表，方法列表没有查找到就会查找当前类的父类，父类的缓存，方法列表，还没查找到就会查找父类的父类直到父类为 nil。 对于缓存的查找是：hash查找， 方法列表是：二分或者遍历。

### 5.3 能否向编译后的类增加实例变量？
不可以，对于编译后的类其实例内存布局都已完成，根据编译后的类的数据结构 class_or_t 的定义可知是不可以的。

但是对于动态添加的类当中是可以动态添加实例变量。主要在动态添加类的注册之前，去完成实例变量的添加是可以实现的。

