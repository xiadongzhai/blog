---
title: string
date: 2026-11-15 22:17:31
tags: [c++, string, STL, 模拟实现]
description: 'C++ string 全面笔记：STL 与 basic_string、构造/容量/遍历/修改/查找/比较全套接口、扩容倍数与 SSO、迭代器四类与解耦、string 的模拟实现（深拷贝、赋值现代写法、insert/erase 挪动、substr、比较运算符、operator>>/getline）'
---

> 内容来源：
> - `25年-05月-10日--string`：STL 简介与版本、六大组件、`basic_string` 家族与编码、string 的构造与赋值、三种遍历方式、`auto` 的四种推导、迭代器的意义
> - `25年-05月-13日--string`：`const_iterator` / `reverse_iterator`、`size/capacity/max_size`、`reserve/shrink_to_fit/resize/clear`、`operator[]` 与 `at`、`push_back/append/+=/assign/insert/erase/replace`、`find/npos`、扩容倍数实测、替换空格的两个解法
> - `25年-05月-15日--string`：模拟实现 string 第一版（构造/析构/`c_str`/`operator[]`/`begin/end`）、`fopen` + `c_str` 读文件、`rfind` + `substr` 取后缀、`split_url` 返回 `vector<string>`、比较运算符三类重载、`cin >>` 与 `getline`
> - `25年-05月-18日--string`：拷贝构造（深拷贝）、`reserve/push_back/append/+=/pop_back`、`insert/erase` 的挪动边界、`find` 用 `strstr`（KMP 见文末链接）、`substr`、六个比较运算符的写法、`operator<<` 重载、`static const size_t npos`
> - `25年-05月-20日--string+ vector`：赋值运算符的传统写法与现代写法（`tmp` + `swap`）、`swap` 成员与非成员、`clear`、`operator>>` 与 `getline` 的自实现（128 字节缓冲）、SSO 小字符串缓冲 `_buff[16]`、临时对象与返回值优化，末尾引入 `vector`

## 目录

- **一、STL 与 string 的关系**
- **二、string 的常用接口全景**
- **三、三种遍历方式与迭代器**
- **四、容量接口：size / capacity / reserve / resize**
- **五、修改接口与 O(N) 数据挪动**
- **六、模拟实现 string（三版演进）**
- **七、拷贝控制：深拷贝、赋值、swap**
- **八、输入输出：operator<< / operator>> / getline**
- **九、OJ 实战**
- **十、速查表与易错清单**

---

## 一、STL 与 string 的关系

### 1.1 STL 是什么

STL（standard template library，标准模板库）是 C++ 标准库的重要组成部分，**不只是一批可复用的组件库，而是一个包罗数据结构与算法的软件框架**。

```text
C++ 标准库
├── IO 库
├── C 库
└── STL
```

### 1.2 六大组件

| 组件 | 代表 | 说明 |
| --- | --- | --- |
| 容器 | `vector` `list` `deque` `map` `set` `multimap` `multiset` | 数据结构 |
| 算法 | `find` `swap` `reverse` `sort` `merge` | 与底层结构解耦 |
| 迭代器 | `iterator` `const_iterator` `reverse_iterator` `const_reverse_iterator` | 容器与算法的桥梁 |
| 仿函数 | `greater` `less` | 定制比较规则 |
| 配器（适配器） | `stack` `queue` `priority_queue` | 容器适配 |
| 空间配置器 | `allocator` | 内存池管理 |

### 1.3 STL 的版本

- **原始版本**：Alexander Stepanov、Meng Lee 在惠普实验室完成，开源，允许任意运用/拷贝/修改/传播/商业使用，唯一条件是也要向原始版本一样开源——HP 版本是所有 STL 实现版本的始祖。
- **P.J. 版本**：由 P.J. Plauger 开发，继承自 HP 版本，被 Windows Visual C++ 采用，**不能公开或修改**，可读性较差，符号命名比较怪异。
- **RW 版本**：由 Rouge Wave 公司开发，继承自 HP 版本，被 C++ Builder 采用，**不能公开或修改**，可读性一般。
- **SGI 版本**：由 Silicon Graphics Computer Systems, Inc 公司开发，继承自 HP 版本，被 GCC(Linux) 采用，**可移植性好，可公开、修改甚至贩卖**，命名风格和编程风格上看，阅读性非常高。**学习 STL 阅读部分源码，主要参考的就是这个版本。**

三个境界：**熟练使用 STL → 了解泛型技术的内涵与 STL 的学理乃至实现 → 扩充 STL（造轮子）**。

### 1.4 为什么 string 在"六大组件"里

`string` 属于**容器**，而且它本身就是模板类实例化出来的：

```cpp
typedef basic_string<char> string;
```

`basic_string` 是一个**类模板**，通过不同的字符类型 + 字符特性 + 空间配置器实例化出一族字符串：

| 类型 | 定义 | 编码 |
| --- | --- | --- |
| `std::string` | `basic_string<char>` | ascii / utf-8（字节序） |
| `std::wstring` | `basic_string<wchar_t>` | unicode（宽字符） |
| `std::u16string`(C++11) | `basic_string<char16_t>` | utf-16 |
| `std::u32string`(C++11) | `basic_string<char32_t>` | utf-32 |

> 板书：`string` 本质上是**char 字符数组的顺序表**。所以它的 `size()/capacity()/迭代器` 都是**按字节（code unit）**计数的，不是按"字符"计数。

### 1.5 编码：UTF-8 是怎么编码的

UTF-8 编码最少的为 1 个字节，最长的目前为 4 个字节，**从首字节就可以判断一个 UTF-8 编码有几个字节**：

