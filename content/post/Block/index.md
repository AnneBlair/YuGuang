---
title: ' Block 必须了解的姿势'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-11-02T00:00:00Z"
lastmod: "2020-11-02T00:00:00Z"
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


# Block 必须了解的姿势

## 1. 什么是 Block?
block 是将函数及其上下文封装起来的对象.

* 包含 isa 指针，isa 指针是对象的标志
* FuncPtr 函数指针， block 用函数指针来指向对于的实现

## 2. 什么是 Block 调用？
block 调用就是函数的调用.


## 3. 截获变量

### 3.1. 关于不同类型的变量类型有不同截获方式.

| 变量 | block 截获方式 |
| --- | --- |
| 1\. 基本数据类型的局部变量 (局部变量) | block 是截获其值 |
| 2\. 对象类型的局部变量 (局部变量) | 连同所有权修饰符一起截获 |
| 3\. 局部静态变量 (局部变量) | 以指针形式截获局部静态变量 |
| 4\. 全局变量 | 不截获 |
| 5\. 静态全局变量 | 不截获 |


### 3.2. __block 修饰符

一般情况下对截获的变量进行赋值操作需添加 __block 修饰符.

赋值 ≠ 使用


#### 3.2.1 需要使用 __block

* 对于局部变量进行赋值操作需要添加 __block，(基本数据类型局部变量，对象类型局部变量)

* 对于局部静态变量，全局变量，全局静态变量进行赋值操作不需要添加 __block

#### 3.2.2 局部变量添加 __block
__block修饰的变量变成对象， 变成一个结构体，有isa 指针
 
__block int multiplier = 6; 实际是把 multiplier 变成了一个对象，当对其再次赋值的时候是：

(multiplier.__forwording->multiplier) = 4;


栈上的 __block 变量有一个 __forwording 指针，其指向自己 __block. 


__forwording 指针用来干什么的？

## 4. block 内存管理
impl.isa = &NSConcreteStackBlock;



| Block | 类别 | Copy 结构 |
| --- | --- | --- |
| StackBlock | 栈上 | 堆上 |
| GlobalBlock | .data 上 | 什么也不做 |
| MallocBlock | 堆上 | 增加引用计数 |


blcok 种类
### 4.1. Global
在内存当中的 .data 区域(已初始化数据)，已初始化的全局变量、全局静态变量区域
### 4.2. Stack
在栈上.

当栈上的 block 在赋值给成员变量的时候，成员变量没有进行关键字 copy 的操作，那么在栈销毁的时候 blcok 就没有了，此时对成员变量进行访问，就会造成异常情况.

#### 4.2.1 栈上的 Block 销毁

| 栈上 |
| --- |
| __block 变量 |
| Block |

__block 变量作用域结束，或者说栈上的函数退出之后，__block 变量以及 Block 都会随之销毁。

#### 4.2.2 栈上 Block Copy 操作

| 栈上 |  | 堆上 |
| --- | --- | --- |
| __block 变量 | --->Copy--- | __block 变量 |
| Block |  | Block |

栈上的变量作用域结束之后就执行了销毁，而此时堆上的还在。

当我们对栈上的 block 执行 copy 操作之后，在 MRC 情况下是否会引起内存泄露？

会造成泄露，因为没有对栈上的进行释放，如何alloc 之后没有进行 release 操作是一样的.



| 栈上 |  | 堆上 |
| --- | --- | --- |
| Blcok |  | Block |
| __block 变量 | ——>Copy--- | __block 变量 |
| __forwarding |  | __forwarding |

当栈上的 blcok 没有执行 copy 操作的时候，其 __forwarding 指针是指向自己。

当栈上的 blcok 执行了 copy 操作的时候，其 栈上的 __forwarding 指针指向的是堆上的 __block 变量，而堆上的 __forwarding 指针指向的是自己（堆上的 __block 变量）


#### 4.2.3 __forwarding 总结
对于在任何内存位置，都可以顺利的访问同一个 __block 变量.

对于栈上的 __block 并且没有进行copy 操作, 通过 __forwarding 访问的是栈上的 __block 变量, 对于执行了 copy 操作之后，通过 __forwarding 访问的是堆上的 __block 变量.

### 4.3. Malloc
在堆上


## 5. Block 循环引用的问题
6.1




## 6. 总结
### 6.1 什么是 blcok?
block 是对函数以及函数执行上下文进行封装的对象

### 6.2 为什么 blcok 会产生循环引用？

1. 如果当前 block 对当前对象的成员变量有一个截获的话，这个blcok 就会对当前对象的变量有一个强引用，而当前 blcok 又被当前对象有一个强引用，这样就造成了自循环引用. 可以通过声明 __weak 的形式对其消除.

2. 如果是 __block 形式的声明，产生循环引用是分场景的：

	MRC 下：不会产生循环引用.
	
	ARC 下：会产生循环引用.

	可以通过断环的形式去解决. 弊端是如果 block 一直得不到调用是无法断开的.

### 6.3 怎么样理解 blcok 截获变量的特性
从被截获变量类型的分类去入手.
对于基本数据类型的局部变量是截获其值，对于对象型的局部变量是一个连同所有权修饰符一起进行的强引用截获, 对局部静态变量是对其指针进行截获,对于全局变量，全局静态变量是不截获.

### 6.4 遇到过哪些循环引用，又是怎么样解决的？

NSTimer 角度
blcok 角度,

1. 当前 block 对当前对象的成员变量有一个强引用，而当前 block 又被当前对象强持有. 造成自循环应用，通过添加 __weak 来解决自循环引用.

2. 对于 __block 所修饰的变量进行循环引用，通过断环来解决。 


