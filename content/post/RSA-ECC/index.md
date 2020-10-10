---
title: ' RSA 与 ECC'
subtitle: '' 
summary: 
authors:
- admin
tags:
- Academic
categories:
- 
date: "2020-10-10T00:00:00Z"
lastmod: "2020-10-10T00:00:00Z"
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

# RSA 与 ECC

## RSA
利用最大素数分解原理

### 1. p,q
第一步：选取 p 和 q. 越大越好。 p, q 为不相等的质数.

### 2. n
第二步：n = p * q

### 3. m
第三步：计算m。 m 是小于或者等于 n 的正整数中与 n互为质数的个数.
利用欧拉函数：
m = (p - 1) * (q - 1)

### 4. e
第四步：计算 e. e 满足 1 < e < m, 且与 m 互为质数的随机数.

### 5. d
第五步：计算d. d 满足 (e * d)% m = 1, 就是求二元一次方程组的解;
利用的是扩展欧几里函数

### 6.如何加密？
利用费儿马小定理两端一致性.

得到公钥 (n, e) 私钥 (n, d)

利用公钥(n, e) 进行加密：
明文 a = 123,进行加密:
密文 b = (a ^ e) % n

利用私钥(n, d)解密:
a = (b ^ d) % n
### 7. 如何保证安全
如果想破解密文就需要 d, 由 第 5 步可知，需要d就需要知道 m，需要 m 就需要 p 和 q, 需要q 就需要对 n 进行因式分解. 如果 n 越大因式分解越困难。通常RSA 的加密位数是 1024 位.


## ECC
### 1. 椭圆曲线方程：
y^2 = x^3 + ax + b, 且满足 4a^3 + 27b^2 ≠ 0的方程;

### 2. 阿贝尔群
* 1、结合律
* 2、交换律
* 3、G 群， a，b ∈ G 必有 a + b ∈ G
* 4、相反数, 曲线上一点 a, 必有 -a
* 5、单位元 o， 满足 a + o = o + a = a

椭圆上的点属于阿贝尔群.

### 3. 代数加法
* 已知点椭圆曲线上两点 P 和 Q， p ≠ -q. 那么必然会有一点 R
-R = P + Q
如果 p = q, 那么必然会存在一点 R， 使得 2P = -Q, 这个时候 P 是一个切点

### 4. 标量积
3P = P + P + P