- 如果首字节以 **0** 开头，则说明是单字节编码（即一个 ASCII 字符）；
- 如果首字节以 **110** 开头，则是双字节编码（即由 2 个字节组成的码元序列）；
- 如果首字节以 **1110** 开头，则是三字节编码（即由 3 个字节组成的码元序列），以此类推。

这解释了一个经典现象：

```cpp
char str1[] = "ascii";   // 5 字节 + '\0'
char str2[] = "比特";    // 6 字节 + '\0'（每个汉字 utf-8 占 3 字节）
str2[1]++;               // 改的是某个汉字的中间字节 —— 输出必然乱码（"连蒙带猜"）
```

> "连蒙带猜"：一个汉字被拆成 3 个字节，单独动其中一个字节，其它两个字节还在，终端还是会尝试拼一下，于是显示成奇怪的字符。

---

## 二、string 的常用接口全景

### 2.1 构造（向前兼容的 7 种形态）

```cpp
void test_string1()
{
    string s1;                 // 空串
    string s2("hello world");  // 常用
    string s3(s2);             // 拷贝构造
    string s4(s2, 1, 6);       // s2 从下标 1 开始取 6 个："ello w"
    string s5(s2, 1, 60);      // 长度超长自动截断到结尾
    string s6(s2, 1);          // 从下标 1 取到结尾
    const char* str = "hello world";
    string s7(str, 5);         // 用前 5 个字符："hello"
    string s8(100, '#');       // 100 个 '#'
}
```

### 2.2 赋值

```cpp
s1 = s2;      // 拷贝赋值
s1 = "xxxx";  // 字符串字面值
s1 = 'x';     // 单个字符（C++11）
```

### 2.3 容量与访问速览

| 接口 | 作用 | 备注 |
| --- | --- | --- |
| `size()` | 有效字符个数 | 不含 `'\0'` |
| `capacity()` | 已分配空间能存的字符数 | 不含 `'\0'` |
| `max_size()` | 理论最大长度 | 实际用不到 |
| `empty()` | 是否为空 | 推荐用它而不是 `size()==0` |
| `clear()` | 清空内容 | **只改 size，不释放空间** |
| `operator[]` | 下标访问 | 越界是**断言**（debug）/ 未定义行为 |
| `at()` | 下标访问 | 越界**抛异常** `out_of_range` |
| `c_str()` | 返回 `const char*` | 给 C 接口用 |

```cpp
string s1("hello world");
s1[0]++;      // s1.operator[](0)++
// s2[0]++;   // const string 不能改，走 const 版本返回 const char&

s1[15];       // 断言失败（debug 下直接挂）
s1.at(15);    // 抛 out_of_range，可被 try/catch 接住

int a[10];
a[15] = 1;    // 原生数组：*(a+15) = 1，完全没有检查
```

> `operator[]` 与 `at` 的分工：**想要性能用 `[]`（只断言），想要安全用 `at`（越界抛异常，可捕获）**。

---

## 三、三种遍历方式与迭代器

### 3.1 方式一：下标 + `[]`

```cpp
for (size_t i = 0; i < s1.size(); i++)
{
    s1[i]++;   // 遍历 + 修改
}
```

这是 string 独有的便利（链表就没有），但**只适合支持随机访问的容器**。

### 3.2 方式二：迭代器（所有容器的主流方式）

```cpp
string::iterator it1 = s1.begin();
while (it1 != s1.end())      // 区间是 [begin(), end())
{
    (*it1)--;
    ++it1;
}
```

四类迭代器：

```cpp
string::iterator               it1 = s1.begin();   // 可读可写
string::const_iterator         it2 = s2.begin();   // 只读（s2 是 const string）
string::reverse_iterator       it3 = s1.rbegin();  // 反向可写
string::const_reverse_iterator it4 = s2.rbegin();  // 反向只读

// (*it2)--;   // 编译不通过：const_iterator 修饰的是指向的数据
// *it4 = 'x'; // 编译不通过
```

| 迭代器 | 能否改指向的数据 | 能否改自身（++/--） |
| --- | --- | --- |
| `iterator` | ✔ | ✔ |
| `const_iterator` | ✘ | ✔ |
| `reverse_iterator` | ✔ | ✔（从后往前） |
| `const_reverse_iterator` | ✘ | ✔ |

> 关键认识：**`const_iterator` 修饰的是"指向的数据不能改"，不是"迭代器自身不能改"**。它对应 `const char*`；而 `const` 迭代器自身（`char* const`）在实战中基本没用。

### 3.3 迭代器的意义（重中之重）

板书写得很直接：

> **1. 统一类似的方式遍历、修改容器。**
> **2. 算法脱离具体底层结构，和底层结构解耦，算法独立模板实现，针对多个容器处理。**

```cpp
// 同一份 reverse 模板，同时作用于 string / vector / list
reverse(s1.begin(), s1.end());
reverse(v.begin(),  v.end());
reverse(lt.begin(), lt.end());
```

`std::reverse` 的原型是个**函数模板**，只要求 `BidirectionalIterator`：

```cpp
template <class BidirectionalIterator>
void reverse(BidirectionalIterator first, BidirectionalIterator last);
```

它的行为等价于：

```cpp
template <class BidirectionalIterator>
void reverse(BidirectionalIterator first, BidirectionalIterator last)
{
    while ((first != last) && (first != --last))
    {
        std::iter_swap(first, last);
        ++first;
    }
}
```

**`reverse` 完全不知道底层是数组、链表还是红黑树**——它只用 `!=`、`++`、`--`、`*` 这几个操作。这就是"降低耦合、降低关联关系"。

