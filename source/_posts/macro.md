---
title: macro
date: 2026-10-22 15:49:13
tags: [macro,注意]
categories: [[注意，宏]]
description: '记录宏的注意事项'
---
缺点：复杂/类型安全检查/不能调试
优点：预处理

一，
错误范例
1.#define ADD（int a,int b) return a+b;

2.#define ADD(a,b) (a+b);
宏没有分号
3.#define ADD(a,b) (a+b)
记得a，b个体也要打括号，以防出现运算符号优先级的情况