---
title: vector
date: 2026-11-23 14:10:17
tags: [c++, vector, STL, 模拟实现, 迭代器失效]
description: 'C++ vector 全面笔记：三指针结构与扩容机制、reserve/resize/insert/erase、手写 vector（initializer_list、迭代器区间构造、int/size_t 重载、拷贝赋值 swap）、迭代器失效四种场景与实验、vector<string> 扩容时 memcpy 的浅拷贝陷阱、vector<vector> 与杨辉三角、emplace_back、SGI STL 源码对照'
---

> 内容来源：
> - `25年-05月-20日--string+ vector`：`reserve` 扩容实验、`insert/erase` 的迭代器版本、`find` + `insert`/`erase` 的配合（本篇从"vector 视角"重述）
> - `25年-05月-22日--vector`：`vector` 的第一版手写实现（三指针 + `memcpy`）、`push_back`/`reserve`/`operator[]`、`initializer_list` 与 `auto il = {…}` 的类型、`emplace_back` vs `push_back`、结构化绑定、`vector<vector<int>>` 实例化示意、力扣 118 杨辉三角、SGI STL `vector` 源码对照（`start/finish/end_of_storage`、`construct` 定位 new、`insert_aux`、`uninitialized_copy`）
> - `25年-05月-25日--vector`：完整版手写 `vector`（`= default`、`initializer_list` 构造、迭代器区间模板构造、`int`/`size_t` 两个 `n,val` 构造、拷贝构造、`swap` + 传值赋值、`resize`/`clear`/`pop_back`/`insert`/`erase`/`empty`）、**`memcpy` → 循环赋值解决 `vector<string>` 浅拷贝**、**迭代器失效**四种场景与 VS/g++ 实测差异、`v7(10, 1)` 重载歧义

## 目录

- **一、vector 是什么**
- **二、常用接口与三指针结构**
- **三、扩容机制：reserve 与增长倍数**
- **四、**⚠️**扩容时的浅拷贝陷阱：为什么不能用 `memcpy`**
- **五、手写 vector：从第一版到完整版**
- **六、迭代器失效（重点）**
- **七、vector\<vector\<int\>\> 与力扣实战**
- **八、push_back vs emplace_back、结构化绑定、泛型打印**
- **九、SGI STL 源码对照**
- **十、速查表与易错清单**

---

## 一、vector 是什么

`vector` 是和 `string` 同源的东西：**都是"连续空间 + 顺序表"**。区别只有两点：

| | `string` | `vector<T>` |
| --- | --- | --- |
| 元素类型 | 固定 `char` | 任意 `T`（模板） |
| 结尾 | 必须补 `'\0'` | 不需要 |
| 内部结构 | `_str + _size + _capacity` | `_start + _finish + _endofstorage`（三指针） |

> `vector` 的设计更"泛型"：**用三个指针代替"指针 + 两个整数"**，这样 `size()` 和 `capacity()` 都能用指针相减算出来，天然适配"只有迭代器"的容器风格。

`vector<int>` 也是**类模板**实例化的结果 —— 这和 `string` 是 `basic_string<char>` 的 typedef 是同一件事：

```cpp
template<class T>
class vector
{
private:
    T*     _a;
    size_t _size;
    size_t _capacity;
};

// 实例化后 ≈
class vector<int>
{
public:
    int& operator[](int i);
private:
    int*   _a;
    size_t _size;
    size_t _capacity;
};

// 嵌套实例化后 ≈
class vector<vector<int>>
{
public:
    vector<int>& operator[](int i);   // 注意：返回的是"对象"的引用，不是 int&
private:
    vector<int>* _a;
    size_t _size;
    size_t _capacity;
};
```

> **为什么 `vector<vector<int>>` 的 `operator[]` 返回 `vector<int>&`？** 因为它存的是"一整个 `vector<int>` 对象"（对象里再含三个指针），所以 `vv[i][j]` 是两次 `operator[]`：第一次返回内层对象引用，第二次才拿到 `int&`。

---

## 二、常用接口与三指针结构

### 2.1 三指针模型

```text
_start                _finish              _endofstorage
  ↓                     ↓                      ↓
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┘
  └──── size() ────┘
  └────────── capacity() ─────────┘
```

```cpp
size_t size() const     { return _finish - _start; }
size_t capacity() const { return _endofstorage - _start; }
bool   empty() const    { return _start == _finish; }
iterator begin()        { return _start; }
iterator end()          { return _finish; }
```

**迭代器就是原生指针**：`typedef T* iterator;` —— 所以 `vector` 的迭代器支持 `+`、`-`、`[]`、比较，是**随机访问迭代器**（`std::sort`、`std::reverse`、`lower_bound` 都能用）。

### 2.2 常用接口一览