> 迭代器可以**想象成指针一样的类型对象**：对 `string`/`vector` 它其实就是原生指针（`typedef char* iterator;`），对 `list` 它是个封装了节点的类。

### 3.4 方式三：范围 for（C++11 语法糖）

```cpp
for (auto& e : s1)   // 引用 → 可修改
{
    e--;
}

for (auto e : s1)    // 值拷贝 → 只读
{
    cout << e << " ";
}

int array[] = { 1,2,3,4,5 };
for (auto e : array) { }    // 原生数组也可以
```

编译器做的事：**自动取容器数据赋值给 `e`、自动判断结束、自动迭代**——底层就是替换成迭代器 `auto e = *it`：

```cpp
// for (auto& e : s1)  ≈
for (auto it = s1.begin(); it != s1.end(); ++it)
{
    auto& e = *it;
    ...
}
```

> 所以要修改元素，`auto` 后面必须加 `&`；写 `for (auto e : s1)` 改的是临时拷贝，原串不变。

### 3.5 `auto` 的四种推导细节

```cpp
auto  x  = 10;      // int
auto  y  = 10.1;    // double
int&  z  = x;
auto& m  = z;       // int&（auto& 保留引用）
auto  p1 = &x;      // int*
auto* p2 = &x;      // int*（auto* 只能是指针，更明确）
```

---

## 四、容量接口：size / capacity / reserve / resize

### 4.1 `size()` 与 `capacity()` 都不含 `'\0'`

```cpp
string s1("123456");
cout << s1.max_size() << endl;
cout << s1.size() << endl;       // 6
cout << s1.capacity() << endl;   // 一般 >= 6
```

内部结构是"指针 + 长度 + 容量"：

```text
┌──────┬──────┬───────────┐        ┌─────────────────┐
│ _str │ _size│ _capacity │ ─────► │ "hello world\0" │
└──────┴──────┴───────────┘        └─────────────────┘
        begin() ↑          ↑ end() = _str + _size
```

### 4.2 扩容倍数：VS 1.5 倍，g++ 2 倍

```cpp
string s2;
size_t old = s2.capacity();
cout << "capacity:" << old << endl;
for (size_t i = 0; i < 100; i++)
{
    s2.push_back('x');
    if (s2.capacity() != old)
    {
        cout << "capacity:" << s2.capacity() << endl;
        old = s2.capacity();
    }
}
```

实测输出：

```text
VS2019（1.5 倍，除了第一次）        g++（2 倍）
15                                  15
16                                  30
31                                  60
47                                  120
70
105
```

> 没有规定倍数，标准只保证 `push_back` 均摊 O(1)。**1.5 倍增长比 2 倍更省内存（有机会复用相邻的老空间），2 倍增长在时间上更快**。

### 4.3 `reserve(n)`：提前开好空间

```cpp
string s2;
// 确定知道需要多少空间，提前开好，避免扩容，提高效率
s2.reserve(100);
// 之后 push_back 100 次，capacity 一次都不变
```

- `reserve(n)`：**只影响 capacity，不改 size**。`n > capacity` 才真正扩容。
- `reserve` 缩容"不靠谱"——标准只要求"不会主动缩"，实现可以直接忽略：

```cpp
string s1("123456");
s1.reserve(100);   // capacity 变大
s1.reserve(3);     // 期望缩容 → 实际上 capacity 还是 100（被忽略）
```

> 想真正缩容要走 `shrink_to_fit()` 或"拷贝交换"。

### 4.4 `resize(n, ch)`：三种情况

```cpp
string s2("123456");          // size=6  capacity=?
s2.resize(20, 'x');           // ① n > capacity > size：扩容 + 用 'x' 填满到 20
s2.resize(25);                // ② capacity > n > size：不扩容，多出来的位置填 '\0'
s2.push_back('y');            //    注意：size 之后仍有 '\0'，push_back 能接着写
s2.resize(5);                 // ③ n < size：截断，删除多余数据（capacity 不变）
```

要点：
1. **`resize` 会改 `size`**（区别于 `reserve`）。
2. `n > size` 时新增位置**默认填 `'\0'`**，也可以指定填充字符。
3. `n < size` 时**只截断 size，不释放空间**。

### 4.5 `clear()` 与 `shrink_to_fit()`

```cpp
s1.clear();     // size 变 0，capacity 不变

// 不会缩容，缩容要重新开空间，代价很大
for (size_t i = 0; i < 50; i++) s2.pop_back();
s2.shrink_to_fit();   // 非约束请求，按需缩容
```

> 工程上的原则：**缩容意味着重新申请 + 拷贝 + 释放，代价很大（"以时间换空间" vs "以空间换时间"），除非确实要省内存，否则不要缩**。这也是 `clear()` 不释放空间的原因——方便复用。

### 4.6 SSO（小字符串优化）

```cpp
class string
{
private:
    char*  _str  = nullptr;
    size_t _size = 0;
    size_t _capacity = 0;

    // _size < 16 ：串存在 _buff 数组中
    // _size >= 16：串存在 _str 指向的数组中
    char _buff[16];
};
```

```cpp
std::string s1("111111");                            // 短串 → 存在对象内部的 _buff 里
std::string s2("1111111111111111111111111111111");   // 长串 → 堆上
cout << sizeof(s1) << " " << sizeof(s2) << endl;     // 两个一样大
```

**短字符串不申请堆空间**，直接存在对象内部的定长数组里——既快又省内存，也没有 `new/delete` 的开销。所以我们看到 `sizeof(std::string)`（VS 上 32 字节）远大于"三指针"的 24。

> 我们模拟实现时用 `_str + _size + _capacity` 的裸指针版本，**功能等价但没有 SSO**——这是一个很好的面试对比点。

