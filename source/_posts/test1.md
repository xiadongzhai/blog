---
title: &
date: 2026-10-21 23:08:26
tags: [&，注意]
categories: [[注意，宏]]
description: '使用引用&的注意事项。'
---

function：
1. 作为变量别名，简化访问

2. 作为函数参数：修改实参 + 避免拷贝
特点：

函数内修改会影响实参

只能绑定非 const 左值

不能绑定字面量、临时对象

3. 作为函数返回值：返回原对象引用（避免局部变量销毁，引用悬空）

4. const 引用：只读、免拷贝、可接右值


一，指针交换的“假动作”
出问题的地方：

cpp
void Swap(int* x, int* y) {
    int* temp = x; x = y; y = temp;
}
为什么错：这里只交换了指针变量本身（交换了两张写着地址的纸条），根本没有顺着指针去修改外面 a 和 b 的值。

正确写法：必须加上 * 去解引用，交换它们指向的内容。

cpp
void Swap(int* x, int* y) {
    int temp = *x; *x = *y; *y = temp;
}
C++ 偷懒写法：直接用引用 void Swap(int& x, int& y)，里面正常写 int temp = x; x = y; y = temp; 就行。
二，类型转换
double d=1.14
const int& r=d; //发生转换。产生临时 int，再绑定