| 分类 | 接口 |
| --- | --- |
| 构造 | `vector()`、`vector(n, val)`、`vector(first, last)`、`vector{…}`（`initializer_list`）、拷贝构造 |
| 容量 | `size()` `capacity()` `empty()` `reserve(n)` `resize(n, val)` `clear()` |
| 访问 | `operator[]` `front()` `back()` `data()` |
| 修改 | `push_back` `pop_back` `insert` `erase` `assign` `swap` |
| 迭代器 | `begin()` `end()` `rbegin()` `rend()` |

### 2.3 迭代器初始化 + find 配合

```cpp
vector<int> v1;                    // 空
v1.push_back(1);
vector<int> v2(10, 1);             // 10 个 1
vector<int> v3(v1.begin(), v1.end());   // 迭代器区间

for (size_t i = 0; i < v1.size(); i++)  cout << v1[i] << " ";   // 下标
for (auto e : v2)                       cout << e << " ";       // 范围 for
vector<int>::iterator it = v3.begin();  while (it != v3.end()) { cout << *it << " "; ++it; }
```

```cpp
// 扩容指数：reserve 100 之后 push_back 100 次，capacity 一次都不变
vector<int> v1;
v1.reserve(100);
size_t old = v1.capacity();
for (size_t i = 0; i < 100; i++) {
    v1.push_back(i);
    if (v1.capacity() != old) { cout << v1.capacity() << endl; old = v1.capacity(); }
}

// find 返回"指向元素的迭代器"，insert/erase 都按迭代器操作
v1.insert(v1.begin(), 1000);
v1.insert(v1.begin() + 3, 1000);
auto it = find(v1.begin(), v1.end(), 5);
if (it != v1.end()) v1.insert(it, 50);

v1.erase(v1.begin());                        // 删第一个
it = find(v1.begin(), v1.end(), x);
if (it != v1.end()) v1.erase(it);            // 删指定元素
```

> `find` 是**函数模板**，只依赖迭代器的 `!=`、`++`、`*`：
>
> ```cpp
> template<class InputIterator, class T>
> InputIterator find(InputIterator first, InputIterator last, const T& val)
> {
>     while (first != last) {
>         if (*first == val) return first;
>         ++first;
>     }
>     return last;
> }
> ```
>
> 这就是"算法与底层结构解耦"的价值——同一份 `find` 能查 `vector`、`list`、`string`。

---

## 三、扩容机制：reserve 与增长倍数

### 3.1 `reserve` 的正确写法

```cpp
void reserve(size_t n)
{
    if (n > capacity())
    {
        size_t old_size = size();          // ★ 必须先保存！
        T* tmp = new T[n];

        if (_start)                        // 旧空间存在才需要拷贝
        {
            for (size_t i = 0; i < old_size; i++)
                tmp[i] = _start[i];        // ★ 逐个赋值（深拷贝），不能用 memcpy
            delete[] _start;
        }

        _start = tmp;
        _finish = _start + old_size;       // ★ 用保存下来的 old_size
        _endofstorage = _start + n;
    }
}
```

### 3.2 一个经典的踩坑：`_finish = _start + size()`

如果按直觉写成：

```cpp
_start = tmp;
_finish = _start + size();     // ✘ 错！
```

`size()` 的实现是 `_finish - _start`，而此时 `_start` 已经指向**新空间**，`_finish` 还指向**旧空间**（而且已经被 `delete[]` 了）——两个指针相减是**不同数组之间的指针相减，未定义行为**，结果是个垃圾值（调试器里能看到 `_finish` 变成 `0x00000000` 附近的野值）。

> **正确做法：先把 `old_size = size()` 存下来**，改完 `_start` 之后再用 `_start + old_size`。这就是严格按"先算后改"的顺序写。

### 3.3 增长倍数

```cpp
void push_back(const T& x)
{
    if (_finish == _endofstorage)
        reserve(capacity() == 0 ? 4 : capacity() * 2);   // 首次开 4，之后 2 倍
    *_finish = x;
    ++_finish;
}
```

实测（与 `string` 一致）：

```text
VS2019（1.5 倍，除了第一次）        g++（2 倍）
15                                  15
16                                  30
31                                  60
47                                  120
70
105
```

> 标准只保证 `push_back` **均摊 `O(1)`**，倍数由实现决定。**1.5 倍更省内存（有机会原地复用），2 倍在时间上更快。**

---

## 四、⚠️ 扩容时的浅拷贝陷阱：为什么不能用 `memcpy`

这是 5-22 → 5-25 之间最大的改动，也是面试高频题。

### 4.1 出问题的写法（5-22 第一版）

```cpp
void reserve(size_t n)
{
    if (n > capacity())
    {
        size_t old_size = size();
        T* tmp = new T[n];
        if (_start)
        {
            memcpy(tmp, _start, sizeof(T) * old_size);   // ✘ 对 string 是浅拷贝
            delete[] _start;
        }
        _start = tmp;
        _finish = _start + old_size;
        _endofstorage = _start + n;
    }
}
```

对 `vector<int>`、`vector<double>` 这类 **POD（平凡可拷贝）类型**，`memcpy` 按字节搬运是安全的（甚至连"对象语义"都不需要）。

但对 `vector<string>` 就出事了：