---

## 五、修改接口与 O(N) 数据挪动

### 5.1 尾插：`push_back` / `append` / `+=`

```cpp
string s1("hello world");
s1.push_back(' ');           // 尾插一个字符
s1.append("hello bit");      // 尾插字符串
s1.append(10, 'x');          // 尾插 10 个 'x'

s1 += ' ';                   // 最常用
s1 += "hello world";
```

`+=` 是"万能尾插"，可读性最好，**优先用它**。

### 5.2 `assign` / `=` / `insert` / `erase` / `replace`

```cpp
s1.assign(10, 'y');          // 清空后填入 10 个 'y'
s1 = 'x';                    // 直接变成 "x"

// insert 谨慎使用，底层数据挪动，效率低下，O(N)
string s2("hello world");
s2.insert(0, "yyy");         // 头插字符串
s2.insert(0, 1, '!');        // 头插 1 个 '!'
s2.insert(s2.begin(), '!');  // 迭代器版本

// erase 谨慎使用，底层数据挪动，效率低下，O(N)
string s3("hello world");
s3.erase(5, 1);              // 从 5 删 1 个
s3.erase(5);                 // 从 5 删到结尾

// replace 谨慎使用，底层数据挪动，效率低下，O(N)
string s4("hello   world");
s4.replace(5, 3, "#");       // 把 [5, 5+3) 换成 "#"
s4.replace(5, 1, "!!!!!!!!!");
```

> **三兄弟（`insert`/`erase`/`replace`）都是 O(N)**：它们要在原地挪动后面所有数据。头插/头删尤其贵。

### 5.3 经典案例：所有空格替换为 `%%`

**能跑但很慢的写法**：

```cpp
string s5("                    hello world hello bit");
size_t pos = s5.find(' ');
while (pos != string::npos)
{
    s5.replace(pos, 1, "%%");   // 每次 replace 都要挪动后面 O(N) 个字符
    pos = s5.find(' ', pos + 2);
}
// 总复杂度 O(N^2)
```

**正确做法：开新空间（时间 O(N)，空间 O(N)）**

```cpp
string s6;
s6.reserve(s5.size());      // 提前开好，避免反复扩容
for (auto ch : s5)
{
    if (ch == ' ')
        s6 += "%%";
    else
        s6 += ch;
}
```

**另一条路：从后往前**（牛客原题要求原地 + 从后往前，见 `ACM技巧收集.md`）——先统计空格数算好新长度，再从尾到头双指针搬运，避免从前往后反复挪动。

> 归纳：**遇到"原地大量插删"，先想"能不能从后往前"**。

### 5.4 `npos` 是什么

```cpp
static const size_t npos = -1;   // 即 size_t 的最大值 4294967295 / 18446744073709551615
```

用它表示"没有找到"，因为**任何合法的下标都不可能等于它**。判断时用 `pos != string::npos`，不要写 `pos != -1`（有符号/无符号比较隐患）。

---

## 六、模拟实现 string（三版演进）

### 6.1 第一版（5-15）：跑起来

`string.h`：

```cpp
#pragma once

#include <string.h>
#include <assert.h>

namespace bit
{
    class string
    {
    public:
        typedef char* iterator;

        iterator begin();
        iterator end();

        string(const char* str = "");   // 默认构造 + 带参构造合一
        const char* c_str() const;
        ~string();

        size_t size() const;
        char& operator[](size_t i);
        const char& operator[](size_t i) const;

    private:
        char*  _str;
        size_t _size;
        size_t _capacity;
    };
}
```

`string.cpp`：

```cpp
namespace bit
{
    string::string(const char* str)
        :_size(strlen(str))
    {
        _capacity = _size;
        _str = new char[_size + 1];    // +1 放 '\0'
        strcpy(_str, str);
    }

    string::~string()
    {
        delete[] _str;
        _str = nullptr;
        _size = _capacity = 0;
    }

    string::iterator string::begin() { return _str; }
    string::iterator string::end()   { return _str + _size; }

    const char* string::c_str() const { return _str; }
    size_t string::size() const       { return _size; }

    char& string::operator[](size_t i)
    {
        assert(i < _size);
        return _str[i];
    }

    const char& string::operator[](size_t i) const
    {
        assert(i < _size);
        return _str[i];
    }
}
```

**为什么不需要单独写无参构造？** 因为 `string(const char* str = "")` 用缺省参数把"无参"和"带参"合并了，还能直接 `string s1;`。

**为什么容量要 +1？** `_str[_size]` 必须能放 `'\0'`，`c_str()` 才能直接交给 C 接口（`fopen`、`printf`、`strstr`…）。

**迭代器就是原生指针**：`begin()` 返回 `_str`，`end()` 返回 `_str + _size`（左闭右开），所以 `for (auto ch : s2)` 能直接工作。

### 6.2 第二版（5-18）：补齐增删查改

头文件补上 `const_iterator` / 拷贝构造 / `reserve` / `push_back` / `append` / `+=` / `pop_back` / `insert` / `erase` / `find` / `substr` / 六个比较运算符 / `operator<<`，以及 `static const size_t npos`。

**`const` 版迭代器（让范围 for 支持 const 对象）**：

```cpp
typedef char*       iterator;
typedef const char* const_iterator;

iterator       begin();
iterator       end();
const_iterator begin() const;   // const 对象调用这个
const_iterator end()   const;
```

**`npos` 的定义放哪**：

```cpp
// .h 里声明
static const size_t npos;

// .cpp 里定义
const size_t string::npos = -1;
```

> 注意 `size_t` 类型 + `-1` 会隐式转换成最大值，这正是标准库原样的做法。

