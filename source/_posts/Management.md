---
title: Management
date: 2026-11-07 22:48:05
tags: [c++, 内存管理]
description: 'C++ 内存管理：虚拟进程地址空间、C/C++ 内存分布、new/delete 与 malloc/free、operator new/delete、定位 new、内存池'
---

> 内容来源：
> - `25年-05月-06日--类和对象+内存管理`：虚拟进程地址空间布局、C/C++ 内存分布、选择题、`new/delete` 初识
> - `25年-05月-08日--内存管理+模板`：`new/delete` 的底层（`operator new` / `operator delete` / 构造析构的调用顺序）、`delete[]` 的数组 cookie、`new` 失败抛异常、池化技术与内存池、定位 `new`、`malloc/free` 与 `new/delete` 的 6 点区别（模板部分另见 `Template.md`）

## 目录

- **一、虚拟进程地址空间** —— 32 位 4G、64 位 2^48、栈 / 堆 / 静态区 / 常量区
- **二、C/C++ 内存分布（必考选择题）** —— 每一类变量落在哪
- **三、`malloc` / `calloc` / `realloc` 的区别**
- **四、`new` / `delete` 的基本用法**
- **五、`new/delete` 与 `malloc/free` 的区别（6 点）**
- **六、`new/delete` 的三层结构（底层原理）** —— `operator new` + 构造、析构 + `operator delete`
- **七、`delete[]` 的数组 cookie 与配对原则**
- **八、`new` 失败为什么抛异常**
- **九、池化技术、内存池与定位 new**
- **十、速查表与易错清单**

---

## 一、虚拟进程地址空间

程序运行时，操作系统给每个进程看到的都是一块"独享"的虚拟地址空间。以 32 位进程为例，地址总线 32 根，能表示的地址范围是 `0 ~ 2^32-1`，所以**虚拟地址空间是 4G**。

```text
1G   = 1024MB
1024MB = 1024 * 1024KB
1024 * 1024 * 1024 = 2^30 byte
32 位：2^32 byte = 4G
64 位：2^32 * 4G = 2^48 byte        （≈ 256TB）
```

> 板书补充：x86-64 实际只用到低 48 位地址线，所以写成 `2^48`；Windows 上用户态通常只能用其中一半（128TB）。

32 位进程地址空间从高到低的大致分布：

```text
高地址
┌─────────────────────────┐
│  内核空间（用户不可见）      │
├─────────────────────────┤
│  命令行参数 / 环境变量       │
├─────────────────────────┤
│  栈（stack）              │  ← 局部变量、函数栈帧（ebp / esp 维护）
│        ↓ 向低地址增长      │     向下增长，空间有限（默认几 MB）
│                          │
│        ↑ 向高地址增长      │
│  堆（heap）               │  ← malloc / calloc / realloc / new
├─────────────────────────┤
│  静态区 / 数据段（.data）   │  ← 全局变量、静态变量
├─────────────────────────┤
│  常量区 / 代码段（.text）   │  ← 字面量常量、可执行代码
└─────────────────────────┘
低地址

（中间还有一块共享库 / 文件映射区，位于栈和堆之间高处）
```

要点：

- **栈**：由编译器自动管理，函数进栈建栈帧，`ebp` 指向栈帧底（高地址），`esp` 指向栈顶（低地址），出栈自动回收。生长方向**由高到低**。
- **堆**：由程序员手动申请、手动释放（`malloc/free`、`new/delete`），生长方向**由低到高**。忘了释放就是内存泄漏。
- **静态区 / 数据段**：全局变量、`static` 变量。进程启动就存在，进程结束才销毁。
- **常量区 / 代码段**：字符串字面量、`const` 全局常量、机器指令。**只读**，写它直接段错误。

> 顺带一个容易混的点：**`const` 局部变量（`const int x = 10;`）在栈上**，只有字符串字面量和全局 `const` 才在常量区。所以 `cout << &x` 打印出来的地址和普通局部变量 `y` 的地址是同一片区域。

