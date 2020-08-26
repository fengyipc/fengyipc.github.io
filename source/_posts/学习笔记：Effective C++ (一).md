---
title: 学习笔记：Effective C++ (一)：让自己习惯C++
tags:
  - C++
  - 学习笔记
categories: 学习笔记
abstract: >-
  1. 视 C++ 为一个语言联邦<br> 
  2. 尽量以 const, enum, inline 替换#define<br> 
  3. 尽可能使用 const<br> 
  4.确定对象被使用前已被初始化<br> 
abbrlink: 37873
date: 2020-08-26 15:58:04
---

## 01. 视 C++ 为一个语言联邦

C++ 为四门次语言组成

1. C
2. 面向对象 C++
3. Template C++
4. STL

---

## 02. 尽量以 const, enum, inline 替换 #define

1. 使用 `const` 替代 #define 常量  
> 将名称记入 symbol table 便于调试  
避免预处理器盲目替换，导致 object code 中出现多份常量的记录  
`#define` 不重视作用域，无法创建 class 专属常量
2. 使用 `emun` 替换 #define 常量
> enum { NumTurns = 5};  
比 `const` 更像 `#define`
3. 使用 `inline` 函数替代形似函数的宏
> 同样避免函数调用的额外开销，同时避免不可预料行为与类型安全问题

---

## 03. 尽可能使用 const

除非需要改动，负责一定加上 `const`，避免很多意外错误

1. 注意区分几种不同的 `const`

```c++
const int p;
const int* p;
int const* p;
int * const p;
const int * const p;
int const * const p;
```

2. const 成员函数

`mutable` 关键字：即使在 `const` 成员函数中，也可改变的成员变量。

- 将某些东西声明为 `const` 可帮助编译器侦测出错误用法。`const` 可悲施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。
- 编译器强制试试 bitwise constness，但我们编写程序应使用 logical constness。即不更改成员变量指针指向的内容。
- 当 `const` 与 `non-const` 成员函数有着实质等价的实现时，令 `non-const` 版本调用 `const` 版本可以避免代码重复。

---

## 04. 确定对象被使用前已被初始化

1. 内置类型（ C part of C++ ）：保证手动进行初始化  
非内置类型：在构造函数保证每个成员初始化

2. 注意区分「初始化」与「构造函数内赋值」，推荐使用成员初始化列表。  
在构造函数内赋值，实质是先调用默认构造函数为其设初值，再对其赋予新值。( copy assignment )

3. `const` 或者 `reference` 成员变量必须使用初始化列表，不能赋值。

4. 成员变量按声明次序被初始化，而与初始化列表中的顺序无关。

5. 通过以返回 local static 对象引用的方式，避免不同编译单元内定义的 non-local static 对象未被初始化即被使用的错误。（单例模式常用方式）