**拷贝构造（深拷贝）**：

```cpp
string::string(const string& s)
{
    _str = new char[s._capacity + 1];   // 按被拷贝者的容量开，而不是 size
    memcpy(_str, s._str, s._size + 1);  // +1 连 '\0' 一起拷
    _size = s._size;
    _capacity = s._capacity;
}
```

> 不写拷贝构造会怎样？编译器生成默认的**浅拷贝**（只拷指针），析构时同一块内存 `delete[]` 两次 → **double free 崩溃**；而且改一个串另一个也跟着变。

**`reserve`**：

```cpp
void string::reserve(size_t n)
{
    if (n > _capacity)                    // 只扩容，不缩容
    {
        char* tmp = new char[n + 1];
        memcpy(tmp, _str, _size + 1);
        delete[] _str;
        _str = tmp;
        _capacity = n;
    }
}
```

**`push_back`（2 倍扩容）**：

```cpp
void string::push_back(char ch)
{
    if (_size >= _capacity)
    {
        size_t newcapacity = _capacity == 0 ? 4 : 2 * _capacity;
        reserve(newcapacity);
    }
    _str[_size] = ch;
    ++_size;
    _str[_size] = '\0';       // 补 '\0'
}
```

**`append`（按需扩容，取 2 倍与"刚好够"的较大者）**：

```cpp
void string::append(const char* str)
{
    size_t len = strlen(str);
    if (_size + len > _capacity)
    {
        size_t newcapacity = 2 * _capacity > _size + len ? 2 * _capacity : _size + len;
        reserve(newcapacity);
    }
    memcpy(_str + _size, str, len + 1);   // +1：把 str 的 '\0' 也搬过来
    _size += len;
}
```

> 这里用 `memcpy` 而不是 `strcpy`：长度已知，`memcpy` 更快，且 `str` 中间若有 `'\0'` 也能完整搬过去。

**`pop_back`**：

```cpp
void string::pop_back()
{
    assert(_size > 0);
    --_size;
    _str[_size] = '\0';
}
```

**`insert` 单个字符——最容易写错的地方**：

```cpp
string& string::insert(size_t pos, char ch)
{
    assert(pos <= _size);

    if (_size >= _capacity)
    {
        size_t newcapacity = _capacity == 0 ? 4 : 2 * _capacity;
        reserve(newcapacity);
    }

    // 写法 A：把 end 用 int，避免 size_t 减到 -1 变成天文数字而越界死循环
    /*
    int end = _size;
    while (end >= (int)pos)
    {
        _str[end + 1] = _str[end];
        --end;
    }
    */

    // 写法 B（推荐）：把 end 定为 _size+1，循环条件用 end > pos，全程 size_t 不下溢
    size_t end = _size + 1;
    while (end > pos)
    {
        _str[end] = _str[end - 1];
        --end;
    }

    _str[pos] = ch;
    ++_size;
    return *this;
}
```

`insert(pos, const char*)` 的边界对比（板书原图）：

```cpp
size_t end = _size + len;
while (end > pos)             // ✘ 少挪一段：pos 右侧数据会丢
{
    _str[end] = _str[end - len];
    --end;
}

size_t end = _size + len;
while (end > pos + len - 1)   // ✔ 最后一个要挪动的数据是 pos+len
{
    _str[end] = _str[end - len];
    --end;
}
```

> 口诀：**先算好"最后要挪到哪"，再倒着搬**。终止位置写错一位，就是"数据错位"和"越界写"两种 bug。

**`erase`（用 `memmove` 一次搬完）**：

```cpp
string& string::erase(size_t pos, size_t len)
{
    assert(pos < _size);

    // 要删除的数据 >= pos 后面的字符个数 → pos 后面全删
    if (len == npos || len >= (_size - pos))
    {
        _size = pos;
        _str[_size] = '\0';
    }
    else
    {
        size_t i = pos + len;
        memmove(_str + pos, _str + i, _size + 1 - i);   // 必须 memmove：源与目标重叠
        _size -= len;
    }
    return *this;
}
```

> **为什么必须 `memmove` 而不是 `memcpy`？** 源区间和目标区间**重叠**，`memmove` 保证重叠安全（内部判断方向拷贝），`memcpy` 重叠是未定义行为。

**`find` 两个版本**：

```cpp
size_t string::find(char ch, size_t pos) const
{
    for (size_t i = pos; i < _size; i++)
        if (_str[i] == ch) return i;
    return npos;
}

size_t string::find(const char* str, size_t pos) const
{
    // kmp —— 标准库实现一般会做更聪明的优化
    const char* p1 = strstr(_str + pos, str);
    if (p1 == nullptr) return npos;
    return p1 - _str;      // 指针相减得到下标
}
```

> `strstr` 是暴力匹配 O(N*M)。想更快就上 **KMP**——课上给了讲解链接（B 站《KMP 算法》 BV1UL411E7M8）。

**`substr`**：

```cpp
string string::substr(size_t pos, size_t len) const
{
    if (len == npos || len >= _size - pos)
        len = _size - pos;

    string ret;
    ret.reserve(len);                 // 提前开好，避免 += 过程中反复扩容
    for (size_t i = 0; i < len; i++)
        ret += _str[pos + i];

    return ret;                       // 返回临时对象（→ 见 7.3）
}
```

**比较运算符：只写 `<` 和 `==`，其余复用**：