## 二、C/C++ 内存分布（必考选择题）

```cpp
int globalVar = 1;                          // 全局
static int staticGlobalVar = 1;             // 全局静态

void Test()
{
    static int staticVar = 1;               // 静态局部
    int localVar = 1;                       // 普通局部

    int num1[10] = { 1, 2, 3, 4 };
    char char2[] = "abcd";
    const char* pChar3 = "abcd";            // 指针在栈，指向常量区的字面量
    int* ptr1 = (int*)malloc(sizeof(int) * 4);
    int* ptr2 = (int*)calloc(4, sizeof(int));
    int* ptr3 = (int*)realloc(ptr2, sizeof(int) * 4);
    free(ptr1);
    free(ptr3);
}
```

选择题：`A. 栈　B. 堆　C. 数据段（静态区）　D. 代码段（常量区）`

| 表达式 | 答案 | 理由 |
| :--- | :--- | :--- |
| `globalVar`（变量本身） | **C 数据段** | 全局变量在静态区 |
| `staticGlobalVar` | **C 数据段** | 全局静态变量在静态区 |
| `staticVar` | **C 数据段** | 静态局部变量在静态区（生命周期是整个程序） |
| `localVar` | **A 栈** | 普通局部变量在栈 |
| `num1`（数组名/数组本身） | **A 栈** | 局部数组，整个数组在栈上 |
| `char2` | **A 栈** | 局部数组，`"abcd"` 被**拷贝**到栈上的数组里 |
| `*char2` | **A 栈** | `char2[0]` 是栈上的字符 `'a'` |
| `pChar3` | **A 栈** | 指针**变量本身**在栈 |
| `*pChar3` | **D 常量区** | 指向的字面量 `"abcd"` 在只读常量区 |
| `ptr1` | **A 栈** | 指针变量在栈 |
| `*ptr1` | **B 堆** | 指向的 4 个 int 空间在堆上 |

> 记住这组"对称关系"，选择题基本不会错：
> - `char2[] = "abcd"` → 数组在**栈**，内容可改（`char2[0] = 'x'` 合法）。
> - `const char* p = "abcd"` → 指针在**栈**，指向的内容在**常量区**，修改内容会崩。
> - 凡是 `malloc/calloc/realloc/new` 出来的**空间**都在**堆**，但**存指针的变量**通常在**栈**。

## 三、`malloc` / `calloc` / `realloc` 的区别

| 函数 | 作用 | 是否初始化 | 参数 |
| :--- | :--- | :--- | :--- |
| `malloc(size)` | 申请 `size` 字节 | **不初始化**（内容是随机值） | 总字节数 |
| `calloc(n, size)` | 申请 `n * size` 字节 | **初始化为 0** | 元素个数 + 单个大小 |
| `realloc(p, size)` | 调整已申请块的大小 | 新扩出来的部分不初始化 | 原指针 + 新总大小 |

```cpp
int* p1 = (int*)malloc(sizeof(int) * 4);       // 16 字节，内容随机
int* p2 = (int*)calloc(4, sizeof(int));        // 16 字节，全部是 0
int* p3 = (int*)realloc(p2, sizeof(int) * 10); // 扩到 40 字节，前 16 字节保留

free(p1);
free(p3);   // ⚠️ realloc 成功后会释放（或搬移）原 p2，只能 free 新指针
```

面试常问的"三者区别"标准答案就是上表三行 + 一句 `realloc` 有可能**原地扩、也可能搬家**（搬家时自动 `free` 老块）。

## 四、`new` / `delete` 的基本用法

