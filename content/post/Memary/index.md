---
title: ' 关于内存管理的底层原理'
subtitle: '关于内存管理的底层原理' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-11-01T00:00:00Z"
lastmod: "2020-11-01T00:00:00Z"
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


# 关于内存管理的底层原理

## 1. 内存布局


| 内存区 | 高地址(0xc0000000) |
| --- | --- |
| 栈区(stack) |  |
|   |  |
| 堆区(heap) |  |
| .bss (未初始化的全局变量) (程序) |  |
| .data (已初始化的全局变量)(程序) |  |
| .txt (代码片段)(程序) |  |
| 保留区 | 低地址(0x08048000) |


### 1.1. stack 栈区
	
一般是我们方法调用的区域
	
### 1.2. heap 堆区

通过 allco 分配的对象
	
### 1.3 bss

未初始化的全局变量，以及未初始化的全局静态变量
	
### 1.4 data

已初始化的全局变量，全局静态变量	

### 1.5 text
	
程序代码片段

## 2. 内存管理方案
iOS 系统如何进行内存管理的？

iOS 是针对不同场景提供不同的内存管理方案；

### 2.1. TaggedPointer
对一些小对象，NSNumber 采用的是 TaggedPointer 内存管理方案

### 2.2. NONPOINTER_ISA
对于 64位架构下，可能32位或者 40位就能够表达出类的地址，那么剩余的比特位是用来存储内存管理相关的信息。这也是非指针型的 isa 来源.

arm64 的架构下：64位的比特位

#### 2.2.1： 0位(第一位)：叫 indexed 标志位
内容：

* 	0：代表这个 isa 是一个指针型的 isa 指针，代理当前类对象的地址.
* 	1：不光有类对象的地址，还有内存管理相关的数据. 是非指针型的 isa.

#### 2.2.2： 1位(第二位): 叫 has_assoc 标志位
	
代表是否有关联对象

* 0： 代表没有
* 1： 代表有

#### 2.2.3： 2位(第三位): 叫 has_cxx_dtor 标志位

代表是否有使用过 C++ 代码，或者 C++ 语言方面的内容, 也可以表示有些对象是通过 MRC 来管理的.

#### 2.2.4： 3 - 35位(第4 - 第36位): 叫 shiftcls 标志位
一共33位代表的是类对象的地址

#### 2.2.5：36 - 41位（第37位 - 42位） magic 字段

#### 2.2.6：42位 （第43位） 叫 weakly_referenced

标记这个对象是否有弱引用指针

#### 2.2.7：43位 （第44位） 叫 deallocating

标志当前是否进行 dealloc 操作

#### 2.2.8：44位 （第45位）叫 has_sidetable_rc
代表当前 isa 指针所存储的引用计数已经达到了上限，那么需要外挂一个 sidetable 表，这样一个数据结构。去存储相关的引用计数信息。

#### 2.2.9：45 - 63 (第 46位 到 64位) extra_rc 存储引用计数值
代表的是额外的引用计数，当我们引用计数在一个很小的范围之内时候，会存储带 isa 指针中，而不是有一个单独的引用计数表。

### 2.3. 散列表
是一个复杂的数据结构。

* 引用计数表

* 弱引用计数表

#### 2.3.1. Side Tables() 结构，其本质是一个哈希表
其内是若干个 Side Table 表。
在非嵌入式系统中有 64个。

一般通过对象指针可以找到所在的 SideTable 表，并找到其内的引用计数表和弱引用表。

SideTable 内存：
①. spinlock_t 自旋锁
②. RefcountMap 引用计数表
③. weak_table_t 弱引用表

##### 2.3.1.1 为什么不是一个 SideTable 而是有多个 SideTable 来组成一个 SideTables 这样的一个数据结构?
当所有的引用计数信息都存储在一张表的时候，在多线程的情况下对对象的引用计数进行操作的时候就会照成对资源的竞争极大的影响效率。

对此系统引入了

分离锁的计算方案：
对引用计数表进行拆分，同时设置分离锁。这样在多线程的情况下可以并发的访问引用计数相关的信息。

##### 2.3.1.2 如果实现快速分流？（如何通过对象指针快速找到对象所在的 SideTable 表）

SideTables() 本质是一张 hash 表. 里面存放 64个 SideTable 表，每个 SideTable 表存放 引用计数表和弱引用表。

对象指针为 key 经过 Hash 函数计算生成一个值：value，就是 SideTable 所在数组中的索引位置. 

Hash 查找：
当给定对象的内存地址，目标值是数组的下标索引.之间的关系：

地址：ptr
Hash 计算：f(ptr)
计算结果: value

f(ptr) = (uintptr_t)ptr % arry.count

在存储的时候直接按照 计算value 存储到相应的 index, 那么在查找的时候直接计算 index 即可找到.