```cpp
// s1 < s2
// "hello"  "hello"   -> false
// "hellox" "hello"   -> false
// "hello"  "hellox"  -> true
bool string::operator<(const string& s) const
{
    size_t i1 = 0, i2 = 0;
    while (i1 < _size && i2 < s._size)
    {
        if (_str[i1] < s[i2])      return true;
        else if (_str[i1] > s[i2]) return false;
        else { ++i1; ++i2; }
    }
    return i2 < s._size;           // 前缀相等时，短的小
}

bool string::operator<=(const string& s) const { return *this < s || *this == s; }
bool string::operator>(const string& s)  const { return !(*this <= s); }
bool string::operator>=(const string& s) const { return !(*this < s); }
bool string::operator!=(const string& s) const { return !(*this == s); }
```

> 只需实现 `<` 和 `==`，另外四个都是组合/取反——**少写一半代码，也不容易写不一致**。

### 6.3 第三版（5-20）：拷贝控制 + IO + SSO

新增 `swap` 成员 / 非成员、赋值运算符（现代写法）、`clear`、`operator>>`、`getline`，并给出 SSO 的 `_buff[16]` 布局（见 4.6 与第七、八章）。

### 6.4 工程细节：`c_str()` 的实战价值

```cpp
void test_string8()
{
    string filename("test.cpp");
    FILE* fout = fopen(filename.c_str(), "r");   // C 接口只认 const char*
    if (fout == nullptr) { cout << "fopen error" << endl; return; }

    char ch = fgetc(fout);
    while (ch != EOF)
    {
        cout << ch;
        ch = fgetc(fout);
    }
}

string findsuffix(const string& filename)
{
    size_t i = filename.rfind('.');
    if (i != string::npos)
        return filename.substr(i);      // ".cpp" / ".zip"
    else
        return "";
}
```

```cpp
vector<string> split_url(const string& url)
{
    vector<string> v;

    size_t i1 = url.find(':');
    if (i1 != string::npos) v.push_back(url.substr(0, i1));      // "http"

    size_t i2 = i1 + 3;                 // 跳过 "://"
    size_t i3 = url.find('/', i2);
    if (i3 != string::npos)
    {
        v.push_back(url.substr(i2, i3 - i2));                    // 域名
        v.push_back(url.substr(i3 + 1));                         // 路径
    }
    return v;
}
// 得到 { "http", "legacy.cplusplus.com", "reference/string/string/" }
```

> 这是"短字符串返回值"的典型：`substr` 返回临时对象，函数再 `return v`——现代编译器会做 **RVO / 移动**，不会真的深拷贝一份。

---

## 七、拷贝控制：深拷贝、赋值、swap

### 7.1 拷贝构造：现代写法（`tmp` + `swap`）

```cpp
void string::swap(string& s)
{
    std::swap(_str, s._str);
    std::swap(_size, s._size);
    std::swap(_capacity, s._capacity);
}

// s3(s1)
string::string(const string& s)
{
    string tmp(s._str);     // 复用构造函数造一份
    swap(tmp);              // 把 tmp 的资源换过来
}                           // tmp 出作用域析构，释放掉原来的空资源
```

**好处**：不用手写 `new[]` / `memcpy` / 赋值三件套，逻辑全部复用构造函数，**不会漏写字段**。

### 7.2 赋值运算符：传统写法 vs 现代写法

**传统写法（需要自己判自赋值、自己管异常安全）**：

```cpp
string& string::operator=(const string& s)
{
    if (this != &s)                  // 必须判自赋值
    {
        char* tmp = new char[s._capacity + 1];   // 先申请新空间
        memcpy(tmp, s._str, s._size + 1);
        delete[] _str;                           // 再释放旧空间：new 失败时对象仍可用
        _str = tmp;
        _size = s._size;
        _capacity = s._capacity;
    }
    return *this;
}
```

**现代写法（推荐）：传值 + `swap`**

```cpp
// s1 = s2
string& string::operator=(string tmp)   // 参数传值，tmp 已经是 s2 的深拷贝
{
    swap(tmp);                          // 直接换
    return *this;
}                                       // tmp 析构，释放 s1 原来的空间
```

三个好处：
1. **不用判自赋值**——`s1 = s1` 时 `tmp` 是独立拷贝，交换后照样正确。
2. **天然异常安全**：拷贝失败在传参阶段就抛异常，此时 `*this` 完全没被改动。
3. **代码短**，且和拷贝构造共享逻辑。

**还能更省：参数 `string s` 由编译器选择拷贝还是移动**

```cpp
// s1 = "xxx"  → 传参时直接构造 tmp，省一次拷贝
// s1 = s2     → 传参时拷贝构造 tmp
string& operator=(string tmp);
```

### 7.3 函数返回与临时对象

`substr` 里：

```cpp
string ret;
ret.reserve(len);
...
return ret;         // 先构造临时对象（返回值），再交给接收者
```

```cpp
string ret = url.substr(0, i1);   // 直接用临时对象构造 ret
cout << ret << endl;
```

板书标注了两条路：**`return ret` 会先构造一个临时对象（返回值），再用临时对象拷贝构造 `ret`**；而写成 `string s = url.substr(...)` 时，编译器会做 **拷贝省略（RVO）/ 移动构造**，实际只构造一次。这也是为什么"函数返回容器"在现代 C++ 里并不慢。

### 7.4 `swap` 的三种说法

```cpp
std::swap(s1, s2);   // 标准库通用 swap：三次拷贝（对 string 来说代价大）
s1.swap(s2);         // 成员 swap：只交换三个成员，O(1)（推荐）
swap(s1, s2);        // 库里也有针对 basic_string 的重载；
                     // 自己的命名空间里常再写一个非成员 swap 版本：
void swap(string& x, string& y) { x.swap(y); }
```