```cpp
// 动态申请一个 int 空间（不初始化）
int* ptr4 = new int;

// 动态申请一个 int 空间并初始化为 1
int* ptr5 = new int(1);

// 动态申请 10 个 int 空间（不初始化）
int* ptr6 = new int[10];

// 动态申请 10 个 int 空间，前 4 个初始化为 1 2 3 4，其余为 0
int* ptr7 = new int[10]{ 1, 2, 3, 4 };

delete ptr4;
delete ptr5;
delete[] ptr6;      // ⚠️ 数组必须用 delete[]
delete[] ptr7;

// 自定义类型：new 会自动调用构造函数，delete 会自动调用析构函数
A* p2 = new A(1);
delete p2;
```

**`new` / `delete` 与 `malloc` / `free` 最本质的区别**（5-6 课核心结论）：

> `new/delete` 与 `malloc/free` 最大的区别是：**`new/delete` 对于【自定义类型】除了开空间，还会调用构造函数和析构函数**。

```cpp
class A
{
public:
    A(int a) : _a(a) { cout << "A():" << this << endl; }
    ~A() { cout << "~A():" << this << endl; }
private:
    int _a;
};

struct ListNode
{
    int _val;
    ListNode* _next;

    ListNode(int val = 0)
        : _val(val)
        , _next(nullptr)
    {}
};

int main()
{
    A* p1 = (A*)malloc(sizeof(A));   // 只开空间：不调构造（里面是随机值）
    free(p1);                        // 只释放空间：不调析构

    A* p2 = new A(1);                // 开空间 + 调用 A(int) 构造
    delete p2;                       // 调用 ~A() + 释放空间

    A* p3 = new A[3]{ 1, 2, 3 };     // 构造 3 个对象
    delete[] p3;                     // 析构 3 个对象 + 释放整块空间

    // new 让"写链表"变成 C++ 里最舒服的写法之一：构造 + 初始化一步到位
    ListNode* n1 = new ListNode(1);
    ListNode* n2 = new ListNode(2);
    ListNode* n3 = new ListNode(3);
    ListNode* n4 = new ListNode(4);

    n1->_next = n2;
    n2->_next = n3;
    n3->_next = n4;

    return 0;
}
```

> 换到 `malloc` 写链表，要写成 `ListNode* n = (ListNode*)malloc(sizeof(ListNode)); n->_val = 1; n->_next = nullptr;`——三行代码换 `new` 一行，而且没有构造函数保证对象一创建就是合法状态。

## 五、`new/delete` 与 `malloc/free` 的区别（6 点）

`malloc/free` 和 `new/delete` 的共同点是：**都是从堆上申请空间，并且需要用户手动释放**。不同点：

| # | 维度 | `malloc` / `free` | `new` / `delete` |
| :-- | :--- | :--- | :--- |
| 1 | **本质** | `malloc/free` 是**函数** | `new/delete` 是**操作符**（关键字） |
| 2 | **初始化** | 只开空间，**不初始化** | 可以初始化（`new int(1)`、`new int[10]{1,2,3}`） |
| 3 | **参数** | 需要**手动算字节数**，且只能按字节传，`sizeof` 要自己写；数组要自己算 `n * sizeof(T)` | 只需给类型和个数，空间大小由编译器计算，多个对象写 `new T[n]` |
| 4 | **返回值** | 返回 `void*`，使用时**必须强制转换** | 返回的就是对应类型的指针，**不需要强转** |
| 5 | **失败处理** | 返回 `NULL`，**必须判空** | 抛 `bad_alloc` 异常，需要**捕获异常**（`try/catch`） |
| 6 | **自定义类型** | 只开/释放空间，**不调构造和析构** | 申请空间后**自动调用构造函数**完成初始化，释放前**自动调用析构函数**完成资源清理 |

> 记忆口诀：**「函数 vs 操作符、不初始化 vs 可初始化、算字节 vs 给类型、void\* 要转 vs 直接是类型、判空 vs 捕获异常、不管生 vs 管生管死」**。

## 六、`new/delete` 的三层结构（底层原理）

`new` 和 `delete` 是操作符，编译器会把它们展开成三步。板书用 `Stack* p1 = new Stack(10); delete p1;` 配合汇编做了拆解：

