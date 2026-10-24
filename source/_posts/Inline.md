---
title: Inline
date: 2026-10-24 18:45:08
tags: [c++,inline，nullptr]
categories: [[c++，inline_nullptr]]
description: '关于inline，nullptr的使用'
---

一，inline
1，效率低范例：
inline 100行代码
10000次调用
10000*100

不是inline
10000+100

2，不建议声明和定义离到两个文件，inline被展开，没有地址，建议直接在头文件定义
在.obj文件中是直接替换，而不call，

3.不进符号表，和static修饰函数相似



二，nullptr
NULL实际上是一个宏，在c++会被代替为0，而nullptr就是解决这种问题，方便指针初始化
