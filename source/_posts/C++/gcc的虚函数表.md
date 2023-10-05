---
title: gcc的虚函数表
date: 2023-09-25 08:17:55
tags: C++ 虚函数 编译器
categories: C++
---
# 虚函数表
 所谓虚函数表，是编译器自动为一个带有虚函数的类生成的一块内存空间，其中存储着每一个虚函数的入口地址。由于函数的入口地址可以看成一个指针类型，因此这些虚函数的地址间隔为四个字节。而每一个带有虚函数类的实例，都拥有一个虚函数指针——vptr，在类的对象初始化完毕后，它将指向虚函数表。
## 相关概念
1. vbase offset
vbase offset全称为(Virtual Base offsets), 也即虚基类偏移。当一个class存在虚基类时，gcc编译器便会在
primary virtual table中安插相应的vbase offset。 其主要作用为：
用于访问对象的虚基类子对象。
这样的条目被添加到派生类对象vtable，以获取虚拟基类子对象的地址。每个虚基类都需要这样一个条目。这些值可以是正的，也可以是负的。
2. top offset
top offset ，也即到class顶部的偏移，指的该class的vptr到对象顶部的位移，其类型为 ptrdiff_t。 它总是存。 偏移量提供了一种使用vptr从任何基类对象中查找对象顶部的方式。 这对于 dynamic_cast<void*> 尤其必要。

3. typeinfo指针
typeinfo 指针指向用于 RTTI 的 typeinfo 对象。 它总是存在的。 给定类的每个vtable中的所有typeinfo 指针都必须指向相同的 typeinfo 对象。 typeinfo 相等性的正确实现是检查指针相等性，但指向不完整类型的指针（直接或间接）除外。 typeinfo 指针在多态类的场景下是有效指针，对于非多态类为零。
4. virtual function 指针
函数指针用于虚函数调度。 每个指针要么保存类的虚函数的地址，要么保存在将控制权转移到虚函数之前执行某些调整的辅助入口点的地址。
先有个初步的了解
```
class A
{
public:
    int a;
    virtual ~A(void) {}
};
class B : virtual public A
{
};
```
对于上面的代码,执行`g++ -fdump-lang-class code.cc`，并执行c++命名反修饰`cat code.001.class | c++filt`会得到下面的输出,其中的VTT下面会解释
```
Class A
   size=16 align=8
   base size=12 base align=8
A (0x0x7fb9e6855d80) 0
    vptr=((& A::vtable for A)   16)

Vtable for B
B::vtable for B: 8 entries
0     16
8     (int (*)(...))0
16    (int (*)(...))(& typeinfo for B)
24    (int (*)(...))B::bf
32    0
40    (int (*)(...))-16
48    (int (*)(...))(& typeinfo for B)
56    (int (*)(...))A::af

VTT for B
B::VTT for B: 2 entries
0     ((& B::vtable for B)   24)
8     ((& B::vtable for B)   56)

Class B
   size=32 align=8
   base size=12 base align=8
B (0x0x7fb9e66fe1a0) 0
    vptridx=0 vptr=((& B::vtable for B)   24)
  A (0x0x7fb9e6855e40) 16 virtual
      vptridx=8 vbaseoffset=-24 vptr=((& B::vtable for B)   56)
```
# gcc查看虚函数表
做些设置
```shell
set print asm-demangle on
set print demangle on 
set print pretty on
```
查看对象结构：p 对象名
```shell
(gdb) p p1
$1 = {_vptr$Parent = 0x400bb8 <vtable for Parent+16>}
(gdb) p p2
$2 = {_vptr$Parent = 0x400bb8 <vtable for Parent+16>}
(gdb) p d1
$3 = {<Parent> = {_vptr$Parent = 0x400b50 <vtable for Derived+16>}, <No data fields>}
(gdb) p d2
$4 = {<Parent> = {_vptr$Parent = 0x400b50 <vtable for Derived+16>}, <No data fields>}
```
dump内存: x/字节数 起始位置
x/300xb 0x400b40
```shell
(gdb) x/300xb 0x400b40
0x400b40 <vtable for Derived>:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x400b48 <vtable for Derived+8>:	0x90	0x0b	0x40	0x00	0x00	0x00	0x00	0x00
0x400b50 <vtable for Derived+16>:	0x80	0x0a	0x40	0x00	0x00	0x00	0x00	0x00
0x400b58 <vtable for Derived+24>:	0x90	0x0a	0x40	0x00	0x00	0x00	0x00	0x00
0x400b60 <typeinfo name for Derived>:	0x37	0x44	0x65	0x72	0x69	0x76	0x65	0x64
0x400b68 <typeinfo name for Derived+8>:	0x00	0x36	0x50	0x61	0x72	0x65	0x6e	0x74
0x400b70 <typeinfo name for Parent+7>:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x400b78 <typeinfo for Parent>:	0x90	0x20	0x60	0x00	0x00	0x00	0x00	0x00
0x400b80 <typeinfo for Parent+8>:	0x69	0x0b	0x40	0x00	0x00	0x00	0x00	0x00
0x400b88:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x400b90 <typeinfo for Derived>:	0x10	0x22	0x60	0x00	0x00	0x00	0x00	0x00
0x400b98 <typeinfo for Derived+8>:	0x60	0x0b	0x40	0x00	0x00	0x00	0x00	0x00
0x400ba0 <typeinfo for Derived+16>:	0x78	0x0b	0x40	0x00	0x00	0x00	0x00	0x00
0x400ba8 <vtable for Parent>:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x400bb0 <vtable for Parent+8>:	0x78	0x0b	0x40	0x00	0x00	0x00	0x00	0x00
0x400bb8 <vtable for Parent+16>:	0xa0	0x0a	0x40	0x00	0x00	0x00	0x00	0x00
0x400bc0 <vtable for Parent+24>:	0x90	0x0a	0x40	0x00	0x00	0x00	0x00	0x00
...
```