```text
new Stack(10)  →
    ① 调用 operator new(sizeof(Stack))      开空间（内部就是 malloc 的封装）
    ② 在开好的空间上调用 Stack::Stack(10)   构造
    ③ 把地址返回给 p1

delete p1      →
    ① 调用 Stack::~Stack()                  析构（清理对象内部资源）
    ② 调用 operator delete(p1)              释放空间（内部就是 free 的封装）
```

意义：

- **`operator new` / `operator delete` 只是"开空间/释放空间"**，它们**不是**重载了 `new`，也不调用构造/析构——本质上等价于 `malloc/free` + 失败抛异常。
- **`new` 操作符 = `operator new` + 构造函数**；**`delete` 操作符 = 析构函数 + `operator delete`**。
- 这也是"为什么 `new` 出来必须用 `delete`、`malloc` 出来必须用 `free`"的根源：混用会导致**构造/析构不配对**（对内置类型可能侥幸没事，对自定义类型就是未定义行为甚至崩溃）。
- `operator new` 和 `operator delete` 可以**在类内重载**（甚至全局重载），这也是内存池/对象池的切入点（见第九节）。
- 标准库里还有一个 `operator new[]` / `operator delete[]`，专门服务于 `new T[n]` / `delete[]`。

```cpp
// 一个 Stack 的例子：注意 new 出来的对象，其内部资源也要在析构里释放
class Stack
{
public:
    Stack(int n = 4)
        : _a((int*)malloc(sizeof(int) * n))   // 内部还有一次堆申请
        , _capacity(n)
        , _top(0)
    {}

    ~Stack()                                  // delete 时会被自动调用，内部资源才不会漏
    {
        free(_a);
        _a = nullptr;
        _top = _capacity = 0;
    }

private:
    int* _a;
    size_t _capacity;
    size_t _top;
};

int main()
{
    Stack* p1 = new Stack(10);   // operator new + 构造（构造里再 malloc）
    delete p1;                   // 析构（free 内部空间）+ operator delete
    return 0;
}
```

> 如果把这里的 `delete p1;` 换成 `free(p1);`：`Stack` 析构不执行 → 内部 `_a` 指向的数组**泄漏**。这就是"一定要配对"的现实后果。

## 七、`delete[]` 的数组 cookie 与配对原则

```cpp
A* p1 = new A[10];
delete[] p1;    // 正确：析构 10 次
```

问题：`delete[]` 怎么知道要调用多少次析构函数？

> 板书批注：**知道数组中多少个对象，`delete[]` 要调用多少次析构函数。**

实现方式是：`new T[n]` 在真正返回给你的地址**前面**多申请几个字节（通常是 4 或 8 字节）的 **cookie**，里面存元素个数 `n`；`delete[]` 拿到指针后往前读这个 cookie，就知道循环调用多少次析构，然后 `operator delete[]` 释放整块空间。

```text
        p1（返回给用户的地址）
        ↓
┌────────┬────────┬────────┬────────┬─────┐
│   10   │  obj0  │  obj1  │  ...   │obj9 │
└────────┴────────┴────────┴────────┴─────┘
  cookie
（存的元素个数，delete[] 靠它决定析构次数）
```

所以：

- **`new[]` 必须配 `delete[]`，`new` 必须配 `delete`，否则结果难料**——这是板书上最醒目的红色批注。
- `new T[n]` + `delete p` → 只析构第一个对象（cookie 被当成对象数据、堆管理信息被破坏），**未定义行为**。
- `new T` + `delete[] p` → 把普通对象当数组处理，读越界的 cookie，**未定义行为**。
- 对**内置类型**（`int` 等）没有析构函数，混用有时"看起来能跑"，但依然是未定义行为，属于典型的"踩坑还没报错"。