```cpp
void test_vector8()
{
    bit::vector<string> v1;
    v1.push_back("11111111111111111111111");   // 长串 → 内部 new 了堆空间
    v1.push_back("11111111111111111111111");
    v1.push_back("11111111111111111111111");
    v1.push_back("11111111111111111111111");
    v1.push_back("11111111111111111111111");   // 第 5 个触发扩容！
    Print(v1);
}
```

**`memcpy` 只是把 `string` 对象里的 `_str` 指针按字节复制过去**，新旧两个 `string` 对象指向**同一块堆内存**：

```text
旧空间 _start ──► [str:{_str→堆A}] [str:{_str→堆B}] ...
                        ╎                ╎
                  memcpy（只复制指针）
                        ╎                ╎
新空间 tmp   ──► [str:{_str→堆A}] [str:{_str→堆B}] ...
                        ↓
              delete[] _start;   // 旧 string 对象析构 → 把堆A、堆B 释放了
                        ↓
              新空间的 _str 全部悬垂 → 再次析构时 double free / 崩溃
```

### 4.2 正确写法：逐个赋值（调用 `operator=` 完成深拷贝）

```cpp
for (size_t i = 0; i < old_size; i++)
    tmp[i] = _start[i];        // ✔ 调用 T::operator=，自己管好自己的资源
```

板书总结得很清楚：

> 1. `int` 这样类型，赋值可以完成浅拷贝 —— 所以用 `memcpy` 也没问题
> 2. `string` 这样的类型，赋值调用 `operator=` 才能完成**深拷贝** —— 所以**必须逐个赋值**

### 4.3 更严谨的说法

`new T[n]` + 逐个赋值其实还有个瑕疵：**`new T[n]` 会先默认构造 n 个对象**，再被赋值覆盖一次，多一次构造/析构开销。

标准库的做法是"**未初始化内存 + 定位 new**"：

```cpp
// SGI STL 的做法（示意）
iterator new_start = data_allocator::allocate(len);          // 只要原始内存
new_finish = uninitialized_copy(start, position, new_start);// 逐元素拷贝构造
construct(new_finish, x);                                    // 定位 new 构造新元素
new_finish = uninitialized_copy(position, finish, new_finish);
// ...
destroy(begin(), end());      // 析构旧元素
deallocate();                 // 释放旧内存
start = new_start;
finish = new_finish;
end_of_storage = new_start + len;
```

> `uninitialized_*` 系列 = **"在原始内存上拷贝构造"**，跳过"先默认构造再赋值"的浪费。这才是 `vector` 能做到"支持不可默认构造类型"的原因。

---

## 五、手写 vector：从第一版到完整版

### 5.1 头文件骨架

```cpp
#pragma once
#include <assert.h>

// 模板的声明和定义不能分离到两个文件（.h / .cpp）
// 否则链接时找不到实例化后的函数 → 链接错误

namespace bit
{
    template<class T>
    class vector
    {
    public:
        typedef T*       iterator;
        typedef const T* const_iterator;

        iterator       begin()       { return _start; }
        iterator       end()         { return _finish; }
        const_iterator begin() const { return _start; }
        const_iterator end()   const { return _finish; }

        vector() = default;                          // 强制生成默认构造
        vector(initializer_list<T> il);              // vector<int> v = {1,2,3};
        vector(const vector<T>& v);                  // 拷贝构造
        template <class InputIterator>
        vector(InputIterator first, InputIterator last);   // 迭代器区间
        vector(int n, T val = T());                  // 兼容字面量 int
        vector(size_t n, T val = T());               // 无符号版本
        ~vector();

        vector<T>& operator=(vector<T> v);           // 传值 + swap
        void swap(vector<T>& v);

        void reserve(size_t n);
        void resize(size_t n, T val = T());
        void clear();
        bool empty() const;

        T&       operator[](size_t i);
        const T& operator[](size_t i) const;

        void push_back(const T& x);
        void pop_back();
        iterator insert(iterator pos, const T& x);
        iterator erase(iterator pos);

        size_t size() const     { return _finish - _start; }
        size_t capacity() const { return _endofstorage - _start; }

    private:
        iterator _start = nullptr;
        iterator _finish = nullptr;
        iterator _endofstorage = nullptr;
    };
}
```

> **`vector() = default;`** 的作用：成员已经给了缺省值（`= nullptr`），编译器生成的默认构造就够用；但一旦我们自己写了别的构造函数（比如带参构造），编译器就不再生成默认构造。写 `= default` 是**显式要求编译器生成**，比手写空函数体更清晰、且保持 trivial。

### 5.2 两个"带 n 和 val"的构造函数：一个都不能少的理由

```cpp
vector(size_t n, T val = T()) { resize(n, val); }
vector(int    n, T val = T()) { resize(n, val); }   // ★ 为什么还要 int 版？
```

因为迭代器区间的构造函数是**函数模板**，而模板在重载决议中**优先级更高（更匹配）**：

