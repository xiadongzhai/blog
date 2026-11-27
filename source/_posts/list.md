---
title: list
date: 2026-11-26 20:56:48
tags: [c++, list, STL, 模拟实现, 迭代器, 缓存]
description: 'C++ list 笔记：带头双向循环链表的结构、迭代器必须封装节点指针、const_iterator 的三种实现与模板参数 Ref/Ptr 正解、operator-> 语法糖、insert/erase 复用、list 专有操作（sort/merge/remove/splice）、迭代器分类与算法对迭代器的要求、CPU 三级缓存与 vector 排序更快的实测原理'
---

> 内容来源：
> - `25年-05月-27日--list`：`std::list` 的构造与遍历、迭代器分类与各算法对迭代器的要求、`list` 的专有成员函数（`sort`/`merge`/`remove`/`splice`/`assign`/`resize`）、**`vector` 与 `list` 排序性能实测**、CPU 三级缓存与缓存命中率、手写 `list` 第一版（`list_node` + `__list_iterator` + 带头循环链表 + `push_back`）、迭代器必须"封装 `Node*`"、`vector` 的 `int`/`size_t` 重载与 `enable_if_t` 的 SFINAE 解法
> - `25年-05月-29日--list`：完整版手写 `list`（`iterator`/`const_iterator` 两种实现路线对比、模板参数 `Ref`/`Ptr` 的正解、`operator->` 语法糖、`insert`/`erase`、`push_front`/`pop_front`、`clear`、拷贝构造、`initializer_list` 构造、`operator=`（传值 + `swap`）、`size()`、`typename` 的必要性）
>
> 参考链接（板书附带）：[CPU 缓存那些事（耗子叔 coolshell）](https://coolshell.cn/articles/20793.html)

## 目录

- **一、list 是什么：带头双向循环链表**
- **二、std::list 的用法与 vector 的对比**
- **三、迭代器分类：算法对容器的要求**
- **四、为什么 vector 排序更快：CPU 三级缓存**
- **五、手写 list（一）：节点、迭代器、骨架**
- **六、手写 list（二）：const_iterator 的三种做法**
- **七、手写 list（三）：`operator->` 与 `typename`**
- **八、手写 list（四）：增删改与拷贝控制**
- **九、速查表与易错清单**

---

## 一、list 是什么：带头双向循环链表

`std::list` 是**带头（带哨兵位）的双向循环链表**：

```text
        ┌──────────────────────────────────────────┐
        │                                          │
        ↓                                          │
   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
   │  _head  │←──→│  1      │←──→│  2      │←──→│  3      │
   │ (哨兵位) │───→│         │───→│         │───→│         │
   └─────────┘    └─────────┘    └─────────┘    └─────────┘
        ↑                                          │
        └──────────────────────────────────────────┘
     end()                                       最后一个节点
```

每个节点长这样：

```cpp
template<class T>
struct list_node
{
    T _data;
    list_node<T>* _next;
    list_node<T>* _prev;

    list_node(const T& x = T())
        :_data(x)
        ,_next(nullptr)
        ,_prev(nullptr)
    {}
};
```

**哨兵位 `_head` 的三个作用**（这样设计的好处）：

1. **不需要单独处理"空链表"**：空链表时 `_head->_next == _head->_prev == _head`，插入删除逻辑完全统一。
2. **`end()` 就是 `_head`**：循环一圈刚好回到哨兵位，`while (it != lt.end())` 天然成立。
3. **头插/尾插都是"在某个位置前插入"**：`push_back` = `insert(end(), x)`，`push_front` = `insert(begin(), x)`。

| 位置 | 对应节点 |
| --- | --- |
| `begin()` | `_head->_next`（第一个真实节点） |
| `end()` | `_head`（哨兵位本身，不是尾节点） |
| 尾节点 | `_head->_prev` |
| 空链表 | `begin() == end() == _head` |

> 板书的那个比喻（了解）：`list` 是"双非，需要 double 努力"——因为它的迭代器必须手写一整个类；`vector` 是"985/211，家庭背景和社会关系丰富"——它的迭代器直接就是原生指针 `T*`，白送一堆能力。**这是"链式结构"与"连续结构"的根本差别。**

---

## 二、std::list 的用法与 vector 的对比

### 2.1 各种构造

```cpp
void test_list1()
{
    list<int> lt1;                              // 空
    list<int> lt2(10, 1);                       // 10 个 1
    vector<int> v1 = { 1,2,3,4,5,6 };
    list<int> lt3(v1.begin(), v1.end());        // 迭代器区间（跨容器！）
    list<int> lt4 = { 1,2,3,4,5,6,1,1,1,1 };    // initializer_list

    int a[] = { 1,20,3,40 };
    list<int> lt5(a, a + 4);                    // 原生数组区间
}
```

迭代器区间构造是个**函数模板**，所以可以"任意容器 → 任意容器"：

```cpp
template <class InputIterator>
list(InputIterator first, InputIterator last)
{
    while (first != last)
    {
        push_back(*first);
        ++first;
    }
}
```

> **迭代器就是一种特殊迭代器，前提是指针指向数组的指针**；更准确地说：**迭代器的行为本质就是模拟"指向数组的指针"的行为**。
> 所以指针 `a` 和 `a+4` 天然能当迭代器用——这就是 `list<int> lt5(a, a+4)` 能编译的原因。

### 2.2 遍历与"迭代器的封装价值"

```cpp
list<int>::iterator it4 = lt4.begin();
while (it4 != lt4.end())
{
    cout << *it4 << " ";
    ++it4;
}

for (auto e : lt4) { cout << e << " "; }   // 范围 for 底层就是迭代器
```

板书用一组类比说明"封装"的价值：

| 支付宝 / 微信 | 迭代器 |
| --- | --- |
| 银行卡支付、网银 | 数组 |
| 各银行验证方式：1. 密保 2. U 盾 3. 短信验证码 … | 链表、树、哈希 … |

> **支付平台把"各家银行千差万别的验证方式"统一成了一个"支付"按钮；迭代器把"数组、链表、树、哈希千差万别的遍历方式"统一成了 `begin/end/++/*`。**
>
> 这正是 `list<int>::iterator` 与 `vector<int>::iterator` 的底层实现**完全不同**，但用户写出来的代码**一模一样**的原因。
>
> `test_list1` 里那两行注释说得很清楚：
> 1. **封装**：通用、相似的遍历容器方式，并且**屏蔽容器结构的差异和底层实现细节**。
> 2. **通用/复用**：实现算法时用**迭代器函数模板**方式实现，跟底层容器结构**解耦**。

### 2.3 增删与 resize

```cpp
list<int> lt2(10, 1);
lt2.push_back(10);
lt2.push_front(10);       // ★ list 有 push_front！vector 没有

lt2.resize(20, 2);        // 变长：填 2
lt2.resize(5);            // 变短：从尾部删
```

> `vector` 没有 `push_front` 的原因：头插要挪动全部数据 `O(N)`，属于"能但不该提供"的接口。`list` 的头插是 `O(1)`，所以提供了。

### 2.4 list 的专有操作（vector 没有）

```cpp
void test_list3()
{
    std::list<double> first, second;
    first.push_back(3.1);  first.push_back(2.2);  first.push_back(2.9);
    second.push_back(3.7); second.push_back(7.1); second.push_back(1.4);

    first.sort();             // ★ 成员函数 sort（归并实现）
    second.sort();
    first.merge(second);      // ★ 合并两个有序链表，second 被搬空

    std::list<double> lt1 = { 1,2,3,4,5,6,2,3,2 };
    lt1.remove(2);            // ★ 删除所有值为 2 的节点

    std::list<double> lt2 = { 1,2,3,4,5,6 };
    // 把 5 转移到头部位置
    auto it = find(lt2.begin(), lt2.end(), 5);
    lt2.splice(lt2.begin(), lt2, it);   // ★ 转移节点（不拷贝、不释放，只改指针）
}
```

| 操作 | 作用 | 复杂度 | 说明 |
| --- | --- | --- | --- |
| `sort()` | 成员排序 | `O(N log N)` | **必须是成员函数**（用归并，见第三章） |
| `merge(other)` | 合并两个有序链表 | `O(N+M)` | `other` 会被清空 |
| `remove(val)` | 删除所有等于 `val` 的节点 | `O(N)` | |
| `remove_if(pred)` | 按条件删除 | `O(N)` | |
| `splice(pos, other, it)` | 把 `other` 里的 `it` 摘下来接到 `pos` 前 | **`O(1)`** | 只改指针，不拷数据 |
| `unique()` | 删除连续重复元素 | `O(N)` | 通常先 `sort` 再 `unique` |
| `reverse()` | 反转 | `O(N)` | 只改指针方向 |
| `assign(first, last)` | 清空后用区间重新赋值 | `O(N)` | |
| `resize(n, val)` | 调整大小 | `O(N)` | |

> **`splice` 是 list 的"独门绝技"**：把一个节点从一条链表"摘"到另一条链表，**不拷贝元素、不释放内存，只改 4 个指针**——`O(1)`。`vector` 想做到这个只能整体搬数据。

---

## 三、迭代器分类：算法对容器的要求

### 3.1 为什么 `sort(lt.begin(), lt.end())` 编译不过

```cpp
int a[] = { 1,20,3,40 };
sort(a, a + 4);              // ✔ 原生数组（随机访问）
vector<int> v1 = {...};
sort(v1.begin(), v1.end());  // ✔ vector（随机访问迭代器）

list<int> lt3(v1.begin(), v1.end());
// sort(lt3.begin(), lt3.end());   // ✘ 不能用，要求是随机迭代器
```

`std::sort` 的原型：

```cpp
template <class RandomAccessIterator>
void sort(RandomAccessIterator first, RandomAccessIterator last);
```

> **一个算法，不是所有的容器都可以使用，算法对迭代器是有一定要求的；算法迭代器名字就是要求（暗示）。**
> —— 看到形参名叫 `RandomAccessIterator`，就知道它要的是"随机访问迭代器"。

### 3.2 各算法对迭代器的要求

| 算法 | 形参名 | 需要的迭代器 |
| --- | --- | --- |
| `find` | `InputIterator` | 输入迭代器（单向逐个读） |
| `reverse` | `BidirectionalIterator` | 双向 |
| `remove` | `ForwardIterator` | 单向 |
| `sort` | `RandomAccessIterator` | 随机访问 |

```cpp
reverse(lt3.begin(), lt3.end());   // ✔ list 是双向迭代器
reverse(v1.begin(), v1.end());     // ✔
```

> `reverse` 的形参是 `BidirectionalIterator`，所以 `list` 能用；`sort` 要 `RandomAccessIterator`，所以 `list` 不能。
> **板书结论：`reverse` 要求双向，也可以传随机，因为随机是一个特殊的双向。**

### 3.3 五类迭代器与要求的运算符

```text
  Random Access ──→ Bidirectional ──→ Forward ──→ Input
       ↑                                          ↑
       └──────────────────────────────────────────┘
                    （能力由强到弱）

  另有 Output（只写不读），与上面是并列关系
```

| 能力 | 支持的运算符 | 典型容器 |
| --- | --- | --- |
| **单向**（Forward） | `++`、`*`、`!=` | 单链表、哈希表 |
| **双向**（Bidirectional） | `++`、`--`、`*`、`!=` | 红黑树、双向链表（`list`） |
| **随机访问**（Random Access） | `++`、`--`、`+`、`-`、`[]`、`<`、`>` | `string`、`vector`、双端队列 `deque` |

### 3.4 从标准文档的 Member types 看三种容器

| 容器 | `iterator` 的定义 |
| --- | --- |
| `list` | **a bidirectional iterator to value_type** |
| `vector` | **a random access iterator to value_type** |
| `forward_list` | **a forward iterator to value_type** |

其余常见 member types（三个容器基本一致）：

| member type | definition |
| --- | --- |
| `value_type` | 模板第一个参数 `T` |
| `allocator_type` | 模板第二个参数 `Alloc` |
| `reference` | `value_type&` |
| `const_reference` | `const value_type&` |
| `pointer` | `allocator_traits<Alloc>::pointer` |
| `const_pointer` | `allocator_traits<Alloc>::const_pointer` |
| `difference_type` | 有符号整型（两个迭代器相减的类型） |
| `size_type` | 无符号整型（表示任意非负的 difference_type） |

> `difference_type` 是**有符号**、`size_type` 是**无符号**——这是标准库的一个经典区分（`ptrdiff_t` vs `size_t`）。

---

## 四、为什么 vector 排序更快：CPU 三级缓存

### 4.1 板书上的实测

```cpp
void test_op1()
{
    srand(time(0));
    const int N = 100000;
    list<int> lt1;
    vector<int> v;

    for (int i = 0; i < N; ++i)
    {
        auto e = rand() + i;
        lt1.push_back(e);
        v.push_back(e);
    }

    int begin1 = clock();
    sort(v.begin(), v.end());        // vector 版排序
    int end1 = clock();

    int begin2 = clock();
    lt1.sort();                      // list 版排序
    int end2 = clock();

    printf("vector sort:%d\n", end1 - begin1);
    printf("list sort:%d\n", end2 - begin2);
}
```

VS2019 x86 实测（板书截图）：

| 配置 | `vector sort` | `list sort` |
| --- | --- | --- |
| **Debug** | **141** | **777** |
| **Release** | **53** | **107** |

> 即使 `list.sort()` 是"专为链表优化的归并"，**依然慢一倍以上**。原因不在算法，而在**内存布局**。

### 4.2 三级缓存与缓存命中

```text
速度：快 / 贵 / 小  →  慢 / 便宜 / 大

        [ CPU ]
           ↓
     ┌───────────┐
     │  L1 Cache │   （几 KB，最快）
     ├───────────┤
     │  L2 Cache │   （几百 KB）
     ├───────────┤
     │  L3 Cache │   （几 MB~几十 MB，多核共享）
     └───────────┘
           ↓
        [ 内存 ]      （几 GB~，最慢）
```

板书写下的两条规则：

> 1. **数据在不在缓存，在缓存就叫命中，不在就叫不命中。**
> 2. **不在缓存，先加载到缓存，再访问。**

关键点：**CPU 从内存搬数据不是"搬一个字节"，而是一下搬一整条 cache line（通常 64 字节）**。

```text
vector（连续）：  [1][2][3][4][5][6][7][8][9] ...   一次搬 64 字节 → 命中剩下一整批，几乎不浪费
list （离散）：   [1]->.  [2]->.  [3]->.  [4]->.     每取一个节点都要一次内存访问，每次都可能不命中
```

再加一层：`vector` 存的是**纯数据**（4 字节 int 紧挨着），而 `list` 每个节点还有 `_next`/`_prev` 两个指针（**有效数据占比可能不到 30%**）——搬同样的 cache line，`list` 能拿到的有用元素更少。

> 这就是**"缓存不友好（cache unfriendly）"** 的准确含义：不是"访问得慢"，而是"**每次访问都在浪费搬过来的那 64 字节**"。
> 板书旁边标注的 `快排 / 归并`、`cpu高速缓存命中率` 就是这一段的标题——**决定性能的不是算法复杂度，而是访存模式**。

### 4.3 工程结论：要给 list 排序怎么办

```cpp
void test_op2()
{
    srand(time(0));
    const int N = 1000000;

    list<int> lt1, lt2;
    for (int i = 0; i < N; ++i)
    {
        auto e = rand() + i;
        lt1.push_back(e);
        lt2.push_back(e);
    }

    int begin1 = clock();
    vector<int> v(lt2.begin(), lt2.end());   // ① 拷到 vector（连续内存）
    sort(v.begin(), v.end());                // ② 在 vector 上排序（快得多）
    lt2.assign(v.begin(), v.end());           // ③ 拷回 list
    int end1 = clock();

    int begin2 = clock();
    lt1.sort();                               // 对照：直接用 list 的 sort
    int end2 = clock();

    printf("list copy vector sort copy list sort:%d\n", end1 - begin1);
    printf("list sort:%d\n", end2 - begin2);
}
```

> **这个"拷贝 → 排序 → 拷贝回"的三步操作，看起来多做了两次 `O(N)`，实际上通常仍然比 `list.sort()` 快**——因为省下的是几十次内存往返。
> 这是"**复杂度相同，常数项差 10 倍**"的经典案例。延伸阅读见板书给的那篇 [CPU 缓存那些事](https://coolshell.cn/articles/20793.html)。

---

## 五、手写 list（一）：节点、迭代器、骨架

### 5.1 第一版（5-27）：能跑起来的最小实现

```cpp
#pragma once

namespace bit
{
    template<class T>
    struct list_node
    {
        T _data;
        list_node<T>* _next;
        list_node<T>* _prev;

        list_node(const T& x = T())
            :_data(x)
            ,_next(nullptr)
            ,_prev(nullptr)
        {}
    };

    template<class T>
    struct __list_iterator
    {
        typedef list_node<T> Node;
        Node* _node;

        __list_iterator(Node* node)
            :_node(node)
        {}

        T& operator*()
        {
            return _node->_data;
        }

        __list_iterator<T>& operator++()
        {
            _node = _node->_next;
            return *this;
        }

        bool operator!=(const __list_iterator<T>& it)
        {
            return _node != it._node;
        }
    };

    template<class T>
    class list
    {
        typedef list_node<T> Node;
    public:
        typedef __list_iterator<T> iterator;

        iterator begin() { return iterator(_head->_next); }
        iterator end()   { return iterator(_head); }

        list()
        {
            _head = new Node;
            _head->_next = _head;
            _head->_prev = _head;
        }

        void push_back(const T& x)
        {
            Node* newnode = new Node(x);
            Node* tail = _head->_prev;

            tail->_next = newnode;
            newnode->_prev = tail;
            newnode->_next = _head;
            _head->_prev = newnode;
        }

    private:
        Node* _head;
    };
}
```

### 5.2 为什么迭代器不能直接用 `Node*`

板书这一页是整个 list 的核心：

```cpp
// 迭代器用 Node*，无法达到预期行为
// 类封装 Node*，重载运算符，控制迭代器的行为
```

```text
        _head
          ↓
    ┌─────────┐   ┌───┐   ┌───┐   ┌───┐
    │  哨兵位  │──→│ 1 │──→│ 2 │──→│ 3 │──┐
    │         │←──└───┘←──└───┘←──└───┘  │
    └─────────┘   ↑                     │
       ↑          begin()                │
       │                                │
      end() ←───────────────────────────┘
```

假设 `typedef Node* iterator`，那么写 `while (it != lt.end()) cout << *it;` 会怎样？

- `*it` 得到的是**整个节点**（`list_node`），不是 `T`！用户要的是 `_data`。
- `++it` 会变成**指针算术**（`it + sizeof(Node)`），直接跳到非法内存——而链表需要的是 `_node = _node->_next`。

> 所以：**`vector` 的迭代器是原生指针（`T*`），`list` 的迭代器必须是一个类**——它内部存一个 `Node*`，然后把 `*`、`++`、`--`、`!=`、`==`、`->` 全部重载成"链表语义"。
> 这就是"**迭代器是行为像指针的对象**"这句话的真正含义。

---

## 六、手写 list（二）：const_iterator 的三种做法

### 6.1 需求：`const list` 不能被修改

```cpp
void print(const list<int>& lt)     // 形参是 const 引用
{
    list<int>::const_iterator it = lt.begin();
    while (it != lt.end())
    {
        // *it += 1;      // ✘ 不能修改 it 指向的节点的数据
        cout << *it << " ";
        ++it;             // ✔ 迭代器本身要能自增
    }
}
```

### 6.2 做法一（错）：`typedef const __list_iterator<T> const_iterator`

```cpp
// 不能实现，因为 const 迭代器本身，迭代器无法++
// typedef const __list_iterator<T> const_iterator;
```

> 这是最容易想到的"聪明"做法，但它是错的：`const __list_iterator<T>` 是"**迭代器对象本身是 const**"，导致 `++it`（要改 `_node`）直接编译不过。
> **我们要的是"指向的数据是 const"，不是"迭代器自己是 const"。** 对应到指针就是 `const T*`（对）而不是 `T* const`（错）。

### 6.3 做法二（能跑但啰嗦）：再写一个 `__list_const_iterator`

```cpp
template<class T>
struct __list_const_iterator
{
    typedef list_node<T> Node;
    Node* _node;

    const T& operator*()                       // ← 关键：返回 const T&
    {
        return _node->_data;
    }

    __list_const_iterator<T>& operator++()
    {
        _node = _node->_next;
        return *this;
    }
    // ... 其余 ++/--/!=/== 与普通迭代器几乎完全一样
};

// 类里
typedef __list_iterator<T>       iterator;
typedef __list_const_iterator<T> const_iterator;
```

课本上还留了一段注释说明"为什么不能在同一个类里靠 `const` 成员函数重载 `operator*`"：

```cpp
// 不能实现，因为 const __list_iterator 对象才能调用这个重载
// 但是 const __list_iterator 对象不能调用 ++
/*const T& operator*() const
{
    return _node->_data;
}*/
```

> 逻辑很关键：**`const` 成员函数只有 `const` 对象才能调，而 `const` 迭代器对象又不能 `++`**——两个条件自相矛盾。
> 所以要么写两个类，要么用模板参数。**代价是：两个类 90% 的代码重复。**

### 6.4 做法三（正解）：把 `Ref` / `Ptr` 变成模板参数

```cpp
template<class T, class Ref, class Ptr>
struct __list_iterator
{
    typedef list_node<T> Node;
    typedef __list_iterator<T, Ref, Ptr> Self;
    Node* _node;

    __list_iterator(Node* node)
        :_node(node)
    {}

    Ref operator*()
    {
        return _node->_data;
    }

    Ptr operator->()
    {
        return &_node->_data;
    }

    Self& operator++()   { _node = _node->_next; return *this; }
    Self  operator++(int){ Self tmp(*this); _node = _node->_next; return tmp; }
    Self& operator--()   { _node = _node->_prev; return *this; }
    Self  operator--(int){ Self tmp(*this); _node = _node->_prev; return tmp; }

    bool operator!=(const Self& it) const { return _node != it._node; }
    bool operator==(const Self& it) const { return _node == it._node; }
};
```

```cpp
template<class T>
class list
{
public:
    // 同一个类模板，实例化出两个不同的类型
    typedef __list_iterator<T, T&, T*>             iterator;
    typedef __list_iterator<T, const T&, const T*> const_iterator;
    ...
};
```

> 板书原话：**"同一个类模板实例化的两个类型"**。
>
> | 类型 | `Ref` | `Ptr` | `operator*` 返回 | 能否改数据 |
> | --- | --- | --- | --- | --- |
> | `iterator` | `T&` | `T*` | `T&` | ✔ `*it += 1` 可以 |
> | `const_iterator` | `const T&` | `const T*` | `const T&` | ✘ `*it += 1` 编译不过 |
>
> **这是模板最漂亮的一次应用**：不用复制粘贴第二个类，只把"差异点（`Ref`/`Ptr`）"提成模板参数。
> 顺带一提：标准库的 `list` 还会多两个参数（`iterator_category`、`difference_type`），思路完全一致。

### 6.5 配套的 `const` 版 `begin/end`

```cpp
iterator       begin()       { return iterator(_head->_next); }
iterator       end()         { return iterator(_head); }
const_iterator begin() const { return const_iterator(_head->_next); }
const_iterator end()   const { return const_iterator(_head); }
```

> 只有这样才能让 `print(const list<T>& lt)` 里 **既编得过、又改不了数据**：
> `const` 对象只能调 `const` 版 `begin/end` → 拿到 `const_iterator` → `*it` 是 `const T&` → 赋值编译错误。

---

## 七、手写 list（三）：`operator->` 与 `typename`

### 7.1 `operator->` 语法糖：为什么 `it->_row` 只写一个箭头

```cpp
struct Pos
{
    int _row;
    int _col;
    Pos(int row = 0, int col = 0) :_row(row), _col(col) {}
};

std::list<Pos> lt1;
lt1.push_back({1,1});
...

auto it = lt1.begin();
while (it != lt1.end())
{
    // cout << (*it)._row << ":" << (*it)._col << endl;   // 原始写法：先解引用再取成员
    cout << it->_row << ":" << it->_col << endl;          // 简洁写法：为了可读性，这里省略了一个 ->
    // 完整展开：it.operator->()->_row
    ++it;
}
```

```cpp
Ptr operator->()
{
    return &_node->_data;      // 返回指向 _data 的指针
}
```

**C++ 对 `->` 的语法糖约定**：

> `it->x` 会被编译器翻译成 `(it.operator->())->x`。
> **`operator->()` 返回一个指针，那个指针再用一次 `->` 才是真正的成员访问**——所以视觉上"少了一个箭头"。

所以 `list<Pos>::iterator` 的 `it->_row` 实际执行的是：

```text
it.operator->()          → Pos*（指向节点里的 _data）
        ↓
(it.operator->())->_row  → int
```

> 这也解释了 `operator*` 和 `operator->` 在迭代器里几乎是**绑定出现**的：`*it` 拿对象，`it->` 拿成员。

### 7.2 `typename`：类模板里为什么必须写

```cpp
template<class T>
void print(const list<T>& lt)
{
    // 类模板未实例化，不能去类模板中找后面的东西
    // 编译器就分不清 const_iterator 是嵌套内类，还是静态成员变量
    // typename 告诉编译器，我确认过了这里是类型
    //typename list<T>::const_iterator it = lt.begin();
    auto it = lt.begin();
    while (it != lt.end())
    {
        cout << *it << " ";
        ++it;
    }
    cout << endl;
}
```

**问题的根源**：写 `list<T>::const_iterator` 时，`T` 还没确定 → `list<T>` 是个**未实例化的模板** → 编译器无法知道 `const_iterator` 到底是

- 一个**类型**，还是
- 一个**静态成员变量**

而这两种解释在语法上是不同含义，`typename list<T>::const_iterator it;` 只有在"它是类型"时合法。所以 C++ 规定：**从属名（依赖模板参数的名字）默认按"非类型"解释，要当类型用必须加 `typename`。**

```cpp
// 写法对比
typename list<T>::const_iterator it = lt.begin();   // ✔ 明确告诉编译器"这是类型"
auto it = lt.begin();                               // ✔ 更省事（C++11 之后首选）
```

> 顺带解释 `test_list1` 里那句 `list<int>::iterator it = lt.begin();` 为什么**不用** `typename`：因为 `list<int>` **已经实例化**了，编译器知道 `iterator` 是类型。**只有"依赖模板参数"的名字才需要 `typename`。**

---

## 八、手写 list（四）：增删改与拷贝控制

### 8.1 全部增删都复用 `insert` / `erase`

```cpp
iterator insert(iterator pos, const T& val)
{
    Node* cur = pos._node;
    Node* newnode = new Node(val);
    Node* prev = cur->_prev;

    // prev newnode cur
    prev->_next = newnode;
    newnode->_next = cur;
    cur->_prev = newnode;
    newnode->_prev = prev;

    ++_size;

    return iterator(newnode);
}

iterator erase(iterator pos)
{
    Node* cur = pos._node;
    Node* prev = cur->_prev;
    Node* next = cur->_next;

    prev->_next = next;
    next->_prev = prev;
    delete cur;

    --_size;

    //return iterator(next);
    return next;
}
```

`insert` 的三指针图（板书）：

```text
              pos
               ↓
     ┌───┐   ┌───┐   ┌───┐
     │ 1 │   │ 3 │   │ 4 │
     └───┘   └───┘   └───┘
       prev    cur
          ↑
       newnode

  // prev newnode cur
  prev->_next = newnode;
  newnode->_next = cur;
  cur->_prev  = newnode;
  newnode->_prev = prev;
```

`erase` 的三指针图：

```text
              pos
               ↓
     ┌───┐   ┌───┐   ┌───┐
     │ 1 │   │ 2 │   │ 4 │
     └───┘   └───┘   └───┘
      prev    cur     next
        ↘_______↗      （prev 直接连到 next，然后 delete cur）
```

**四个接口全部是一行**：

```cpp
void push_back(const T& x)  { insert(end(), x); }     // 尾插 = 在哨兵位前插
void push_front(const T& x) { insert(begin(), x); }   // 头插 = 在首节点前插
void pop_back()             { erase(--end()); }       // 尾删：end() 前一个是尾节点
void pop_front()            { erase(begin()); }
```

> 这就是"带头循环链表"的最大红利：**头、尾、中间，全部是同一套逻辑**，不像单链表要分"删的是不是头结点"。

### 8.2 `clear` 与析构

```cpp
void clear()
{
    iterator it = begin();
    while (it != end())
    {
        it = erase(it);        // ★ erase 返回下一个，才能安全遍历删除
    }
}

~list()
{
    clear();                   // 先释放所有数据节点
    delete _head;              // 再释放哨兵位
    _head = nullptr;
}
```

> ⚠️ **`it = erase(it)` 是必须的写法**：`erase` 之后 `it` 已经指向被 `delete` 的节点，再用就是未定义行为（和 `vector` 的规则一致）。
> 对比 `list - 副本.h` 里那个"忘了写"的析构函数：

```cpp
~list()
{
    /*iterator it = begin();
    while (it != end())
    {
        it = erase(it);
    }
    delete _head;
    _head = nullptr;*/
}                              // ← 整个函数体被注释掉 = 内存泄漏
```

### 8.3 构造、拷贝、赋值

```cpp
void empty_init()
{
    _head = new Node;
    _head->_next = _head;
    _head->_prev = _head;
}

list()
{
    empty_init();
}

// lt2(lt1)
list(const list<T>& lt)
{
    empty_init();              // ★ 先建哨兵位，再 push_back
    for (const auto& e : lt)
    {
        push_back(e);          // 深拷贝：每个节点都是新建的
    }
}

// lt3 = {1,2,3}
list(initializer_list<T> il)
{
    empty_init();
    for (const auto& e : il)
    {
        push_back(e);
    }
}

void swap(list<T>& lt)
{
    std::swap(_head, lt._head);
    std::swap(_size, lt._size);
}

// lt1 = lt3  —— 现代写法：传值 + swap
list<T>& operator=(list<T> lt)
{
    swap(lt);
    return *this;
}
```

**传统写法（板书中被注释掉的对照版）**：

```cpp
/*list<T>& operator=(const list<T>& lt)
{
    if (this != &lt)            // 要判自赋值
    {
        clear();               // 先清空
        for (const auto& e : lt)
        {
            push_back(e);      // 再逐个拷
        }
    }
    return *this;
}*/
```

| | 传统写法 | 现代写法（传值 + `swap`） |
| --- | --- | --- |
| 自赋值 | 必须 `if (this != &lt)` | 天然免疫 |
| 异常安全 | 中途抛异常会让对象半残 | 拷贝失败在传参阶段，`*this` 未动 |
| 代码量 | 长 | 两行 |

> 拷贝构造里**只需 `empty_init()` 而不需要 `clear()`**——因为新对象还没有数据；但**赋值**必须处理"我原来是有数据的"，这正是 `swap` + 析构的巧妙之处：旧数据跟着形参 `lt` 一起被析构掉。

### 8.4 `size()`：记一个变量还是每次遍历

```cpp
size_t size() const
{
    /*size_t n = 0;
    for (auto& e : *this)
    {
        ++n;
    }
    return n;*/

    return _size;              // ★ 用成员变量，O(1)
}

private:
    Node* _head;
    size_t _size = 0;          // 每次 insert ++，erase --，clear 归零
}
```

> **为什么不用遍历算？** 遍历是 `O(N)`。`list::size()` 在 C++11 之后被要求是 `O(1)`，所以标准库实现也维护了一个 `_size` 计数。
> 代价是：`insert`/`erase`/`clear` 里都要记得维护它（`++_size`、`--_size`、`clear` 后置 0）。

### 8.5 一个副产品：`vector` 的 `int`/`size_t` 重载与 `enable_if_t`

这一版板书顺带把上一节 `vector` 遗留的问题彻底解决了。

**问题回顾**：`vector<int> v(10, 1)` 会去匹配"更匹配"的函数模板（迭代器区间构造），把 `10` 当迭代器 → 报错。所以课上先加了 `int` 版本：

```cpp
vector(int n, const T& val)      { resize(n, val); }
vector(size_t n, const T& val)   { resize(n, val); }
```

**更正统的解法（了解）**：用 SFINAE 把"不是迭代器的类型"排除掉。

```cpp
// 函数模板
// 任意类型容器迭代器初始化
template <class InputIterator>
vector(InputIterator first, InputIterator last,
       enable_if_t<Is_iterator_v<InputIterator>, int> = 0)
{
    while (first != last)
    {
        push_back(*first);
        ++first;
    }
}
```

> 板书原话：**"不是迭代器，就不实例化这个函数模板"**。
> `enable_if_t<条件, 类型>` 的语义是：**条件为假时，这个函数模板直接"替换失败"→ 不参与重载决议**（SFINAE），于是 `vector<int> v(10, 1)` 就只能匹配到 `(int, T)` 那个版本，**歧义从根上消失**，也不需要再写两个 `n, val` 版本。
> 这是标准库真实采用的写法——课上标注为"**了解**"级别。

---

## 九、速查表与易错清单

### 9.1 vector vs list

| | `vector` | `list` |
| --- | --- | --- |
| 底层结构 | 连续数组 | 带头双向循环链表 |
| 迭代器 | `T*`（原生指针） | 类封装 `Node*` |
| 迭代器类型 | 随机访问 | **双向** |
| 随机访问 `[]` | `O(1)` | ✘ 不支持 |
| 头插/头删 | `O(N)`（没提供） | **`O(1)`**（`push_front`） |
| 中间插入删除 | `O(N)` | **`O(1)`**（已持有迭代器时） |
| 查找 | `O(N)` | `O(N)`（不能二分） |
| 空间开销 | 小（可能浪费 capacity） | 大（每个节点两个指针） |
| 缓存友好度 | **好**（连续） | 差（离散） |
| 迭代器失效 | **扩容/插入/删除都会失效** | **只有被删的那个节点失效** |
| 排序 | `std::sort` | 成员 `sort()`（归并） |

### 9.2 手写 list 接口全表

| 接口 | 关键实现 | 易错点 |
| --- | --- | --- |
| `empty_init()` | 建哨兵位，`_next = _prev = _head` | 忘建 → `begin()` 解引用空指针 |
| `begin()` / `end()` | `_head->_next` / `_head` | `end()` 是**哨兵位**，不是尾节点 |
| `push_back` | `insert(end(), x)` | 别自己写指针，让 `insert` 统一处理 |
| `push_front` | `insert(begin(), x)` | |
| `pop_back` | `erase(--end())` | 空链表时 `--end()` 就是哨兵位，需自行保证非空 |
| `insert` | 改 4 个指针 + `++_size`，返回新节点 | 别漏 `_size`；别忘 `return` |
| `erase` | 改 2 个指针 + `delete` + `--_size`，返回下一节点 | ⚠️ 必须返回 `next`，否则调用方无法安全遍历删除 |
| `clear` | `while (it != end()) it = erase(it);` | 用 `++it` 会漏删/悬垂 |
| `~list` | `clear()` 再 `delete _head` | 只 `delete _head` 会泄漏全部节点 |
| 拷贝构造 | `empty_init()` + 逐个 `push_back` | 忘 `empty_init` → 野指针 |
| `operator=` | 传值 + `swap` | 别写成 `const list&`（丢掉现代写法好处） |
| `size()` | 返回 `_size` | 每次 `insert`/`erase` 都要维护 |
| `iterator` | `__list_iterator<T, T&, T*>` | 别试图用 `const __list_iterator` 当 `const_iterator` |
| `operator->` | `return &_node->_data;` | 记得 `it->x` 会自动多套一层 `->` |
| 模板里用嵌套类型 | `typename list<T>::const_iterator` | 漏 `typename` → 编译不过 |

### 9.3 易错清单

1. **`end()` 不是尾节点**：`end()` 就是哨兵位 `_head`；要尾节点写 `_head->_prev`。
2. **迭代器不能 `typedef Node*`**：`*it` 会拿到整个节点、`++it` 会变成指针算术。
3. **`const_iterator` 不要用 `const __list_iterator<T>`**：`const` 修饰的是迭代器自身，导致 `++` 不能用。**正解是模板参数 `Ref`/`Ptr`**。
4. **`erase` 必须返回下一节点**：`it = lt.erase(it)` 是遍历删除的唯一安全写法。
5. **`clear` 里不能用 `++it`**：节点已被 `delete`。
6. **拷贝构造忘 `empty_init()`**：新对象的 `_head` 是未初始化的野指针。
7. **`_size` 忘了维护**：`insert`/`erase`/`clear` 三处都要管。
8. **`std::sort` 不能用于 `list`**：它要求随机访问迭代器，用 `lt.sort()`。
9. **`typename` 漏写**：`list<T>::const_iterator` 在模板里必须写 `typename`（或用 `auto`）。
10. **`operator->` 的返回值**：要返回"指向数据"的指针 `&_node->_data`，不是返回节点指针。
11. **`splice` 之后原迭代器**：节点被摘走，指向它的迭代器仍指向同一节点，但属于新链表——别混用。
12. **不该用 list 的场景**：需要随机访问、需要排序、需要缓存友好 → 用 `vector`（**绝大多数场景 `vector` 都更快**）。

---

> 下一篇进入 `stack` / `queue`：`25年-06月-03日-stack_queue` 与 `25年-06月-05日-stack_queue` 开始用**容器适配器**的方式把 `vector`/`list`/`deque` 复用成栈与队列，再往后 `25年-07月-10日` 是 `priority_queue`（堆）。
> `stack`/`queue` 的关键转变：**它们不自己管理内存，只"包一层"限制接口**——这正是"适配器（配器）"这个 STL 组件的含义。