> C++ 还有一处非常隐蔽的坑：**多态场景下基类析构函数不是 `virtual`**，`delete[]` 或 `delete` 基类指针只调基类析构 → 派生类资源泄漏。所以准备被继承的类，析构函数必须 `virtual`。

## 八、`new` 失败为什么抛异常

5-8 课用"不断申请 4M"的实验验证了内存上限：

```cpp
void func()
{
    for (size_t i = 0; i < 100000; i++)
    {
        int* p1 = new int[1024 * 1024];     // 1024*1024 个 int = 4M
        cout << i << "->" << p1 << endl;    // 打印地址，观察什么时候崩
    }
}

int main()
{
    try
    {
        func();
    }
    catch (const exception& e)      // bad_alloc 继承自 exception
    {
        cout << e.what() << endl;   // 例如：bad allocation
    }
    return 0;
}
```

结论：

- 32 位进程堆上限大概 2G 左右，`4M * 493 ≈ 2G`，所以打印到 **493 次左右**就抛 `bad_alloc`（板书在 `i == 493` 处打了个断点 `int x = 0;` 来定位）。
- `new` 申请失败**不返回 `NULL`，而是抛 `std::bad_alloc` 异常**；`malloc` 失败才返回 `NULL`。这就是第五节第 5 点的由来。
- 所以 `new` 之后**不需要判空**，但要保证程序能处理异常（或者用 `new (std::nothrow) T` 让它返回空指针——不常用）。

## 九、池化技术、内存池与定位 new

### 9.1 为什么要池化

板书用了一个很形象的比喻：

```text
爸爸 ──(给 20 块)──▶ 儿子（大学，高频用钱）──▶ 微信 亲密付 ──▶ 妈妈（家里的"管家"/系统）
女儿（小学，3 块）
```

- **系统（妈妈）** = 真正管着大笔内存的人，但每次找她要都得走完整流程，慢。
- **高频申请/释放小块内存的程序** = 儿子，一次次找妈妈要钱（系统调用 / 堆管理器）太费劲。
- **内存池** = 爸爸先批量拿到一大笔，儿子日常花销走内存池，实在不够再找系统要。

对应到技术名词，这一族都叫**池化技术**：**内存池、线程池、连接池**——核心思想都是**一次拿一批，反复复用，减少系统开销**。

### 9.2 `operator new` / `operator delete` 就是最原始的"内存池"

标准库实现里，真正干活的往往是 `allocator`（分配器）：

```cpp
link_type get_node() { return list_node_allocator::allocate(); }
void put_node(link_type p) { list_node_allocator::deallocate(p); }

link_type create_node(const T& x)
{
    link_type p = get_node();      // 只开空间
    __STL_TRY {
        construct(&p->data, x);    // 在空间上构造（placement new）
    }
    __STL_UNWIND(put_node(p));
    return p;
}

void destroy_node(link_type p)
{
    destroy(&p->data);             // 先析构
    put_node(p);                   // 再还空间
}
```

> 这正是 STL `list` 的节点管理代码——**"开空间"和"构造"是分开的两步**，`operator new`/`allocate` 只负责前者。STL 的二级空间配置器（`alloc`）会在 `allocate/deallocate` 里维护**自由链表**，就是典型的内存池。

### 9.3 定位 new（placement new）

**定位 new 表达式**：在**已经分配好的空间**上显式调用构造函数。

```cpp
// 标准库的写法
template <class T>
inline void destroy(T* pointer)
{
    pointer->~T();
}

template <class T1, class T2>
inline void construct(T1* p, const T2& value)
{
    new (p) T1(value);      // ← 定位 new：在 p 指向的空间上构造 T1
}
```

用法：

```cpp
class A
{
public:
    A(int a = 0) : _a(a) { cout << "A():" << this << endl; }
    ~A() { cout << "~A():" << this << endl; }
private:
    int _a;
};

int main()
{
    A* p1 = (A*)malloc(sizeof(A));   // 只有空间，没有对象

    // p1->A(1);  ✗ 不能这样显式调用构造函数
    new (p1) A(1);                   // ✓ 定位 new：在已开好的空间上构造

    p1->~A();                        // 显式调用析构（析构可以显式调用）
    free(p1);                        // 再释放空间

    return 0;
}
```

