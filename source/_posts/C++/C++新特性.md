---
title: c++11新特性
date: 2023-01-17 19:40:56
tags: C++
categories: C++
---
# 智能指针
C++11 新标准在废弃 auto_ptr 的同时，增添了 unique_ptr、shared_ptr 以及 weak_ptr 这 3 个智能指针来实现堆内存的自动回收。实现原理是通过类达到作用域后的构造和析构来实现的
## 独占指针
unique_ptr 是一个独享所有权的智能指针，它提供了严格意义上的所有权，包括：

1、拥有它指向的对象

2、无法进行复制构造，无法进行复制赋值操作。即无法使两个unique_ptr指向同一个对象。但是可以进行移动构造和移动赋值操作

3、保存指向某个对象的指针，当它本身被删除释放的时候，会使用给定的删除器释放它指向的对象

## 共享指针
多个 shared_ptr 智能指针可以共同使用同一块堆内存。并且，由于该类型智能指针在实现上采用的是引用计数机制，即便有一个 shared_ptr 指针放弃了堆内存的“使用权”（引用计数减 1），也不会影响其他指向同一堆内存的 shared_ptr 指针（只有引用计数为 0 时，堆内存才会被自动释放）。

## 弱指针
weak_ptr是用来解决shared_ptr相互引用时的死锁问题,如果说两个shared_ptr相互引用,那么这两个指针的引用计数永远不可能下降为0,资源永远不会释放。它是对对象的一种弱引用，不会增加对象的引用计数，和shared_ptr之间可以相互转化，shared_ptr可以直接赋值给它，它可以通过调用lock函数来获得shared_ptr。

# 移动语义
编译器通过参数的类型实现重载函数决断，对于右值入参，优先调用形参为右值引用的函数。形参为右值引用类型的接口实现方式一般和传统接口（例如拷贝构造、拷贝赋值）实现方式不同，简单来说前者为浅拷贝，后者为深拷贝，即前者为“窃取”后者为副本复制，形如本文开篇那张图片所示（class string）

# 多线程支持
 原子类,thread类，条件变量类

 **注意** 本文测试代码见
https://github.com/dingweiqings/study/tree/master/cpp_study/src/smartpoiner
https://github.com/dingweiqings/study/tree/master/cpp_study/src/thread

 # 引用
 1. 移动语义
 https://cloud.tencent.com/developer/article/1385969
 2. 多线程
 https://immortalqx.github.io/2021/12/04/cpp-notes-3/
 3. C++智能指针
 https://www.cnblogs.com/tenosdoit/p/3456704.html