> 关键认识：**通用 `std::swap` 的语义是 `T c(a); a = b; b = c;`——三次拷贝**。对持有堆资源的 string/vetor 来说代价很大，所以要特化/重载成"只换指针"。

### 7.5 赋值的验证实验

```cpp
string s1("hello"), s2("worldxxxxx");
string s3(s1);
cout << s1 << endl;    // hello
cout << s3 << endl;    // hello
s1[0] = 'x';
cout << s1 << endl;    // xello   （深拷贝：改 s1 不影响 s3）
cout << s3 << endl;    // hello

s1 = s2;
cout << s1 << endl;    // worldxxxxx
cout << s2 << endl;    // worldxxxxx（互不影响）
```

**如果只做浅拷贝**，`s1[0]='x'` 会把 `s3` 也改成 `xello`，析构时还会 double free。

---

## 八、输入输出：operator<< / operator>> / getline

### 8.1 `operator<<`

```cpp
ostream& operator<<(ostream& out, const string& s)
{
    // out << s.c_str();          // 遇到中间的 '\0' 会截断
    for (size_t i = 0; i < s.size(); i++)
        out << s[i];              // 按 size 输出，能输出中间的 '\0'
    return out;
}
```

**`size()` 与 `c_str()` 的差别**，一段实验就能看出来：

```cpp
std::string s2("hello world");
s2 += '\0';
s2 += '\0';
s2 += '!';
cout << s2 << endl;          // 按 size 输出（长度 14）
cout << s2.c_str() << endl;  // 遇到第一个 '\0' 就停 → "hello world"

s2 += "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy";
cout << s2 << endl;          // 完整（含 '\0'）
cout << s2.c_str() << endl;  // 仍然截断在第一个 '\0'
```

> 结论：**`std::string` 可以存 `'\0'`，`size()` 是权威长度；`c_str()` 只适合"交给 C 接口"**。要输出全部内容，用 `cout << s`（按 size 输出）。

### 8.2 `operator>>`：自己实现（带缓冲，减少扩容）

```cpp
istream& operator>>(istream& in, string& s)
{
    s.clear();

    char buff[128];              // 128 字节小缓冲：先攒满再一次性 +=
    int i = 0;

    char ch = in.get();
    while (ch != ' ' && ch != '\n')
    {
        buff[i++] = ch;
        if (i == 127)            // 缓冲将满，批量倒进 string
        {
            buff[i] = '\0';
            s += buff;
            i = 0;
        }
        ch = in.get();
    }

    if (i > 0)                   // 处理剩余不足 128 的部分
    {
        buff[i] = '\0';
        s += buff;
    }
    return in;
}
```

**为什么要 128 字节缓冲？** 一个字符一个字符 `push_back` 会频繁触发扩容判断；**攒 128 个再 `append` 一次**，把扩容次数降到 1/128。

### 8.3 `getline`：只把结束符从空格改成 `delim`

```cpp
istream& getline(istream& in, string& s, char delim)
{
    s.clear();
    char buff[128];
    int i = 0;

    char ch = in.get();
    while (ch != delim)          // 唯一区别：分隔符可指定
    {
        buff[i++] = ch;
        if (i == 127) { buff[i] = '\0'; s += buff; i = 0; }
        ch = in.get();
    }
    if (i > 0) { buff[i] = '\0'; s += buff; }
    return in;
}
```

对应的使用习惯：

```cpp
string s;
cin >> s;              // 遇空格/换行停
getline(cin, s);       // 读一整行（吃掉换行）
getline(cin, s, '!');  // 读到 '!' 为止

while (cin >> s) { }   // 循环读直到 EOF（VS 里 Ctrl+Z 手动结束）
```

> 小技巧：读完一行里"最后一个单词"的长度

```cpp
getline(cin, s);
size_t pos = s.rfind(' ');
if (pos != string::npos) cout << s.size() - pos - 1 << endl;
else                     cout << s.size() << endl;
```

---

## 九、OJ 实战

### 9.1 387. 字符串中的第一个唯一字符

```cpp
class Solution {
public:
    int firstUniqChar(string s) {
        int count[26] = { 0 };
        // 统计次数
        for (auto ch : s)
            count[ch - 'a']++;

        for (size_t i = 0; i < s.size(); ++i)
            if (count[s[i] - 'a'] == 1)
                return i;

        return -1;
    }
};
```

**映射表思想**：`'a' → 0`、`'z' → 26`，用数组当下标映射，O(N)。

### 9.2 125. 验证回文串（`isPalindrome`）

```cpp
class Solution {
public:
    bool isLetterOrNumber(char ch)
    {
        return (ch >= '0' && ch <= '9')
            || (ch >= 'a' && ch <= 'z')
            || (ch >= 'A' && ch <= 'Z');
    }

    bool isPalindrome(string s) {
        for (auto& ch : s)                 // 统一转大写（或统一转小写）
            if (ch >= 'a' && ch <= 'z')
                ch -= 32;

        size_t begin = 0, end = s.size() - 1;
        while (begin < end)
        {
            while (begin < end && !isLetterOrNumber(s[begin])) ++begin;
            while (begin < end && !isLetterOrNumber(s[end]))   --end;

            if (s[begin] != s[end]) return false;
            ++begin;
            --end;
        }
        return true;
    }
};
```

要点：**双指针 + 跳过非字母数字**；内层两个 `while` 必须带 `begin < end` 判断，否则全是标点时指针会互相越过。

### 9.3 415. 字符串相加（大数运算）