```cpp
template <class InputIterator>
vector(InputIterator first, InputIterator last) { ... }

bit::vector<int> v7(10, 1);       // 若只有 size_t 版：10 和 1 都是 int
                                  // → 更匹配 template<InputIterator>！
                                  // → 实例化成 vector(int, int)，把 "10" 当成迭代器 → 编译报错
bit::vector<size_t> v8(10u, 1u);  // size_t 版能匹配，OK
// bit::vector<size_t> v9(10, 1); // int 实参 → template 更匹配 → 编译不过
```

> 板书的结论：**`vector<int> v(10, 1)` 会去匹配"更匹配"的函数模板**，把 `10` 当迭代器、`1` 当哨兵，得到 `10 != 1` 恒真、`*10` 直接非法 → 报错。
> 所以要么额外写一个 `int` 版本（课上做法），要么在调用处写 `10u`。**设计上并不完美，但这是踩过坑之后的标准妥协。**

### 5.3 各种构造

```cpp
// 迭代器区间（可以是任意容器的迭代器，也能顺带做类型转换）
template <class InputIterator>
vector(InputIterator first, InputIterator last)
{
    while (first != last)
    {
        push_back(*first);       // char → int 自动转换
        ++first;
    }
}

// initializer_list
vector(initializer_list<T> il)
{
    reserve(il.size());          // 一次开够，避免边插边扩
    for (auto& e : il)
        push_back(e);
}

// 拷贝构造：按被拷贝者的"容量"申请，保持容量一致
vector(const vector<T>& v)
{
    reserve(v.capacity());
    for (auto& e : v)
        push_back(e);
}
```

实测：

```cpp
bit::vector<int> v2(v1);                            // 拷贝构造
bit::vector<int> v3 = {1,2,3,4,5,5};                // initializer_list
bit::vector<int> v4 = {1,2,3,4,5,51,1,1,1,1,1,1,1};
bit::vector<int> v5(v3.begin() + 1, v3.end() - 1);  // 迭代器区间
string s1("hello world");
bit::vector<int> v6(s1.begin(), s1.end());          // string 迭代器 → vector<int>（char 转 int）
bit::vector<int> v7(10, 1);                         // int 版 n,val
bit::vector<size_t> v8(10u, 1u);                    // size_t 版
v1 = v7;                                            // 赋值
```

> `bit::vector<int> v6(s1.begin(), s1.end())` 打印出来是 `104 101 108 108 111 …`——正好是 `'h' 'e' 'l' 'l' 'o'` 的 ASCII。**迭代器区间构造天然支持"任意可迭代源 + 目标类型可转换"**。

### 5.4 拷贝、赋值、`swap`

```cpp
void swap(vector<T>& v)
{
    std::swap(_start, v._start);
    std::swap(_finish, v._finish);
    std::swap(_endofstorage, v._endofstorage);
}

// v1 = v7：参数传值 → tmp 已是 v7 的深拷贝 → 直接换 → tmp 析构释放 v1 原来的空间
vector<T>& operator=(vector<T> v)
{
    swap(v);
    return *this;
}
```

**现代写法三件套（和 `string` 完全一致）：**
1. 不用判自赋值（`v1 = v1` 时形参是独立副本）。
2. 天然异常安全（拷贝失败发生在传参阶段，`*this` 未动）。
3. 顺带把 `v7`（左值）和临时对象（右值）都覆盖了。

### 5.5 增删改

```cpp
void push_back(const T& x)
{
    if (_finish == _endofstorage)
        reserve(capacity() == 0 ? 4 : capacity() * 2);
    *_finish = x;
    ++_finish;
}

void pop_back()
{
    assert(!empty());
    --_finish;                  // 注意：不是 "== nullptr"，判空用 empty()
}

void resize(size_t n, T val = T())
{
    if (n > size())
    {
        reserve(n);
        while (_finish != _start + n)
        {
            *_finish = val;
            ++_finish;
        }
    }
    else
    {
        _finish = _start + n;   // 只挪 finish，不释放空间
    }
}

void clear() { _finish = _start; }   // size 归零，capacity 不变
```

**`insert`（两件事最容易漏：更新 `pos`、返回 `pos`）**

```cpp
iterator insert(iterator pos, const T& x)
{
    assert(pos >= _start);
    assert(pos <= _finish);

    if (_finish == _endofstorage)
    {
        size_t len = pos - _start;                          // ★ 先记下偏移
        reserve(capacity() == 0 ? 4 : capacity() * 2);
        pos = _start + len;                                 // ★ 扩容后重建 pos！
    }

    iterator end = _finish - 1;
    while (end >= pos)          // 从后往前挪
    {
        *(end + 1) = *end;
        --end;
    }
    *pos = x;
    ++_finish;

    return pos;                 // ★ 返回新元素的迭代器
}
```

