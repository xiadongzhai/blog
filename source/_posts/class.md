---
title: class
date: 2026-10-24 19:30:57
tags: [c++,类与对象]
categories: [[c++，类与对象]]
description: '关于类和对象的使用'
---
一，
在c++里，struct得以升级，升级为类，可以不用typedef来


1.C++一种实现封装的方式，用类将对象的属性与方法结合在一块，让对象更加完善，通过访问权限选择性的将其接口提供给外部的用户使用。

2.public修饰的成员在类外可以直接被访问；protected和private修饰的成员在类外不能直接被访问，protected和private是一样的，以后继承章节才能体现出他们的区别。

3.访问权限作用域从该访问限定符出现的位置开始直到下一个访问限定符出现时为止，如果后面没有访问限定符，作用域就到 }即类结束。

4.class定义成员没有被访问限定符修饰时默认为private，struct默认为public。

一般成员变量都会被限制为private/protected，需要给别人使用的成员函数会放为public。

访问限定符
1，public
2，private
3，protect

定义在里面的成员函数默认inline

二，this（存在栈里） 

1.Date类中有Init与Print两个成员函数，函数体中没有关于不同对象的区分，那当d1调用Init和Print函数时，该函数是如何知道应该访问的是d1对象还是d2对象呢？那么这里就要看到C++给了一个隐含的this指针解决这里的问题

2.编译器编译后，类的成员函数默认都会在形参第一个位置，增加一个当前类类型的指针，叫做this指针。比如Date类的Init的真实原型为，void Init(Date* const this, int year, int month, int day)

3.类的成员函数中访问成员变量，本质都是通过this指针访问的，如Init函数中给_year赋值，this->_year = year;

4.C++规定不能在实参和形参的位置显示的写this指针(编译时编译器会处理)，但是可以在函数体内显示使用this指针。

注：vs为了效率，会将this存在ecx寄存器里