# 单继承的虚函数表
```
#include <iostream>

class Parent {
 public:
  virtual void Foo() {}
  virtual void FooNotOverridden() {}
  int a;
};

class Derived : public Parent {
 public:
  void Foo() override {}
  int b;
};

int main() {
  Parent p1, p2;
  Derived d1, d2;

  std::cout << "done" << std::endl;
}
```
子类的内存布局
![gdb中查看](/img/vtable-single.png)
|子类布局|
|-------|
|_vptr$Parent|
|int a (Parent data)|
|int b (Child data)|
虚函数表,Drived只有一个函数表，但Parent和Derived都指向Derived的虚函数表,Derived复用了父类的vptr。下面是vtable布局
![vtable布局](/img/vable-single-desc.png)

# 多继承的虚函数表
```
class Mother {
 public:
  virtual void MotherFoo() {}
   int mother_data;
};

class Father {
 public:
  virtual void FatherFoo() {}
   int father_data;
};

class Child : public Mother, public Father {
 public:
  void FatherFoo() override {}
   int child_data;
};
```
按照上面gdb的查看步骤，可以得到:
![内存布局](/img/vtable-multi-memory.png)
![vtable](/img/vtable-multi-function.png)
这里是把Monther当做主基类,Child复用Monther的vptr.观察可以发现，父类的vptr指针对于被覆盖的函数，实际上指向的子类的地址。
当做下面转换时，
```
Father f  = Child();
f.fatherFoo();
```
实际上调用的一段汇编的桩代码，类似于
gdb执行`info vtbl c`查看vtable有:
![查看vtable](/img/gdb-vtbl-info.png)
实际的内存地址可能和上面表格有出入，但顺序是一致的 
得到函数地址再反汇编,有
![反汇编覆盖的虚函数](/img/vtable-multi-adjust-this.png)
第一行`sub $0x10 %rdi`就是在调整this指针，rdi寄存器一般用来保存成员函数的第一个参数，也就是this指针.偏移量有前面的top_offset得到
**注意** 为什么需要调整this指针呢？
因为Father实际上基类，但调用的实例是Child，成员函数的this指针必须指向当前实际类型的内存空间,因为在被覆盖的成员函数可能会访问子类成员数据，this指针如果不设置正确，则可能访问错误。所有的多态调用都必须调整this指针,主基类由于和子类共用vptr不需要调整。在我们这个例子中Mother是主基类，调佣Mother的虚函数，不需要调整this指针