> ⚠️ **隐藏坑**：`while (end >= pos)` 在 `pos == _start` 时，`end` 会减到 `_start - 1`——**指向数组之前的位置，属于未定义行为**（虽然绝大多数编译器"碰巧"能停）。稳妥写法与 `string::insert` 一致：**从 `_finish` 开始、用 `end > pos` 判定**：
>
> ```cpp
> iterator end = _finish;
> while (end > pos)
> {
>     *end = *(end - 1);
>     --end;
> }
> ```

**`erase`（返回删除位置的迭代器，供循环使用）**

```cpp
iterator erase(iterator pos)
{
    assert(pos >= _start);
    assert(pos < _finish);

    iterator it = pos + 1;
    while (it != _finish)       // 从前往后覆盖
    {
        *(it - 1) = *it;
        ++it;
    }

    --_finish;
    return pos;                 // ★ 必须返回
}
```

**析构**

```cpp
~vector()
{
    if (_start)
    {
        delete[] _start;
        _start = _finish = _endofstorage = nullptr;
    }
}
```

---

## 六、迭代器失效（重点）

**迭代器的本质是"指向容器内部某处的指针"**，一旦那块内存被释放、或元素整体搬家，这个指针就悬垂了 → 继续用就是**未定义行为**。

### 6.1 四种失效场景

| 场景 | 是否一定失效 | 原因 |
| --- | --- | --- |
| **`reserve` / `push_back` 触发扩容** | 一定失效 | 旧空间被 `delete[]`，所有迭代器、指针、引用全悬垂 |
| **`insert` 触发扩容** | 一定失效，且**传入的 `pos` 本身也失效** | 同上，必须用 `pos = _start + len` 重建 |
| **`insert` 未扩容** | `pos` 及其之后失效 | 元素整体后移，原来位置的值变了 |
| **`erase`** | `pos` 及其之后失效 | 元素前移覆盖，`pos` 本身也被覆盖了 |

### 6.2 实验一：`insert` 之后还能用 `it` 吗？

```cpp
auto it = find(v.begin(), v.end(), x);
if (it != v.end())
{
    v.insert(it, x * 10);
    // it 失效？这里 it 能否继续使用？→ 不能
    // 扩容会导致 it 失效，但 C++ 标准没有规定 vector 的扩容规则，
    // 所以这里我们认为 insert 之后 it 就失效了，不能再使用 it
    // *it = 1000;
}
```

> **关键推理**：标准没规定"什么时候扩容"，所以**即使这次没扩容，也不能假定以后不扩容**。工程上的安全约定是：**`insert` 之后传入的 `pos` 一律作废**（除非你使用 `insert` 的返回值）。

### 6.3 实验二：`erase` 之后还能用 `it` 吗？

```cpp
auto it = find(v.begin(), v.end(), x);
if (it != v.end())
{
    v.erase(it);
    *it = 1000;        // ✘ 未定义行为
}
```

板书给了 VS2019 与 g++ 的**实测差异**：

| 环境 | 结果 |
| --- | --- |
| VS2019（Debug） | 进程直接退出，代码 `-1073741819`（`0xC0000005` 访问冲突）——**编译器对失效迭代器做了严格检查** |
| g++ `-O0`/release | 输出看起来"正常"（`1 3 4 5` → `1 3 5 4`）——**碰巧那块内存还没被复用** |
| g++（某些情况） | `segmentation fault` |

> 结论：**"跑得通"不代表"写对了"**。VS 对 `vector` 失效迭代器的访问有 `_ITERATOR_DEBUG_LEVEL` 级别的严格校验，g++ 默认不管。同一份代码两种表现，正是 UB 的典型特征。

### 6.4 正确姿势：用返回值更新迭代器

```cpp
// 要求删除所有偶数
auto it = v.begin();
while (it != v.end())
{
    if (*it % 2 == 0)
        it = v.erase(it);   // ✔ erase 返回"被删位置"的新迭代器
    else
        ++it;
}
Print(v);
```

```text
原：1 2 3 4 4 5 4
删：1 3 4 5 4      ← 注意这里只删掉了"第一个偶数序列"能删的部分吗？不，是全部命中
```

> 记忆口诀：**`erase` 之后必须 `it = v.erase(it)`；`insert` 之后必须用 `v.insert(...)` 的返回值，原来的 `it` 一律不要。**
>
> 这也解释了标准库几乎每个修改接口都**返回迭代器**的原因——**返回值的意义就是"给你一个可用的新迭代器"**。

---

## 七、`vector<vector<int>>` 与力扣实战

### 7.1 用 `malloc` 手开二维数组 vs 用 `vector`

```cpp
/**
 * Return an array of arrays of size *returnSize.
 * The sizes of the arrays are returned as *returnColumnSizes array.
 * Note: Both returned array and *columnSizes array must be malloced, assume caller calls free().
 */
int** generate(int numRows, int* returnSize, int** returnColumnSizes)
{
    int** aa = (int**)malloc(sizeof(int*) * numRows);   // 指针数组
    for (int i = 0; i < numRows; ++i)
        aa[i] = (int*)malloc(sizeof(int) * (i + 1));    // 每个一维数组单独开

    *returnSize = numRows;
    *returnColumnSizes = malloc(numRows * sizeof(int)); // 每一行多长
    ...
}
```