```cpp
class Solution {
public:
    string addStrings(string num1, string num2) {
        string str;
        str.reserve(max(num1.size(), num2.size()) + 1);   // 提前开好，少扩容

        int end1 = num1.size() - 1, end2 = num2.size() - 1;
        int carry = 0;                                     // 进位
        while (end1 >= 0 || end2 >= 0)
        {
            int x1 = end1 >= 0 ? num1[end1--] - '0' : 0;
            int x2 = end2 >= 0 ? num2[end2--] - '0' : 0;

            int val = x1 + x2 + carry;
            carry = val / 10;
            val = val % 10;

            // str.insert(0, 1, '0' + val);      // O(N^2)：每次头插都挪动数据
            str += ('0' + val);                  // O(1) 尾插
        }

        if (carry == 1)
            str += '1';

        reverse(str.begin(), str.end());          // 结果反转
        return str;
    }
};
```

**为什么不能用 `int` 存结果**：`4 byte` 的 `int` 只能装 10 位左右的十进制数；`8 byte` 的 `long long` 也只有 19 位。像 `"9999999999999999999999999" + "1"` 这种，**必须逐位模拟**。

**为什么别用 `insert(0, ...)`**：每次头插都要把已有数据整体后移，总复杂度 O(N²)；改成"尾插 + 最后 `reverse`"就是 O(N)。

> 例子：`"456" + "77" = "533"`；`"11111111456" + "77" = "11111111533"`；`"9999999999999999999999999" + "1" = "10000000000000000000000000"`（位数变了，所以最后还要处理最高位进位）。

### 9.4 917. 仅仅反转字母

```cpp
class Solution {
public:
    bool isLetter(char ch)
    {
        if (ch >= 'a' && ch <= 'z') return true;
        if (ch >= 'A' && ch <= 'Z') return true;
        return false;
    }

    string reverseOnlyLetters(string s) {
        size_t begin = 0, end = s.size() - 1;
        while (begin < end)
        {
            while (begin < end && !isLetter(s[begin])) ++begin;
            while (begin < end && !isLetter(s[end]))   --end;
            swap(s[begin], s[end]);
            ++begin;
            --end;
        }
        return s;
    }
};
```

**非字母位置不动，只交换字母**——双指针 + 跳过条件的又一次应用。

---

## 十、速查表与易错清单

### 10.1 容量接口一图流

| 接口 | 改 size | 改 capacity | 会缩容吗 |
| --- | --- | --- | --- |
| `reserve(n)` | ✘ | ✔（n > capacity 时） | ✘（标准不要求） |
| `resize(n, ch)` | ✔ | ✔（空间不够时） | ✘ |
| `clear()` | ✔（→0） | ✘ | ✘ |
| `shrink_to_fit()` | ✘ | ✔ | ✔（非约束请求） |
| `push_back` | ✔(+1) | ✔（不够时，1.5/2 倍） | ✘ |
| `erase` / `pop_back` | ✔(−) | ✘ | ✘ |

### 10.2 三种遍历的选择

| 场景 | 用哪个 |
| --- | --- |
| 需要下标做运算 | `for (size_t i = 0; i < s.size(); ++i)` + `s[i]` |
| 通用、要给算法用 | 迭代器 `[begin(), end())` |
| 简单遍历（可读性最好） | `for (auto& e : s)`（要改加 `&`） |
| 不会修改元素 | `for (const auto& e : s)`（省拷贝） |

### 10.3 模拟实现易错清单

1. **忘记 `+1` 放 `'\0'`**：`new char[_size + 1]`、`reserve(n)` 里也要 `new char[n + 1]`。
2. **`insert` 用 `size_t` 从 `_size` 往 0 减**：减到 `-1` 变天文数字 → 越界死循环。要么把 `end` 设为 `_size+1` 用 `while(end > pos)`，要么用 `int` 强转。
3. **`insert(pos, const char*)` 的边界**：终止条件是 `end > pos + len - 1`，不是 `end > pos`。
4. **`erase` 用了 `memcpy`**：源/目标重叠，必须 `memmove`。
5. **不写拷贝构造**：默认浅拷贝 → 两个对象指向同一块内存 → double free / 改一个动全部。
6. **赋值不判自赋值**：传统写法必须先 `if (this != &s)`；现代写法（传值 + swap）天然免疫。
7. **`append` 的扩容条件**：是 `_size + len > _capacity`；`push_back` 的才是 `_size >= _capacity`。
8. **`substr` 忘了 `reserve`**：`+=` 循环里反复扩容，白浪费。
9. **比较运算符全手写**：只写 `<` 和 `==`，其余用组合实现，避免不一致。
10. **`npos` 当成 `-1` 比较**：`pos != string::npos` 才是正确写法（`size_t` 与有符号数比较有坑）。
11. **忘了该有 `const` 版本的成员**：`begin/end/operator[]/find/substr` 都要有 const 版本，否则 `const string` 对象用不了。

### 10.4 复杂度速查

| 操作 | 复杂度 | 备注 |
| --- | --- | --- |
| `operator[]` / `at` | O(1) | |
| `push_back` / `pop_back` | 均摊 O(1) | 扩容那一次 O(N) |
| `append` / `+=` | 均摊 O(len) | |
| `insert` / `erase` / `replace` | O(N) | 数据挪动 |
| `find(char)` | O(N) | |
| `find(str)` | O(N*M)（`strstr`） | KMP 可优化到 O(N+M) |
| `substr` | O(len) | |
| `size` / `capacity` / `clear` | O(1) | |

---

> 下一篇进入 `vector`：`25年-05月-20日--string+ vector` 末尾已开始 `reserve` 扩容实验与 `insert/erase` 迭代器版本（`find(v.begin(), v.end(), x)` + `v.insert/erase(it)`），`25年-05月-22日--vector` / `25年-05月-25日--vector` 起正式手写 `vector.h`。