## 3. 数据结构
SideTable:

### 3.1. SpinLock_t 自旋锁
自旋锁是忙等的锁，当前线程会不断的去询问当前锁是否被释放，如果被释放就去获取. 所以说是忙等的锁.

自旋锁适用于轻量访问.

其它锁：比如信号量
它是当锁被其它线程持有的时候，它会阻塞线程进入休眠状态，当锁被释放的时候再唤醒线程并进行加锁相关操作。

#### 3.1.1 你是否使用过自旋锁，自旋锁适用于那些场景，和其它的锁区别是什么？

系统的 SideTable 表内部就是使用的自旋锁

自旋锁是一个忙等的锁，当多线程使用自自旋锁进行加锁访问的时候，它会不断的询问当前锁是否被释放，而比如信号量当被其他线程持有的时候会阻塞线程并进入休眠状态，当锁被释放的时候再把线程唤醒并进行加锁相关的操作。 所以自旋锁适用于快进快出的资源加锁操作。不然就会因为忙等造成性能的下降。


### 3.2. RefcountMap 引用计数表

是一个hash 表，也称为字典。可以通过指针来到相应的引用计数。主要是通过哈希函数来对指针进行运算得到的index 值，系统在操作的时候根据index 值插入相应的位置，取的时候也是使用hash 函数进行计算出 index 值进行查找.


| 指针地址 | hash 函数运算得到index | 得到结果|
| --- | --- | --- |
| ptr | DisguisedPtr(objc_object) | size_t(unsign long) |

#### 3.2.1 引用计数表是通过什么来实现的？
是通过 hash 表来实现的。

#### 3.2.2 为什么通过hash 表来实现？
因为提高查找效率，每次的插入和获取是通过一个hash算法来计算位置的，这样就不用每次 for 循环遍历。

#### 3.2.3. size_t 的数据结构
引用计数在 64位系统下面用64位来表示：

##### 3.2.3.1 0位 (第一位)：weakly_referenced 代表当前是否有弱引用
0是没有
1是有

##### 3.2.3.2 1位 (第二位): deallocating 
代表当前是否正在进行 dealloc

##### 3.2.3.3 2-63 (第3到第64) RC: 代表的是引用计数值

在实际操作的时候需要向右进行偏移两位，因为前两位不是代表的引用计数值。通过偏移后才是真正的引用计数值。

### 3.3. weak_table_t 弱引用表
是一张 hash 表.

根据给出对象的指针通过 hash 函数计算出弱引用对象的的存储位置。 


| 对象指针（key）  | Hash 函数 -> | Value (weak_entry_t) |
| --- | --- | --- |

#### 3.4. weak_entry_t 
是一个结构体数组，其数组内存储的是实际的弱引用指针，比如：__weak obj. 其地址就是弱引用地址，存储在 weak_entry_t 里面.
	

## 4. ARC/MRC

### 4.1 MRC 
MRC是手动引用计数的方式来进行对象的内存管理. 

1. alloc: 分配内存空间
2. retain: 引用计数加 1
3. release: 引用计数减 1
4. retainCount: 引用计数值
5. autorelease: 如果当前对象进行 autorelease 操作的时候，会在 autoreleasepop 结束的时候进行 release 操作，引用计数减 1操作.
6. dealloc: 需要显示的调用 super dealloc 来废弃父类的相关成员变量。

### 4.2 ARC
自动引用计数管理内存.

在 ARC 下手动进行 retain, release, retainCount, autorelease 会编译报错.

ARC 是编译器自动为我们插入 retain, release 之外还需要 Runtime 的功能来协作.

1. ARC 是 LLVM 和 Runtime 共同协作的结果.
2. ARC 中禁止调用 retain/release/autorelease/retainCount/dealloc。但是在 ARC 中可以重写 dealloc, 但是不能调用 super dealloc。
3. ARC 中新增 weak、srong 属性关键字.

## 5. 引用计数
### 5.1. alloc
经过一系列的调用，最终调用了 C 函数的 calloc。
通过 alloc 操作之后并没有设置引用计数为 1，但是我们通过allco 之后查询 retainCount 发现引用计数加1了.
	
### 5.2. retain 
进行 retain 操作的时候先在 SideTables 表中找到 sidetable 结构体， 然后在sidetabel结构体中找到引用计数表，然后在引用计数表中找到对象的引用计数。通过地址进行两次hash 查找, 对于查找后的值进行偏移操作。
	
总结：进行两次hash 查找，对查到到的引用计数值进行加一操作。
	
### 5.3. release
进行 release 操作，先通过对象的地址进行 hash 算法计算出引用表所在的 sidetable 表， 然后再进行 hash 算法找到引用计数表中值，然后进行减一操作。
	
### 5.4. retainCount
	