```text
aa ──► [ int* ][ int* ][ int* ][ int* ]   ← 指针数组
          ↓      ↓      ↓      ↓
        [1]    [1 1]  [1 2 1][1 3 3 1]     ← 每个一维数组长度不同
```

> 这是 C 风格"二维数组"的真实结构：**不是连续的一整块，而是"指针数组 + 一堆小数组"**。用 `vector` 写，就是 `vector<vector<int>>`——**结构一模一样，但不用手动 `malloc`/`free`、长度自己记、越界有断言**。

### 7.2 118. 杨辉三角

```cpp
class Solution {
public:
    vector<vector<int>> generate(int numRows) {
        vector<vector<int>> vv;
        vv.resize(numRows, vector<int>());     // 先开 numRows 个空 vector
        for (size_t i = 0; i < numRows; ++i)
        {
            vv[i].resize(i + 1, 1);            // 每行长度 i+1，全填 1（首尾天然是 1）
        }

        for (size_t i = 2; i < vv.size(); ++i)
        {
            for (size_t j = 1; j < vv[i].size() - 1; ++j)
            {
                vv[i][j] = vv[i-1][j] + vv[i-1][j-1];   // 左肩 + 右肩
            }
        }
        return vv;
    }
};
```

**`vv.resize(numRows, vector<int>())` 与 `vector<vector<int>> vv(numRows)` 的区别：**

```cpp
vector<vector<int>> vv(numRows);                    // 直接开 n 个（值初始化的）空 vector
vv.resize(numRows, vector<int>());                  // 同上，但更"显式"
```

> 板书标注了这两种写法的等价关系。要点是：**`resize` 的第二个参数是"每个新元素用什么填"**，对 `vector<vector<int>>` 就是一个默认构造的空 `vector<int>`。

### 7.3 二叉树遍历：返回 `vector` 而不是 `int*` 指针

```cpp
// C 风格：还要 malloc + 传出 returnSize
int* preorderTraversal(struct TreeNode* root, int* returnSize);

// C++ 风格：直接返回 vector
vector<int> preorderTraversal(TreeNode* root);
```

```cpp
vector<vector<int>> levelOrder(TreeNode* root);   // 层序遍历：每层一个 vector
```

> 这就是"返回容器"的价值：**长度信息随数据一起返回，不需要额外的 `returnSize` 出参，更不需要调用方 `free`。**（注意 `vector` 支持移动/RVO，返回并不慢。）

---

## 八、`push_back` vs `emplace_back`、结构化绑定、泛型打印

### 8.1 `initializer_list` 与 `auto il = {…}`

```cpp
int a[] = { 10, 20, 30 };              // 原生数组
auto il1 = { 10, 20, 30 };             // std::initializer_list<int>
auto il2 = { 10, 20, 30, 1,1,1,1,1 };  // 同上，元素个数可变
cout << typeid(il1).name() << endl;    // 类型名是 initializer_list<int>
cout << il1.begin() << endl;
cout << a << endl;                     // 数组名退化为指针

for (auto e : il1) cout << e << " ";   // 可以范围 for
```

```cpp
vector<int> v1{ 10, 20, 30 };          // 花括号 → initializer_list 构造
vector<int> v2 = { 10, 20, 30 };       // 拷贝列表初始化，效果相同
```

> `{1,2,3}` 的**"推导类型"**：单独写 `auto il = {1,2,3}` 时它就是 `initializer_list`；而 `vector<int> v{1,2,3}` 能用它，就是因为 `vector` 提供了 `initializer_list` 构造函数。

### 8.2 `push_back` vs `emplace_back`

```cpp
struct A
{
    A(int a1, int a2) : _a1(a1), _a2(a2) { cout << "A(int a1, int a2)" << endl; }
    A(const A& aa) : _a1(aa._a1), _a2(aa._a2) { cout << "A(const A& aa)" << endl; }
    int _a1, _a2;
};

vector<A> v2;
A aa1(1, 1);

v2.push_back(aa1);        // 拷贝构造
v2.push_back(A(2, 2));    // 构造 + 移动/拷贝
v2.push_back({3, 3});     // 隐式转换构造 + 拷贝

v2.emplace_back(aa1);     // 拷贝构造
v2.emplace_back(A(2, 2)); // 构造 + 移动
// v2.emplace_back({3, 3});   // ✘ 花括号没有类型，无法推导
v2.emplace_back(3, 3);    // ✔ 直接把构造参数传进去，效率最高（只构造一次）
```

| | 接口 | 本质 |
| --- | --- | --- |
| `push_back(x)` | 参数是**已构造好的对象** | 先造对象 → 再拷/移进去 |
| `emplace_back(args…)` | 参数是**构造该对象的参数包** | 直接在容器内存上**定位 new** 构造 → 省一次拷/移 |

