---
title: test1
date: 2026-10-21 23:08:26
tags:
---
🚨 避坑一：C++ 默认参数的“潜规则”
出问题的地方：void Func1(int a = 10, int b, int c = 20)

为什么错：C++ 规定，一旦给某个参数设置了默认值，它右边的所有参数必须都有默认值。不能断开。

正确写法：void Func1(int a = 10, int b = 20, int c = 30)，或者把没有默认值的参数放在最左边。

🚨 避坑二：指针交换的“假动作”
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