①.同样通过对象的地址进行 hash 函数的计算并找到 sidetable 表. 
②. 定义一个局部变量为 size_t refcnt_result = 1
③. 在进行一次 hash 查找, 取出引用计数信息, 进行地址偏移操作。然后返回 refcnt_result += 引用计数。
	
这就说明了 alloc 后并不会引用计数加一，而通过 retainCount 查询却是1的问题. 对于查找结果加 1 进行了返回。
	
### 5.5. dealloc
	
第一步：_objc_rootDealloc()
第二步：rootDealloc()
第三步：判断是否可以释放？
	①. nonpointor_isa 是否是非指针型 isa
	②. weakly_referenced 是否有 weak 指针指向它
	③. has_assoc 是否有管理对象
	④. has_cxx_dtor 是否有 C++ 相关的引用，或者是使用 MRC 管理内存
	⑤. has_sidetable_rc 判断当前的引用计数表是否由 sideTable 表来维护的，也就是说当非指针型 isa 存储的引用计数信息无法满足的时候会使用 sidetable 表来处理引用计数相关的信息.
		 		
当都是否的时候会调用 C 函数的free() 函数进行直接释放。 否则会调用另外的函数进行后续的清理工作.
	
#### 5.5.1 后续：object_dispose() 实现

第一步: objc_destructInstance() 销毁实例的含义
第二步: C 函数销毁 free()

##### 5.5.1.1 objc_destructInstance() 实现

第一步: 首先判断是否有 C++ 相关的内容，has_CxxDtor? 
	有： object_CxxDestruct() 直接到第二步：
	没有：直接到第二步：
	
第二步: 判断当前对象是否有关联对象： hasAssociatedObjects?
	有: _objc_remove_assocations() 对象相关关联对象的移除. 下一步：
	没有：下一步：

第三步：clear_deallocating()

##### 5.5.1.2 clear_deallocating() 流程
第一步：sidetable_clearDeallocating()

第二步：weak_clear_no_lock() 将该对象的弱引用指针置为 nil

第三步：table_refcnts.erase() 从引用计数表中擦除该对象的引用计数

## 6. 弱引用

id __weak obj1 = obj;
objc_initWeak(&obj1, obj);

添加 weak 变量：
	第一步：objc_initWeak()
	第二步：storeWeak()
	第三步：weak_register_no_lock() 具体在这.
	
如果一个对象被声明成__weak, 经过编译器编译 会调用相应的 objc_initWeak() 函数 然后经过一系列的函数调用栈会执行 weak_register_no_lock() 函数进行一个弱引用添加，具体添加是通过hash 算法来进行位置查找的，如果查找位置已经有了当前对象的一个对应的弱引用数组，我们就把新的弱引用对象添加到这个数组中。如果没有的话会重新创建一个弱引用数组并把第0个位置存储我们最新的弱引用指针，后面都初始化为 nil 或者为 0；

### 6.1 当一个weak对象被废弃后是如何处理的？

当执行 dealloc() 之后，最终会执行 weak_clear_no_lock() 也就是内部会调用弱引用清除的函数，在函数的内部实现过程中会根据当前对象指针查找出所在的弱引用表，在弱引用表中把当前对象相对应的弱引用数组拿出来, 然后变量所有的弱引用指针进行置为nil操作.

## 7. 自动释放池 AutoRelease Pool

@autoreleasepool() 具体由编译器实现如下：

```
void *ctx = objc_autoreleasePoolPush(); /// ctx 无类型的指针
// 代码
objc_autoreleasePoolPop(ctx);
```

### 7.1 自动释放池的数据结构
什么是自动释放池？

* 以栈为结点通过双向链表的形式组合而成.
* 并且和线程是一一对应的.

### 7.2 AutoreleasePage C++ 的类

| AutoreleasePage 组成结构 |
| --- |
| id *next (栈当中的下一个可填充的位置)  |
| AutoreleasePoolPage *const parent; (双向链表的父指针) |
| AutoreleasePoolPage *child; (双向链表的孩子指针) |
| pthread_t const thread; (和线程一一对应，从这里提现出来的) |

结构

|   | 栈顶 (低地址) |
| --- | --- |
| ... |  |
|  | <- next (指向空位置) |
| id obj(2) |  |
| id obj(1) | 栈底 |
| AutoreleasePoolPage | 自身所占用的内存 (高地址) |

### 7.3 objc_autoreleasePoolPush 的内部实现

void *objc_autoreleasePoolPush(void) 内部会调用 C++ 的 voud * AutoreleasePoolPage::push(void) 的方法


如果执行 push 操作会把 next 的指针位置为 nil(称为哨兵对象), 然后 next 指向下一个可入栈的位置.

实际每次执行 push 操作都会插入哨兵对象.