> 结论：**能用 `emplace_back` 传参就用它**（`v.emplace_back(3, 3)`）；但 `emplace_back({3,3})` 不合法，因为 `{}` 没有类型，模板无法推导。

### 8.3 三种遍历方式 + C++17 结构化绑定

```cpp
// 1. 迭代器
vector<A>::iterator it2 = v2.begin();
while (it2 != v2.end())
{
    cout << it2->_a1 << ":" << it2->_a2 << endl;   // 迭代器是 A*，可以用 ->
    ++it2;
}

// 2. 范围 for
for (auto& aa : v2)
    cout << aa._a1 << ":" << aa._a2 << endl;

// 3. C++17 结构化绑定
for (auto& [x, y] : v2)
    cout << x << ":" << y << endl;
```

> `it2->_a1` 与 `(*it2)._a1` 完全等价——**因为 `vector` 的迭代器就是 `A*`**。这也侧面证明了"迭代器是指针一样的类型对象"（对 `list` 就得写 `it->` 且它是重载的 `operator->`）。

### 8.4 泛型打印函数（函数模板）

```cpp
template<class Container>
void Print(const Container& con)
{
    for (auto e : con)
        cout << e << " ";
    cout << endl;
}
```

**一份代码，通吃 `bit::vector<int>`、`std::vector<int>`、`string`、数组……** 只要求容器有 `begin()` / `end()`（能被范围 for 用）。

> 注意形参是 **`const Container&`**：既省拷贝，又迫使范围 for 走 `const_iterator`（验证 `const` 版 `begin/end` 有没有写对）。这就把 5-25 新增的 `const_iterator` 版本"用起来了"。

---

## 九、SGI STL 源码对照

课上对照了 SGI 版（`stl30` 源码），看的是"标准实现长什么样"。

### 9.1 类结构与成员

```cpp
template <class T, class Alloc = alloc>
class vector {
public:
    typedef T value_type;
    typedef value_type* iterator;

    // 成员变量
    iterator start;             // 目前使用空间的头
    iterator finish;            // 目前使用空间的尾
    iterator end_of_storage;    // 目前可用空间的尾
    ...
    iterator begin() { return start; }
    iterator end()   { return finish; }
};
```

> 标准 SGI 版**没有构造函数初始化列表**吗？有：`vector() : start(0), finish(0), end_of_storage(0) {}`（板书标注"构造初始化"）。

### 9.2 `push_back` 与 `insert_aux`

```cpp
void push_back(const T& x) {
    if (finish != end_of_storage) {     // 还有备用空间
        construct(finish, x);           // 定位 new：在 finish 处构造
        ++finish;
    }
    else
        insert_aux(end(), x);           // 没空间了：扩容 + 插入
}
```

```cpp
template <class T, class Alloc>
void vector<T, Alloc>::insert_aux(iterator position, const T& x) {
    if (finish != end_of_storage) {
        construct(finish, *(finish - 1));            // 先在尾部构造一个副本（补齐要塞位的空位）
        ++finish;
        T x_copy = x;
        copy_backward(position, finish - 2, finish - 1);   // 后移
        *position = x_copy;
    }
    else {
        const size_type old_size = size();
        const size_type len = old_size != 0 ? 2 * old_size : 1;   // 2 倍扩容
        iterator new_start = data_allocator::allocate(len);       // 只分配原始内存
        iterator new_finish = new_start;

        #ifdef __STL_TRY
        __STL_TRY {
            new_finish = uninitialized_copy(start, position, new_start);  // 拷贝构造前半段
            construct(new_finish, x);                                     // 构造新元素
            ++new_finish;
            new_finish = uninitialized_copy(position, finish, new_finish);// 拷贝构造后半段
        }
        catch(...) {
            destroy(new_start, new_finish);
            data_allocator::deallocate(new_start, len);
            throw;
        }
        #endif

        destroy(begin(), end());        // 析构旧元素
        deallocate();                   // 释放旧内存
        start = new_start;
        finish = new_finish;
        end_of_storage = new_start + len;
    }
}
```

`construct` 的实现就是**定位 new**：

```cpp
template <class T1, class T2>
inline void construct(T1* p, const T2& value) {
    new (p) T1(value);      // placement new：在已分配的原始内存上调用构造函数
}
```

**四个关键设计，对比我们的玩具版：**

| 点 | SGI STL | 手写版 |
| --- | --- | --- |
| 扩容 | `allocate`（原始内存）+ `uninitialized_copy`（拷贝构造） | `new T[n]`（默认构造）+ 逐个赋值 |
| 异常安全 | `__STL_TRY` / `catch` → 失败时 `destroy` + `deallocate` 再 `throw` | 无（`new` 抛异常时旧数据可能泄漏） |
| 元素个数为 0 | `len = old_size != 0 ? 2 * old_size : 1` | `capacity() == 0 ? 4 : capacity() * 2` |
| 空间配置 | `data_allocator`（内存池） | 直接 `new[]` |