> 定位 new 的典型用途：
> - **内存池 / 对象池**：池子只负责开大块内存，对象要哪块就在哪块上"定位 new"构造，用完显式析构后归还给池子（不还给系统）。
> - 需要在**指定地址**（如共享内存、嵌入式固定地址、`std::vector` 的未初始化缓冲区）上创建对象时。
>
> 注意：定位 new 开的"空间"如果来自 `malloc`，最后要用 `free`；如果来自 `operator new`，要用 `operator delete`——**底下的内存从哪来，就还给哪**。

## 十、速查表与易错清单

### 📊 四种申请方式对照

| 方式 | 开空间 | 调构造 | 调析构 | 失败行为 | 释放方式 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `malloc/free` | ✓ | ✗ | ✗ | 返回 `NULL` | `free` |
| `operator new/delete` | ✓ | ✗ | ✗ | 抛 `bad_alloc` | `operator delete` |
| `new/delete`（操作符） | ✓ | ✓ | ✓ | 抛 `bad_alloc` | `delete` |
| `new[]/delete[]` | ✓ | ✓（n 次） | ✓（n 次，靠 cookie） | 抛 `bad_alloc` | `delete[]` |
| 定位 `new` | ✗（用现成空间） | ✓ | 需手动 `p->~T()` | — | 视底层空间来源释放 |

### 📊 内存分区速查

| 东西 | 位置 |
| :--- | :--- |
| 局部变量、函数形参、指针变量本身、局部数组 | 栈 |
| `malloc/calloc/realloc/new` 出来的空间 | 堆 |
| 全局变量、`static` 全局、`static` 局部 | 静态区（数据段） |
| 字符串字面量、全局 `const`、函数机器码 | 常量区（代码段） |

### ⚠️ 易错清单（考前最后一遍）

1. **`char2[] = "abcd"` 的内容在栈**（可改）；**`const char* p = "abcd"` 指向的内容在常量区**（写它会崩）——但两个**指针/数组本身**都在栈。
2. `const int x = 10;`（局部）在**栈**，不在常量区。
3. `malloc` 不初始化，`calloc` 初始化为 0，`realloc` 只保证原有内容，**新扩部分不初始化**。
4. `realloc` 后要 free **新指针**，不要 free 旧指针（可能已经被搬走释放）。
5. `new` 失败**抛 `bad_alloc`**，不是返回 `NULL`；`malloc` 失败才返回 `NULL`。判空是 `malloc` 的习惯，`new` 要 `try/catch`。
6. **`new[]` 必须配 `delete[]`**，`delete` 也不行、`free` 更不行；`delete[]` 靠数组前面的 **cookie** 知道析构几次。
7. **`new` = `operator new` + 构造**；**`delete` = 析构 + `operator delete`**；`operator new/delete` 本身不调构造析构。
8. 对自定义类型，`malloc + free` 会**跳过构造和析构**；对象内部还有资源（如 `Stack` 里的 `_a`）时会泄漏。
9. 不能显式调用构造函数（`p->A(1)` 报错），但**可以显式调用析构函数**（`p->~A();`）。
10. 定位 new 需要 `#include <new>`，语法是 `new (ptr) T(args);`，只构造、不开空间。
11. 基类析构函数**不是 `virtual`** 时，`delete` 基类指针不会调派生类析构 → 资源泄漏（准备被继承的类必须写 `virtual ~Base() {}`）。
12. 池化技术（内存池 / 线程池 / 连接池）的共同思想是**批量申请、复用、减少系统开销**；STL 的 `allocator` 就是最典型的内存池。
13. 栈的生长方向是**由高到低**，堆是**由低到高**，两者相向增长（中间隔着共享库区）。