执行添加操作： [obj autorelease]
第一步: 判断当前 next 指针是否指向栈顶
	没有指向： 把当前对象添加到page 里面去
	指向了栈顶：增加一个栈结点到链表上，然后对这个栈执行添加操作.

执行 autoreleasePoolPush 会返回一个值，那个值就是哨兵对象的位置.

### 7.4 objc_autoreleasePoolPop 的内部实现
void *objc_autoreleasePoolPop(void *ctxt); 内部会调用
void * AutoreleasePoolPage::pop(void *ctxt)

一次pop 相当于一次批量的 pop 操作.

执行pop过程:

1. 根据传入的哨兵对象找到对应的位置
	哨兵对象：每次执行 push 操作都会插入哨兵位置，并返回哨兵位置.
		pop操作是回到上次push 返回的哨兵对象的位置.
	
2. 根据上次 push 操作之后添加的对象依次发送 release 消息.
3. 回退 next 指针到正确的位置


### 7.5 AutorealeasePool 的实现原理是怎么样的？

以栈为结点，通过双向链表的形式组合而成的一个数据结构.

1. 要清楚自动释放池的数据结构
2. 要知道 push 操作的原理，及返回哨兵值的逻辑。以及添加 autorelease 对象.
3. 要知道 pop 操作的原理，及释放操作的步骤，next 指针指向上一次 push 的哨兵位置


### 7.6 AutoreleasePool 为何可以嵌套使用？

多层嵌套就是多次插入哨兵对象, 在底层的实现原理上就是每次 push 操作就是在 autoreleasepoolpage 上插入哨兵对象，多次嵌套其实就是多次插入。所以是可以多层嵌套使用的.



### 7.7 总结：

#### 7.7.1 viewDidload() 中创建的对象在什么时候释放？

在每一次执行 runloop 的循环过程中，在将要结束的时候会对前一次进行 autoreleasepoolpush的操作进行 AutoreleasePoolPage::pop() 操作。同时会进行新的 autoreleasePoolPage::push() 操作.

#### 7.7.2 autoreleasePool 的使用场景

在 for 循环中 alloc 图片数据等内存消耗较大的场景中进行手动插入 autoreleasePool. 在特定的循环机制下进行一个手动的释放.


## 8. 循环引用

三种循环引用

1. 自循环引用
2. 相互循环引用
3. 多循环引用


### 8.1 自循环引用

id __strong obj

给obj 赋值原对象就是强引用；


### 8.2 相互循环引用

    id __strong obj1;
    id __strong obj2;
    
    obj1 = obj2;
    obj2 = obj1;

### 8.3 多循环引用
    id __strong obj1;
    id __strong obj2;
    id __strong obj3;
    id __strong obj4;
    id __strong obj5;
    
    
    obj1 = obj2;
    obj2 = obj3;
    obj3 = obj4;
    obj4 = obj5;
    obj5 = obj1;
    
    
###    8.4 循环引用的问题：
1. 代理
2. block
3. NSTimer
4. 大环引用


#### 8.4.1 如何破除循环引用？
1. 避免产生循环引用，比如代理一个strong 一个 weak
2. 在合适的时机手动断环

具体解决案例：

1. __weak

	 
```
    id __strong obj1;
    id __weak obj2;
    
    obj2 = obj1;
    obj1 = obj2;
```
	
2. __block: 一般用在 block 方面
两种情况：
	①. MRC：
	__block 修饰对象不会增加其引用计数，避免了循环引用
	
	②. ARC：
	__block 修饰的对象会被强引用，无法避免循环引用，需要手动解环.
	
	
3. __unsafe_unretained
	①. 修饰对象不会增加引用计数，避免了循环引用
	②. 如果修饰对象在某一时机被释放，会产生悬垂指针.
	
	不建议使用.
	
### 	8.5 循环引用的事例
你在开发过程中是否遇到循环引用，你是如何解决循环引用带来的问题的

①. block 事例

#### 8.5.1. NSTimer 事例

当前有个页面，页面上有广告栏，广告栏1秒播放一个动画

一般是广告栏作为一个对象，被 VC 强持有。 因为一秒中使用一次就会使用定时器。

在广告对象的中添加了 NSTimer, 添加 NSTimer 之后， NSTimer 会有一个回调，那么它对回调 target 是一个强引用。

由于广告对象添加了 NSTimer 是对它有一个强引用，即使是 weak 形式，也会是强引用，因为此时的 NSTimer 已经被 RunLoop 强持有了。


解决方案：
使用创建中间对象，中间对象持有弱引用分别是： NSTimer 和 原对象, 在 NSTimer 的回调对象是在中间对象中实现的, 在中间对象的执行方法对 target 进行判断，如果当前对象存在则相应相应的方法，如果不存在则无效 NSTimer.