> `uninitialized_copy` 的价值：**"在原始内存上拷贝构造"**。因为 `new T[n]` 会先把 n 个对象默认构造一遍再被覆盖——对 `string` 这类类型就是白白构造+析构 n 次。**这就是"未初始化的内存 + 定位 new"这个套路的动机。**

---

## 十、速查表与易错清单

### 10.1 手写 vector 的接口全表

| 接口 | 关键实现 | 易错点 |
| --- | --- | --- |
| `vector() = default` | 成员已给缺省值 | 别手写空函数体（会变成非 trivial） |
| `vector(il)` | `reserve(il.size())` + 逐个 `push_back` | 忘了 `reserve` → 反复扩容 |
| `vector(first, last)` | `while (first != last) push_back(*first)` | 必须是**函数模板**才能接任意迭代器 |
| `vector(n, val)` | `resize(n, val)` | `int` / `size_t` 两个版本都要有 |
| 拷贝构造 | `reserve(v.capacity())` + 逐个 `push_back` | 别 `memcpy` |
| `operator=(vector v)` | 传值 + `swap` | 别写成 `const vector&`（丢掉了现代写法的好处） |
| `reserve(n)` | `old_size` 先存 + 逐个赋值 | ❌ `_finish = _start + size()` |
| `resize(n, val)` | 变大填 `val`，变小只挪 `_finish` | 不释放空间 |
| `clear()` | `_finish = _start` | 不改容量 |
| `insert(pos, x)` | 扩容后 `pos = _start + len`；返回 `pos` | ❌ 用旧 `pos`；❌ 不返回 |
| `erase(pos)` | 前移覆盖 + `--_finish`；返回 `pos` | ❌ 调用方不用返回值 |
| `pop_back()` | `assert(!empty())` + `--_finish` | 空容器 `pop_back` 是 UB |

### 10.2 与 `string` 的对照

| 点 | `string` | `vector<T>` |
| --- | --- | --- |
| 结构 | `char* + size + capacity` | `start + finish + end_ofstorage` |
| 迭代器 | `char*` | `T*` |
| 结尾 | 必须写 `'\0'` | 不需要 |
| 扩容拷贝 | `memcpy` 安全（char 是 POD） | **必须逐个赋值**（`T` 可能是 `string`） |
| 元素个数 | `size()` | `size()` |
| 插入/删除 | `O(N)` 挪动 | `O(N)` 挪动 |

### 10.3 易错清单

1. **`reserve` 里 `_finish = _start + size()`**：`size()` 依赖 `_start`，改完就成了跨数组指针相减 → UB。**必须先存 `old_size`。**
2. **`memcpy` 搬运非 POD 元素**：`vector<string>` 扩容时是浅拷贝 → 悬垂 + double free。**必须 `tmp[i] = _start[i]`。**
3. **`insert` 扩容后不重建 `pos`**：`pos` 指向已释放的旧空间。**用 `pos = _start + len`。**
4. **`insert`/`erase` 不返回迭代器**：调用方没法在循环里安全地继续。**两者都必须返回 `pos`。**
5. **`erase` 后继续用旧迭代器**：`it = v.erase(it)`。VS 会直接崩，g++ 可能"看起来正常"——**这是 UB，不是"能用"**。
6. **`insert` 后继续用 `pos`**：标准没规定扩容规则，所以**一律认为失效**。
7. **`vector<int> v(10, 1)` 只提供 `size_t` 版构造**：实参是 `int` 时函数模板更匹配 → 把 `10` 当迭代器 → 编译报错。加一个 `int` 版，或写成 `v(10u, 1u)`。
8. **忘写 `const` 版 `begin/end`/`operator[]`**：`const vector` 用不了，`Print(const Container&)` 编不过。
9. **模板的声明与定义分到 `.h`/`.cpp`**：链接期找不到实例化 → `unresolved external symbol`。**模板全放头文件。**
10. **`emplace_back({3,3})`**：`{}` 无类型，模板无法推导。要写 `emplace_back(3, 3)`。
11. **空 `vector` 调 `pop_back()`**：UB，先 `assert(!empty())`。

### 10.4 复杂度

| 操作 | 复杂度 | 备注 |
| --- | --- | --- |
| `operator[]` / `at` | `O(1)` | 随机访问 |
| `push_back` / `pop_back` | 均摊 `O(1)` | 扩容那一次 `O(N)` |
| `insert` / `erase` | `O(N)` | 数据挪动 |
| `reserve` / `resize` | `O(N)` | |
| `size` / `capacity` / `empty` / `clear` | `O(1)` | |
| `find`（`std::find`） | `O(N)` | 不是二分 |

---

> 下一篇进入 `list`：`25年-05月-27日--list` / `25年-05月-29日--list` 开始手写带头双向循环链表 `list.h`。
> 那时会看到"迭代器不再是指针"——`list` 的迭代器必须**封装节点指针**并重载 `++/--/->/*`，而 `vector` 的"原地挪动 + 扩容搬家"整套问题（迭代器失效、`memcpy`、`reserve`）在 `list` 里全部消失，代价是**丧失随机访问**。