# 虚拟继承的虚函数表
```cpp
#include <iostream>
using namespace std;

class Grandparent {
 public:
  virtual void grandparent_foo() {}
  int grandparent_data;
};

class Parent1 : virtual public Grandparent {
 public:
  virtual void parent1_foo() {}
  int parent1_data;
};

class Parent2 : virtual public Grandparent {
 public:
  virtual void parent2_foo() {}
  int parent2_data;
};

class Child : public Parent1, public Parent2 {
 public:
  virtual void child_foo() {}
  int child_data;
};

int main() {
  Child child;
}
```
还是按照上面步骤来看,打印Child的内存布局
![gdb内存布局](/img/gdb-object-virtual.png)
![内存布局](/img/vtable-virtual.png)
可以发现，实际上虚继承的每个父类都只有一份内存拷贝，相当于直接继承GrandParent。下面是vtable
![vtable布局](/img/vtable-virtual-layout.png)
![gdb内存布局](/img/vtable-virtual-layout-gdb.png)
实际的内存地址可能和上面表格有出入，但顺序是一致的
这里多出来了construct table和vtt。这篇[反汇编gcc编译bin的论文](https://github.com/bingseclab/VirtAnalyzer/blob/master/paper.pdf)有提到这些是干什么的.下面我也会解释这两个的作用。
**注意** construct table和vtt在内存中实际存在的,要占用内存空间
construct table是在实例化Child需要用到的,比如在实例化Child时，会最先实例化GrandParent(相当于Child直接继承),然后再实例化Parent1,Parent2.但GrandParent已经有了，如何告诉Parent1呢？这里就用到了virtual base offset和construct table，告诉Parent1，GrandParent已经在this-32的位置(这里this是指向Child的内存首地址,因为是构造Child).这样就可以把类拼起来了。top_offset还是子类到Child数据成员首地址的偏移量。
![Parent1 construct table](/img/vtable-parent1-construct.png)
还有一个东西就是Vtable table，用来记录多个vtable的位置，充当一个索引的作用
![vtt](/img/vtable-vtt.png)

# 虚函数表如何解决面向对象中的问题
## 实现多态
1. 基类的vptr指向子类的虚函数
2. 在调用时，根据子类记录的基类偏移量来调整this指针
## 遗留的问题
我搞不懂既然vptr是指向function pointer，那编译器怎么直到vptr前面有几个才是vtable的首地址呢，第几项是vbase offset呢？如果有多个虚继承的类，又当怎么vbase offset呢？
# 继承结构可视化
在有一篇反汇编gcc编译出的bin论文中有提到说可以可视化vtable,但我是没跑通，最后需要下载ida pro这个反编译工具,没跑通
https://github.com/bingseclab/VirtAnalyzer/blob/master/README.md

# 引用
1. 深入探索C++对象-候捷
2. [反汇编gcc编译bin的论文](https://github.com/bingseclab/VirtAnalyzer/blob/master/paper.pdf)
2. [LLVM虚函数表](https://llvm.org/devmtg/2021-11/slides/2021-RelativeVTablesinC.pdf)
3. [vtable单继承详解](https://shaharmike.com/cpp/vtable-part1/)
3. [vtable多重继承详解](https://shaharmike.com/cpp/vtable-part2/)
3. [vtable虚继承详解](https://shaharmike.com/cpp/vtable-part3/)