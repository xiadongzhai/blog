---
title: C++ 类和对象（上）：封装、this 指针、六大默认成员函数与运算符重载
date: 2026-10-24 19:30:57
tags: [c++, 类与对象]
categories:
  - c++
  - 类与对象
description: '从类的定义与封装、this 指针，到构造/析构/拷贝构造，再到运算符重载与完整日期类实战'
---

> 内容来源：
> - `25年-04月-22日--类和对象`：类与对象基础、this 指针、构造/析构
> - `25年-04月-24日--类和对象`：拷贝构造、运算符重载、日期类
> - `25年-04月-26日--类和对象`：赋值运算符重载、日期类完善（非法日期检查、负数天数、流运算符）、const 成员函数、取地址重载
> - `25年-04月-29日--类和对象`：初始化列表、类型转换与 `explicit`、编译器优化、静态成员、友元、内部类
> - `25年-05月-06日--类和对象+内存管理`：**匿名对象**、**对象拷贝时的编译器优化（专题）**（传值传参 / 传值返回 / 连续拷贝构造 + 赋值重载、VS2019 与 VS2022 的差异）
>
> 本文把五节课的内容串成一条完整主线，并修正了课堂代码里的几处隐患（闰年判断优先级、`++` 前后置签名、`Sum arr[n]` 变长数组等），相关位置都有标注。
>
> 5-6 课的后半段（虚拟进程地址空间、`new/delete`）属于内存管理，已整理进 `Management.md`。

## 目录

- **一、类的定义与封装** —— struct 升级、三种访问限定符、成员函数默认 inline
- **二、类的实例化与对象大小** —— 图纸与房子、成员函数不占对象空间、内存对齐、空类 1 字节
- **三、this 指针** —— 编译器如何处理、存在哪里、空指针调用成员函数的经典题
- **四、类的六大默认成员函数（总览）** —— 哪些是编译器白送的
- **五、构造函数** —— 四种特点、默认构造的三种形态、内置/自定义类型初始化规则
- **六、析构函数** —— 与构造函数对照、自定义类型成员的析构规则
- **七、拷贝构造函数** —— 为什么要传引用、浅拷贝灾难、深拷贝、传值/传引用返回
- **八、运算符重载** —— 规则、不能重载的 5 个运算符、前后置 `++`、流运算符、**赋值运算符重载**、取地址重载与 const 成员函数
- **九、完整日期类实战** —— `GetMonthDay`、`+=`/`+`、`-`/`-=`、日期差、**非法日期检查**、负数天数、完整测试
- **十、初始化列表** —— 为什么引用/const/无默认构造的成员必须放初始化列表、成员初始化顺序、缺省值
- **十一、类型转换与 explicit** —— 隐式类型转换、`explicit`、**匿名对象**、**对象拷贝时的编译器优化专题**（传值传参 / 传值返回 / 赋值重载、VS2019 vs VS2022）
- **十二、static 成员、友元与内部类** —— 静态成员、友元函数/友元类、内部类是外部类的友元
- **十三、从 C 到 C++：Stack 的演进与括号匹配** —— RAII 与"自动"的价值
- **十四、速查表与易错清单** —— 面试/选择题高频点

---

## 一、类的定义与封装

在 C++ 里，`struct` 得以升级，升级为类。最大的变化是：**不用再 `typedef struct Stack ST;` 这种写法**，类型名本身就是类型名；同时结构体内部可以放函数，并且支持访问权限控制。

```cpp
// C 的老写法
typedef struct Stack
{
    int* a;
    int top;
    int capacity;
} ST;

// C++ 的写法：struct 升级成类，直接就能当类型用
struct Stack
{
    void Init() { /* ... */ }   // 成员函数
private:
    int* _a;
    int _top;
    int _capacity;
};
```

### 1. 封装

C++ 一种实现封装的方式：用类把**对象的属性**与**方法**结合在一块，让对象更加完善，再通过**访问权限**选择性地把接口提供给外部用户使用。

用户只管调用接口，不需要（也不允许）直接改内部数据 —— 这就是封装带来的"高内聚、低耦合"。

### 2. 三种访问限定符

| 限定符 | 类外能否访问 | 说明 |
| :--- | :--- | :--- |
| `public` | 可以 | 对外提供的接口 |
| `protected` | 不可以 | 继承章节前与 `private` 完全等价 |
| `private` | 不可以 | 内部实现细节 |

- `public` 修饰的成员在类外可以直接被访问；`protected` 和 `private` 修饰的成员在类外不能直接被访问。
- `protected` 和 `private` 目前看起来是一样的，**以后讲到继承才能体现出它们的区别**。
- 访问权限的作用域从该访问限定符出现的位置开始，直到**下一个访问限定符**出现时为止；如果后面没有再出现访问限定符，作用域就到 `}` 即类结束。
- `class` 定义的成员如果没有访问限定符修饰，默认是 `private`；`struct` 默认是 `public`。**这也是 C++ 里 class 与 struct 的唯一实质区别**（当然继承默认权限也有差别，后面再说）。

一般成员变量都会被限制为 `private` / `protected`，需要给别人使用的成员函数才放为 `public`。

### 3. 成员函数默认 inline

**定义在类里面的成员函数默认是 `inline`**（只是"默认"，是否真正内联仍由编译器决定）。所以像 `Print`、`GetMonthDay` 这种短小函数直接写在类里，既清爽又高效。

---

## 二、类的实例化与对象大小

### 1. 什么是实例化

类就像**楼房的设计图纸**，对象就是按图纸**盖出来的房子**。图纸只有一份，房子可以有很多栋；改动某一栋房子，不影响图纸，也不影响别的房子。

**类本身不占存储空间**，只有实例化出来的对象才真正分配空间。一个类可以实例化出多个对象：

```cpp
class Date
{
public:
    void Init(int year, int month, int day)
    {
        _year = year;
        _month = month;
        _day = day;
    }

    void Print()
    {
        cout << _year << "/" << _month << "/" << _day << endl;
    }

private:
    int _year;   // 这里是"声明"，不是定义，不占空间
    int _month;
    int _day;
};

int main()
{
    Date d1;              // 类实例化对象
    Date d2;
    d1.Init(2024, 12, 1);
    d2.Init(2025, 11, 2);
}
```

### 2. 成员函数不占对象空间

`d1` 和 `d2` 的 `_year` 必须是**两块不同的空间**（各存各的值），但 `d1.Print` 和 `d2.Print` 调用的却是**同一份函数代码**，所以成员函数不需要为每个对象各存一份。

结论：

> **对象里只存成员变量，成员函数存放在公共的代码段。**
> 因此 `sizeof(Date) == 12`，而不是 12 + 函数大小。

### 3. 内存对齐：以空间换时间

```cpp
class A
{
public:
    void Print() { cout << _ch << endl; }
private:
    char _ch;   // 1 字节
    int _i;     // 4 字节
};
```

`sizeof(A)` 的结果是 **8**，不是 5。`_ch` 后面被填充了 3 个字节，为的是让 `_i` 落在 4 的整数倍地址上。

**为什么要内存对齐？** 因为 CPU 访问内存时按"块"读取，对齐后一次就能读到，不对齐可能要读两次再拼接。本质上是**以空间换时间**。具体规则与 C 语言结构体内存对齐完全一致（对齐数取"成员大小"与"默认对齐数（VS 为 8）"的较小值）。

### 4. 空类为什么是 1 个字节

```cpp
class B
{
public:
    void Print() { /* ... */ }
};

class C
{};

int main()
{
    cout << sizeof(A) << endl;   // 8
    cout << sizeof(B) << endl;   // 1
    cout << sizeof(C) << endl;   // 1

    B b1;
    B b2;
    cout << &b1 << endl;
    cout << &b2 << endl;         // 两个地址通常只差 1（因为 sizeof(B) == 1）
}
```

**没有成员变量的类对象，也要分配 1 字节，纯粹是为了占位，表示"这个对象存在"**。否则 `&b1` 和 `&b2` 就没法区分了。

> 小结：`sizeof(类)` = 成员变量按对齐规则累加的结果（空类特例为 1），**与成员函数无关**。

---

## 三、this 指针

### 1. 问题的引出

`Date` 类中 `Init` 与 `Print` 两个成员函数，函数体里并没有任何"区分对象"的代码。那当 `d1` 调用 `Init`、`Print` 时，函数是怎么知道该访问 `d1` 而不是 `d2` 的？

**C++ 给了一个隐含的 this 指针来解决这个问题。**

### 2. 编译器做了什么

- 编译器编译后，类的成员函数默认都会在形参的**第一个位置**增加一个**当前类类型的指针**，叫做 this 指针。比如 `Date` 类 `Init` 的**真实原型**是：

```cpp
void Init(Date* const this, int year, int month, int day);
```

- 类的成员函数中访问成员变量，本质都是通过 this 指针访问的。`Init` 里给 `_year` 赋值，本质是 `this->_year = year;`。
- 调用时 `d1.Init(2024, 12, 1);` 会被编译器处理成 `d1.Init(&d1, 2024, 12, 1);`。

```cpp
// 你写的
d1.Print();
d2.Print();

// 编译器处理之后的样子（示意）
void Print(Date* this)
{
    cout << this->_year << "/" << this->_month << "/" << this->_day << endl;
}
d1.Print(&d1);
d2.Print(&d2);
```

### 3. 使用规则

- C++ 规定**不能在实参和形参的位置显式地写 this 指针**（编译时编译器会处理），但是**可以在函数体内显式使用 this 指针**。
- `this` 本身是 `Date* const`（指针本身不能修改），所以 `this = nullptr;` 编译不过；但 `*this`、`this->_year` 都是合法用法。
- 实际写代码时通常隐式使用即可，显式写 `this->` 常见于两种场景：**形参与成员同名**（如 `Date(int year) { _year = year; }` 与 `Date(int _year) { this->_year = _year; }`），以及**在成员函数里返回自身**（`return *this;`，见运算符重载）。

### 4. this 存在哪里

课堂实验给出的答案是：**this 指针（严谨说是"对象的地址"）随函数参数一起压在栈上**。

`d1.Init(2024, 12, 1);` 对应的反汇编：

```asm
001361E3  push        1                 ; day  = 1
001361E5  push        0Ch               ; month= 12  (0Ch = 12)
001361E7  push        7E8h              ; year = 2024 (7E8h = 2024)
001361EC  lea         ecx,[d1]          ; ecx = &d1  ← this
001361EF  call        Date::Init (0131451h)
```

可以看到参数遵循 `__cdecl` 的**从右往左**压栈，而对象地址放进了 `ecx`。

> **注意（vs 的优化）**：按照标准模型 this 是第一个参数，但在 VS 的 `__thiscall` 调用约定下，为了效率**会把 this 存到 ecx 寄存器里**，不再额外压栈。所以"this 在栈上"是概念模型，实际要看编译器的实现。若函数被内联或优化掉，栈上的痕迹还会消失。

### 5. 经典面试/选择题：空指针调用成员函数

```cpp
class A
{
public:
    void Print()
    {
        cout << "A::Print()" << endl;
    }
private:
    int _a;
};

int main()
{
    A* p = nullptr;
    p->Print();   // ① 正常运行
    return 0;
}
```

```cpp
class A
{
public:
    void Print()
    {
        cout << "A::Print()" << endl;
        cout << _a << endl;   // ← 等价于 this->_a
    }
private:
    int _a;
};

int main()
{
    A* p = nullptr;
    p->Print();   // ② 运行崩溃
    return 0;
}
```

**答案：① 正常运行；② 运行崩溃（不是编译报错）。**

原因：成员函数的调用**不依赖对象是否存在**（函数的地址在编译期就确定了，`p->Print()` 并不解引用 `p`，只是把 `p` 当作 this 传进去）。所以只要函数体内**没有通过 this 访问成员变量**，就能跑完；一旦访问 `_a`，就是对空指针 `((A*)nullptr)->_a` 解引用，直接崩溃。

反过来再看：

```cpp
A aa;
A* p = &aa;
p->Print();     // 等价于 aa.Print()，等价于 Print(&aa)
(*p).Print();   // 同上
```

---

## 四、类的六大默认成员函数（总览）

**默认成员函数 = 用户没有显式实现，编译器会自动生成的成员函数。**

一个类在我们什么都不写的情况下，编译器默认会生成以下 **6 个**默认成员函数：

| 分类 | 成员函数 | 作用 |
| :--- | :--- | :--- |
| 初始化与清理 | **构造函数** | 主要完成初始化工作（替代 C 里的 `Init`） |
| 初始化与清理 | **析构函数** | 主要完成清理工作（替代 C 里的 `Destroy`） |
| 拷贝复制 | **拷贝构造函数** | 使用同类对象初始化创建对象 |
| 拷贝复制 | **赋值运算符重载**（`operator=`） | 把一个对象赋值给另一个对象 |
| 取地址重载 | `operator&` | 普通对象取地址，很少自己实现 |
| 取地址重载 | `const operator&` | `const` 对象取地址，很少自己实现 |

要点：

- 这 6 个里**最重要的是前 4 个**；最后两个（取地址重载）重不重要，稍微了解即可。
- C++11 以后还会增加**移动构造**和**移动赋值**两个默认成员函数，后面再讲。
- 学习默认成员函数要从两个方向入手：
  1. **我们不写时，编译器默认生成的函数行为是什么，是否满足我们的需求？**
  2. **编译器生成的函数不满足需求时，我们自己怎么实现？**

> 课堂上的比喻：前 4 个默认成员函数把"半自动"变成了"全自动" —— 以前要手动调 `STInit`、`STDestroy`，现在对象一造出来就自动初始化，一出作用域就自动清理。

---

## 五、构造函数

### 💡 一句话理解

构造函数虽然名叫"构造"，但它的主要任务**不是开空间创建对象**，而是**对象实例化时初始化对象**。

局部对象的空间是在**栈帧创建时**就开好的，本来不需要构造函数来"造空间"。构造函数的本质是**替代以前 C 语言里写的 `Init()` 函数**，并且由系统**自动调用**，从而保证"对象一旦被造出来，就一定是被初始化过的"。

### 📝 核心特点（基础规则）

1. **函数名**：与**类名**相同。
2. **返回值**：**无返回值**（啥都不用写，连 `void` 都不需要写，C++ 规定如此）。
3. **自动调用**：对象实例化（创建）时，系统会自动调用对应的构造函数。
4. **可以重载**：构造函数可以有多个（参数不同即可）。

```cpp
class Date
{
public:
    Date()                       // 无参构造
    {
        _year = 1;
        _month = 1;
        _day = 1;
    }

    Date(int year, int month, int day)   // 带参构造
    {
        _year = year;
        _month = month;
        _day = day;
    }

    void Print()
    {
        cout << this->_year << "/" << this->_month << "/" << _day << endl;
    }

private:
    int _year;
    int _month;
    int _day;
};

int main()
{
    Date d1;                 // 调用无参构造
    d1.Print();              // 1/1/1
    Date d2(2025, 4, 22);    // 调用带参构造
    d2.Print();              // 2025/4/22
    return 0;
}
```

> 小坑提醒：`Date func();` 是**函数声明**，不是定义一个对象。想定义无参对象就写 `Date d1;`。

### ⚠️ 默认构造函数（核心易错考点）

#### 1. 编译器的自动生成规则

- **不写就有**：如果类中没有显式定义构造函数，C++ 编译器会自动生成一个**无参的默认构造函数**。
- **写了就没**：一旦用户显式定义了任何一个构造函数，编译器将**不再**自动生成无参构造。

#### 2. 什么是"默认构造函数"？

> **核心定义：不传实参就可以调用的构造，就叫默认构造。**

**三种形态（有且只有一个存在，不能同时存在）**：

1. 编译器默认生成的无参构造。
2. 我们自己写的**无参**构造。
3. 我们自己写的**全缺省**构造（例如 `Date(int y = 1, int m = 1, int d = 1)`）。

**注意歧义**：无参构造和全缺省构造虽然构成重载，但如果同时存在，无参调用 `Date d1;` 时会产生**歧义**（编译器不知道调哪个），直接编译报错。

很多同学会误认为"默认构造函数"是编译器默认生成的那个，实际上**无参构造、全缺省构造也都是默认构造**。总结一句话：**不传实参就可以调用的构造，就叫默认构造。**

```cpp
class Date
{
public:
    // 全缺省构造：既是有参构造，也是默认构造
    Date(int year = 1, int month = 1, int day = 1)
    {
        _year = year;
        _month = month;
        _day = day;
    }

    void Print()
    {
        cout << _year << "/" << _month << "/" << _day << endl;
    }
private:
    int _year;
    int _month;
    int _day;
};

int main()
{
    Date d1;                 // 1/1/1     —— 全缺省，一个都不传也能调
    Date d2(2025);           // 2025/1/1
    Date d3(2025, 12);       // 2025/12/1
    Date d4(2025, 12, 23);   // 2025/12/23
    return 0;
}
```

> **扩展知识（C++11）**：C++11 之后支持在成员变量声明处给**缺省值**（`int _year = 1;`），编译器生成的默认构造会使用这些缺省值初始化。这是对"编译器生成的构造不初始化内置类型"这一缺陷的直接补救。

#### 3. 编译器默认生成的构造，到底干了啥？

如果你不写，让编译器自动生成一个构造函数，它的初始化规则是：

- **内置类型**（如 `int`、`char`、`double`、指针等）：**没有要求**，是否初始化是不确定的，**看编译器心情**（可能是随机值）。
- **自定义类型**（如类里包含了一个 `Stack`/`Date` 对象）：要求**调用这个成员变量的默认构造函数**来初始化。
  - *隐患：如果这个自定义类型成员**没有**默认构造函数，就会报错。此时必须用**初始化列表**来解决，这是下个章节的内容。*

```cpp
typedef int STDataType;

class Stack
{
public:
    Stack(int n = 4)          // 提供了默认构造（全缺省）
    {
        cout << "Stack(int n = 4)" << endl;
        _a = (STDataType*)malloc(sizeof(STDataType) * n);
        if (nullptr == _a)
        {
            perror("malloc申请空间失败");
            return;
        }
        _capacity = n;
        _top = 0;
    }

    ~Stack()
    {
        cout << "~Stack()" << endl;
        free(_a);
        _a = nullptr;
        _capacity = _top = 0;
    }

private:
    STDataType* _a;
    size_t _capacity;
    size_t _top;
};

// 两个栈实现一个队列
class MyQueue
{
private:
    // 自定义类型成员
    Stack _pushst;
    Stack _popst;
};

int main()
{
    MyQueue q;    // 我们不写构造，编译器生成的那个会去调 Stack 的默认构造
    Stack st;     // 显式实例化，也会自动调构造
    return 0;
}
```

输出：

```text
Stack(int n = 4)
Stack(int n = 4)
Stack(int n = 4)
```

**结论：`MyQueue` 这种"成员全是自定义类型、且这些类型都有默认构造"的类，完全不需要自己写构造函数 —— 编译器生成的就够了。** 但大多数场景（成员含内置类型）都需要我们自己实现。

#### 4. 单参数构造与隐式类型转换

如果构造函数**只有一个参数**（且未加 `explicit`），会发生**隐式类型转换**：

```cpp
class A
{
public:
    A(int x) : _x(x) {}
private:
    int _x;
};

A a1 = 10;   // 隐式转换：10 -> A(10) 构造一个临时对象，再拷贝构造 a1
A a2(10);    // 直接构造
```

工程中为防止隐蔽 Bug，**单参数构造函数强烈建议加上 `explicit`**。

---

## 六、析构函数

### 💡 一句话理解

析构函数与构造函数功能相反。它**不是**用来销毁对象本身的（局部对象的栈帧由系统自动回收），而是在对象生命周期结束时，由系统自动调用，用来**完成对象中资源的清理和释放工作**（比如释放 `malloc` / `new` 出来的内存、关闭文件）。

*注：像 `Date` 类这种没有申请额外资源的类，严格来说是不需要写析构函数的；但像 `Stack` 这种需要动态开空间的类，就必须写。*

### 📝 核心特点（基础规则）

1. **函数名**：在类名前加上字符 `~`（例如 `~Date()`）。
2. **参数与返回值**：**无参数**，**无返回值**（同样不需要加 `void`）。
3. **唯一性**：一个类**只能有一个**析构函数（不支持重载）。若未显式定义，系统会自动生成。
4. **自动调用**：对象生命周期结束时，系统会自动调用析构函数。

### ⚠️ 核心考点与注意事项

#### 1. 编译器自动生成的析构函数

- 跟构造函数类似，如果我们不写，编译器会自动生成一个。
- **做了什么**：对**内置类型**成员不做处理；对**自定义类型**成员，会自动去调用它自己的析构函数。

#### 2. 自定义类型成员的析构规则（必考）

- 只要对象生命周期结束，**自定义类型成员无论什么情况都会自动调用它的析构函数**。
- 即使你显式写了外部类的析构函数，编译器也会在外部类析构函数执行完毕后，自动去调用内部自定义类型成员的析构函数（**不需要你在外部类的析构函数里手动去调**）。

```cpp
int main()
{
    MyQueue q;   // 先构造 _pushst、_popst
    return 0;    // q 销毁：先调 q 自己的析构（编译器生成的，什么都不做）
                 // 再自动调 _popst、_pushst 的析构 → 输出两次 "~Stack()"
}
```

> **析构顺序**：同一个作用域内，对象按**构造的逆序**析构；类内部则按**成员声明的逆序**析构。

### 📊 小对比（必背）

| | 构造函数 | 析构函数 |
| :--- | :--- | :--- |
| 名字 | 与类名相同 | 类名前加 `~` |
| 参数 | 可以有参、可重载、可全缺省 | **绝无参数** |
| 重载 | **可以**重载 | **绝不重载**，只能有一个 |
| 调用时机 | 对象实例化时 | 对象生命周期结束时 |
| 是否必须写 | 建议必写（防随机值） | 只有需要释放资源时才写 |

---

## 七、拷贝构造函数

### 1. 定义

> **如果一个构造函数的第一个参数是自身类类型的引用，且任何额外的参数都有默认值，则此构造函数也叫作拷贝构造函数。** 也就是说，拷贝构造是一个特殊的构造函数。

```cpp
class Date
{
public:
    Date(int year = 1, int month = 1, int day = 1)
    {
        cout << "Date(int, int, int)" << endl;
        _year = year;
        _month = month;
        _day = day;
    }

    // 拷贝构造：d2(d1)
    Date(const Date& d)
    {
        cout << "Date(const Date& d)" << endl;
        _year = d._year;
        _month = d._month;
        _day = d._day;
    }

    ~Date()
    {
        cout << "~Date()" << endl;
    }
private:
    int _year;
    int _month;
    int _day;
};

int main()
{
    Date d1(2025, 4, 24);   // 构造
    Date d2(d1);            // 拷贝构造
    Date d4 = d1;           // 拷贝构造！（注意：不是赋值）
    return 0;
}
```

**关键区分**：`Date d4 = d1;` 是**用一个已存在的对象初始化一个新对象**，属于**拷贝构造**，不是赋值运算符。

### 2. 特点（考点密集区）

1. 拷贝构造函数是构造函数的一个**重载**。
2. 拷贝构造函数的第一个参数**必须是类类型对象的引用**；使用**传值**方式编译器直接报错，因为语法逻辑上会引发**无穷递归调用**。拷贝构造函数也可以有多个参数，但第一个参数必须是类类型对象的引用，后面的参数必须有缺省值。
3. **C++ 规定自定义类型对象进行拷贝行为必须调用拷贝构造**，所以这里**自定义类型传值传参和传值返回都会调用拷贝构造**完成。
4. 若未显式定义拷贝构造，编译器会生成**自动生成**拷贝构造函数。自动生成的拷贝构造对**内置类型**成员变量会完成**值拷贝/浅拷贝**（一个字节一个字节的拷贝），对**自定义类型**成员变量会调用他的拷贝构造。

#### 为什么必须传引用？——无穷递归

```cpp
// 错误示范
Date(Date d)      // 传值
{
    _year = d._year;
    // ...
}
```

调用 `Date d2(d1);` 时要先把实参 `d1` **传值**给形参 `d`，而**传值本身就要调用拷贝构造**，于是又要传值……形成无休止的递归，编译器直接报错。

用图表示就是：

```text
Date d2(d1)
  └─ 传值 → 需要拷贝构造 Date(Date d)
              └─ 传值 → 又需要拷贝构造 Date(Date d)
                        └─ ... 无限递归
```

改成引用就断了这个链条：

```cpp
Date(const Date& d)   // 引用是别名，不产生拷贝
{
    _year = d._year;
    _month = d._month;
    _day = d._day;
}
```

> 为什么还要加 `const`？因为拷贝一个对象不应该改变它，加 `const` 之后既能接收普通对象，也能接收 `const` 对象。**如果不加 const，`const Date d1` 就无法被拷贝。**

#### 什么时候会调用拷贝构造

```cpp
void func(Date d) { }      // 传值传参 → 拷贝构造

Date func2()
{
    Date d;
    return d;              // 传值返回 → 拷贝构造（编译器可能优化掉，见下文）
}

int main()
{
    Date d1(2025, 4, 24);
    Date d2(d1);           // ① 拷贝构造
    Date d4 = d1;          // ② 拷贝构造
    func(d1);              // ③ 传值传参 → 拷贝构造
    Date d3(&d1);          // ✗ 这是构造（Date(Date* d) 若存在），不是拷贝构造
    Date d5 = &d1;         // ✗ 同上，走指针类型的构造
    return 0;
}
```

### 3. 浅拷贝的灾难：Stack 为什么必须自己写拷贝构造

```cpp
class Stack
{
public:
    Stack(int n = 4)
    {
        _a = (int*)malloc(sizeof(int) * n);
        _capacity = n;
        _top = 0;
    }

    void Push(int x) { _a[_top++] = x; }
    int  Top() { return _a[_top - 1]; }

    ~Stack()
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
    Stack st1;
    st1.Push(1);
    st1.Push(2);
    st1.Push(3);

    Stack st2(st1);        // 编译器生成的浅拷贝：st2._a == st1._a
    st2.Top();             // 5 ？其实这里两人共用同一块堆内存
    return 0;
}
```

编译器生成的拷贝构造做的是**值拷贝**：`st2._a` 被**原封不动地拷成** `st1._a` 的地址。于是两个对象指向**同一块堆内存**，带来两个致命问题：

1. **一个修改会影响另一个**（`st2.Push` 可能和 `st1` 互相踩数据）。
2. **析构两次** —— `st1` 和 `st2` 析构时各自 `free(_a)`，同一块内存被释放两次，程序崩溃（double free）。

**解决办法：显式实现深拷贝**，让每个对象拥有自己独立的一份资源。

```cpp
// 深拷贝
Stack(const Stack& s)
{
    cout << "Stack(const Stack& s)" << endl;
    _a = (int*)malloc(sizeof(int) * s._capacity);
    if (nullptr == _a)
    {
        perror("malloc fail");
        return;
    }
    memcpy(_a, s._a, sizeof(int) * s._top);   // 把数据搬过去，而不是搬指针
    _top = s._top;
    _capacity = s._capacity;
}
```

```text
浅拷贝                          深拷贝
st1._a ──┐                      st1._a ──▶ [1 2 3 _]
         ├──▶ [1 2 3 _]         st2._a ──▶ [1 2 3 _]
st2._a ──┘                      （各自一块，互不影响）
```

### 4. 什么时候需要自己写拷贝构造？

| 类的情况 | 是否要自己写拷贝构造 |
| :--- | :--- |
| 成员全是**内置类型**且**没有指向任何资源**（如 `Date`） | **不需要**，编译器生成的够用 |
| 成员**指向了资源**（如 `Stack` 的 `_a`，malloc/new 出来的） | **必须自己写**深拷贝 |
| 成员全是**自定义类型**且这些类型自己管好了资源（如 `MyQueue` 里的两个 `Stack`） | **不需要**，编译器生成的会去调成员的拷贝构造 |
| 类**显式实现了析构函数**（说明自己有资源要释放） | **几乎一定要写**拷贝构造 |

> 最后一条是**极其实用的小技巧**：**如果一个类显式实现了析构并释放资源，那他就需要显式写拷贝构造，否则就不需要。**（这正是"三法则/五法则"的雏形：需要析构 → 通常也需要拷贝构造与拷贝赋值。）

### 5. 传值返回 vs 传引用返回（野引用陷阱）

```cpp
int& f1()      // 或 int f1()
{
    int ret = 0;
    return ret;      // ret 出了作用域就销毁了
}

Stack f2()
{
    Stack st;
    st.Push(1);
    st.Push(2);
    st.Push(3);
    return st;       // 传值返回：产生临时对象 / 被编译器优化掉
}

int main()
{
    cout << f1() << endl;        // 危险！
    cout << f2().Top() << endl;  // 安全
    return 0;
}
```

- **传值返回**会产生一个**临时对象**（调用拷贝构造）；但现代编译器会做 **RVO / 拷贝省略**优化，往往直接把临时对象构造在调用者的栈位上，看不到拷贝。
- **传引用返回**，返回的是返回对象的**别名（引用）**，没有产生拷贝，**效率更高**。
- **但是**：如果返回对象是一个**当前函数局部的对象**，函数结束就销毁了，那这个引用就指向了已释放的栈空间 —— 这时的引用相当于一个**野引用**，类似**野指针**。
- 结论：**传引用返回可以减少拷贝，但一定要确保返回对象在当前函数结束后还在**，才能用引用返回。所以 `f1()` 里返回局部变量 `ret` 的引用是**未定义行为**；而返回 `*this`（生命周期由调用者管）或静态/全局对象是安全的。

---

## 八、运算符重载

### 8.1 基本规则

- 当运算符被用于**类类型**的对象时，C++ 语言允许我们通过**运算符重载**的形式指定新的含义。C++ 规定类类型对象使用运算符时，必须转换成**调用对应的运算符重载函数**，若没有对应的运算符重载，则会编译报错。
- 运算符重载是**具有特殊名字的函数**，它的名字是由 `operator` 和后面要定义的**运算符**共同构成。和其他函数一样，它也具有其**返回类型、参数列表以及函数体**。
- 重载运算符函数的**参数个数**和该运算符作用的**运算对象数量**一样多：
  - **一元运算符**有一个参数；
  - **二元运算符**有两个参数，二元运算符的**左侧运算对象**传给第一个参数，**右侧运算对象**传给第二个参数。
- 如果一个重载运算符函数是**成员函数**，则它的**第一个运算对象默认传给隐式的 this 指针**，因此运算符重载作为成员函数时，**参数比运算对象少一个**。
- 运算符重载以后，其**优先级和结合性**与对应的内置类型运算符保持一致。

```cpp
// 全局的 operator==
bool operator==(const Date& x1, const Date& x2)
{
    // 若成员变量是 private，则需配合友元，或提供 Getxxx 接口
    return x1._year == x2._year
        && x1._month == x2._month
        && x1._day == x2._day;
}
```

**成员变量是 private 时，想让全局 `operator==` 访问有两个办法**（课堂上列的三种）：

1. 提供 `Getxxx()` 函数；
2. 在类里声明为 `friend` **友元**；
3. 干脆把运算符重载**直接写成成员函数**（最常用）。

### 8.2 不能重载的运算符（选择题常考，务必记住）

```text
.      ::      sizeof      ?:      .*
```

以上 5 个运算符**不能重载**（`.*` 是成员指针解引用运算符）。

其他限制：

- **重载操作符至少有一个类类型参数**，不能通过运算符重载改变内置类型对象的含义，如 `int operator+(int x, int y)` 是**非法**的（全是内置类型）。
- **一个类需要重载哪些运算符，是看哪些运算符重载后有意义的**。比如 `Date` 类重载 `operator-` 就有意义（算日期差），但重载 `operator+` 让两个日期相加就**没有意义**（语法上可以，实践上不该做）。

```cpp
class Date
{
public:
    bool operator==(const Date& d) const { /* ... */ }

    // 有意义：日期 - 日期 = 天数（返回 int）
    int operator-(const Date& d) const;

    // 有意义：日期 + 天数 = 日期
    Date operator+(int day) const;

    // 语法上可以，但实践上没意义：日期 + 日期
    // Date operator+(const Date& d);
private:
    int _year;
    int _month;
    int _day;
};
```

### 8.3 前置 `++` 与后置 `++` 如何区分

重载 `++` 时，前置 `++` 和后置 `++` 的**运算符重载函数名都是 `operator++`**，无法很好地区分。C++ 规定：**后置 `++` 重载时，增加一个 `int` 形参**，跟前置 `++` 构成函数重载，方便区分。

```cpp
Date& operator++();       // 前置：++d
Date  operator++(int);    // 后置：d++   ← 这个 int 只是占位，编译器传 0 给它
```

> 课堂上 PPT 把两行都写成了 `Date operator++();`，属于笔误。另外还有两点工程约定：
> - 前置 `++` 返回 `Date&`（返回自己），后置 `++` 返回 `Date`（返回自增前的旧值临时对象）；
> - 编译器在调用后置版本时会给那个 `int` 形参传一个 `0`，你不需要也不能在调用处写它。

### 8.4 流插入 `<<` / 流提取 `>>`

想让 `cout << d1` 能用，就必须重载 `operator<<`。但它**不能写成成员函数**：

如果写成成员函数，`d1 << cout` 才是这个函数的调用形式（第一个运算对象是 this），而我们要的是 `cout << d1`（左侧是 `ostream`，右侧是 `Date`）。所以**必须写成全局函数**；又因为要访问 `private` 成员，通常配合**友元**：

```cpp
class Date
{
    friend ostream& operator<<(ostream& out, const Date& d);
    friend istream& operator>>(istream& in, Date& d);
    // ...
};

ostream& operator<<(ostream& out, const Date& d)
{
    out << d._year << "年" << d._month << "月" << d._day << "日";
    return out;      // 支持连续输出 cout << d1 << d2;
}

istream& operator>>(istream& in, Date& d)
{
    in >> d._year >> d._month >> d._day;
    return in;       // 支持连续输入 cin >> d1 >> d2;
}
```

### 8.5 赋值运算符重载（拷贝赋值）

**赋值运算符重载是一个默认成员函数，用于完成两个已经存在的对象直接的拷贝赋值。**

> **这里要注意跟拷贝构造区分：拷贝构造用于一个对象拷贝初始化给另一个要创建的对象。**

```cpp
Date d1(2025, 4, 24);
Date d2(d1);      // 拷贝构造：d2 这一刻才被创建
Date d3(2025, 4, 28);

d1 = d3;          // 赋值：两个对象都已存在
d1.operator=(d3); // 与上面完全等价
```

#### 特点（考点密集）

1. **赋值运算符重载是一个运算符重载，规定必须重载为成员函数**。赋值运算符重载的参数建议写成 `const 当前类类型&`，否则会传值传参会有拷贝。
2. **有返回值，且建议写成当前类类型引用**，引用返回可以提高效率；有返回值目的是为了支持**连续赋值**场景。
3. 没有显式实现时，编译器会自动生成一个**默认赋值运算符重载**，默认赋值运算符重载行为跟默认拷贝构造函数类似，对**内置类型**成员变量会完成**值拷贝/浅拷贝**（一个字节一个字节的拷贝），对**自定义类型**成员变量会调用他的赋值重载函数。
4. 像 `Date` 这样的类成员变量全是内置类型且没有指向什么资源，编译器自动生成的赋值运算符重载就可以完成需要的拷贝，所以不需要我们显式实现赋值运算符重载。像 `Stack` 这样的类，虽然也都是内置类型，但是 `_a` 指向了资源，编译器自动生成的赋值运算符重载完成的值拷贝/浅拷贝不符合我们的需求，所以需要我们**自己实现深拷贝**（对指向的资源也进行拷贝）。像 `MyQueue` 这样的类型内部主要是自定义类型 `Stack` 成员，编译器自动生成的赋值运算符重载会调用 `Stack` 的赋值重载，也不需要我们显示实现 `MyQueue` 的赋值运算符重载。
   - **小技巧**：如果一个类**显示实现了析构并释放资源**，那么他就**需要显示写赋值运算符重载**，否则就不需要。

#### 标准写法：自赋值的坑

```cpp
class Date
{
public:
    // d1 = d3
    Date& operator=(const Date& d)
    {
        if (this != &d)          // ① 防止自赋值 d1 = d1（不加也能对，但性能/健壮性差）
        {
            _year = d._year;
            _month = d._month;
            _day = d._day;
        }
        return *this;            // ② 返回 *this 以支持 d1 = d2 = d3
    }
private:
    int _year;
    int _month;
    int _day;
};
```

两个细节：

- **`if (this != &d)`**：自赋值（`d1 = d1`）时如果先 `delete` 旧资源再拷贝，就会读已释放内存，所以必须先判自赋值。`Date` 这种纯内置类型不加也没错，但**有资源的类必须加**。
- **返回 `Date&` 而不是 `Date`**：`d1 = d2 = d3` 的展开顺序是 `d1 = (d2 = d3)`，`d2 = d3` 返回的必须是"`d2` 本体"才能继续给 `d1` 赋值；若返回值（`Date`），返回的是一份临时拷贝，虽能编译但多一次拷贝。

#### 拷贝构造 vs 赋值重载（必考对比）

| | 拷贝构造 | 赋值运算符重载 |
| :--- | :--- | :--- |
| 触发时机 | **用一个已存在对象初始化一个正在创建的对象** | **两个已存在对象之间**的赋值 |
| 函数形式 | `Date(const Date& d)` | `Date& operator=(const Date& d)` |
| 是否必须为成员 | 否（是构造，可以随便写） | **必须是成员函数** |
| 返回类型 | 无 | `Date&`（支持连续赋值） |
| 编译器默认行为 | 内置类型浅拷贝 + 自定义类型调其拷贝构造 | 内置类型浅拷贝 + 自定义类型调其赋值重载 |
| 典型代码 | `Date d2(d1);` / `Date d2 = d1;` | `d1 = d3;` |

> 注意 `Date d2 = d1;` 这行**永远是拷贝构造**，不要被 `=` 迷惑 —— 判断依据是"**这个对象是不是刚刚才被创建**"。

### 8.6 取地址运算符重载与 `const` 成员函数

#### 1. const 成员函数

- 将 `const` 修饰的成员函数称为 **const 成员函数**，`const` 修饰成员函数**放到成员函数参数列表的后面**。
- `const` 实际修饰的是该成员函数**隐含的 this 指针**，表明在该成员函数中**不能对类的任何成员进行修改**。比如 `Print` 加上 `const` 后，隐含的 this 指针由 `Date* const this` 变为 `const Date* const this`。

```cpp
class Date
{
public:
    // void Print(const Date* const this) const
    void Print() const;

    // 建议：只要不改变调用对象的函数都建议加 const
    int GetMonthDay(int year, int month) const;

    bool operator==(const Date& d) const;
    Date operator+(int day) const;    // 不修改自己
    // Date& operator+=(int day);     // 修改自己 → 不能加 const
};
```

**为什么必须加？——权限放大**

```cpp
Date d1(2100, 2, 28);
d1.Print();            // &d1 -> Date*

const Date d2(2100, 2, 28);
d2.Print();            // &d2 -> const Date*
                       // 若 Print 没加 const，相当于 const Date* 传给 Date* const this
                       // 这是权限的放大，编译器直接报错
```

所以 `const` 的一个重要意义是：**让 `const` 对象也能调用成员函数**。凡是"不修改成员"的函数一律加 `const`，否则 `const` 对象和 `const` 引用什么都调不了。

`const` 对象只能调用 `const` 成员函数，非 const 对象两者都能调。

#### 2. 取地址运算符重载（了解即可，但很有意思）

`operator&` 也属于六大默认成员函数之一。一般用编译器生成的默认版本就行，但**它可以被重载**，甚至可以返回一个假地址：

```cpp
class Date
{
public:
    // 普通对象取地址
    Date* operator&()
    {
        // return this;            // 正常语义
        // return nullptr;         // 邪恶：&d1 得到空指针
        return (Date*)0x00FF1200;  // 邪恶：返回一个假的、非法地址
    }

    // const 对象取地址
    const Date* operator&() const
    {
        return (const Date*)0x00FF1240;
    }
};
```

改写之后：

```cpp
Date d1(2100, 2, 28);
cout << &d1 << endl;              // 0000000000FF1200
const Date d2(2100, 2, 28);
cout << &d2 << endl;              // 0000000000FF1240
```

> 这就是**"取地址重载"为什么也叫默认成员函数**的原因：编译器默认生成的 `operator&` 就是 `return this;`，两个版本（普通 / const）分别对应非 const 与 const 对象。
> **工程中不要这么玩**，它是理解语法机制的例子，不是能写进项目的技巧。

#### 3. `operator[]` —— 让自定义类型也支持下标

```cpp
class SeqList
{
public:
    // 返回引用，既能读也能写
    int& operator[](size_t i)
    {
        return _a[i];
    }
private:
    int _a[10];
    int _size;
};

int main()
{
    SeqList s;
    for (size_t i = 0; i < 10; i++)
    {
        // s[i] = i;
        s.operator[](i) = i;    // 与上一行完全等价
    }
}
```

**为什么必须返回引用？** 因为 `s[i] = i` 要给返回值赋值；若返回 `int`（值），改的是临时对象，原本 `_a[i]` 不会被修改，这就成了"只能读不能写"的下标。

> 顺带回顾：`a[1]` 本质就是 `*(a + 1)`，`ptr[1]` 就是 `*(ptr + 1)` —— 下标运算符的参数是偏移量，返回引用是 C++ 容器（`vector`、`string`）的标准做法。

---

## 九、完整日期类实战

这节课的核心产出就是"日期计算器"：`+ 天数`、`- 天数`、`日期 - 日期`。

课堂目标是一个"日期计算器"：

```text
开始日期  2025 年 4 月 24 日
结果日期  2025 年 7 月 3 日
结果：相差 70 天
```

### 9.1 头文件 `Date.h`

```cpp
#pragma once
#include <iostream>
#include <cassert>
using namespace std;

class Date
{
public:
    Date(int year = 1, int month = 1, int day = 1);

    void Print() const;

    // 检查日期是否合法（4-26 新增）
    bool CheckDate();

    // 某年某月有多少天
    int GetMonthDay(int year, int month) const;

    // 比较
    bool operator==(const Date& d) const;
    bool operator!=(const Date& d) const;
    bool operator<(const Date& d) const;
    bool operator<=(const Date& d) const;
    bool operator>(const Date& d) const;
    bool operator>=(const Date& d) const;

    // 日期 += 天数 / 日期 + 天数
    Date& operator+=(int day);
    Date  operator+(int day) const;

    // 日期 -= 天数 / 日期 - 天数
    Date& operator-=(int day);
    Date  operator-(int day) const;

    // 日期 - 日期（难）：返回相差天数
    int operator-(const Date& d) const;

    // 自增自减
    Date& operator++();      // 前置 ++d
    Date  operator++(int);   // 后置 d++
    Date& operator--();
    Date  operator--(int);

    // 流运算符（全局 + 友元）
    friend ostream& operator<<(ostream& out, const Date& d);
    friend istream& operator>>(istream& in, Date& d);

private:
    int _year;
    int _month;
    int _day;
};
```

> **改进点**：`Print` 与 `GetMonthDay` 都加了 `const`。它们不修改对象状态，加 `const` 后既能被普通对象调用，也能被 `const Date` 对象调用 —— 这是工程中很常见的低层 const 习惯。

### 9.2 实现 `Date.cpp`

```cpp
#include "Date.h"

Date::Date(int year, int month, int day)
{
    _year = year;
    _month = month;
    _day = day;

    // 构造时就做合法性检查：2025/2/30 这种非法日期当场暴露
    // if (this->CheckDate())
    if (!CheckDate())
    {
        cout << "非法日期->" << *this;   // *this 本体被打印，靠 operator<< 支持
    }
}

// 检查日期是否合法
bool Date::CheckDate()
{
    if (_month < 1 || _month > 12
        || _day < 1 || _day > GetMonthDay(_year, _month))
    {
        return false;
    }
    else
    {
        return true;
    }
}

void Date::Print() const
{
    cout << _year << "/" << _month << "/" << _day << endl;
}

int Date::GetMonthDay(int year, int month) const
{
    assert(month >= 1 && month <= 12);

    static int monthDayArray[13] = { 0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31 };

    // 注意括号！必须把整个闰年判断用括号包起来
    if (month == 2 && ((year % 4 == 0 && year % 100 != 0) || (year % 400 == 0)))
        return 29;
    else
        return monthDayArray[month];
}
```

> ⚠️ **课堂上这段代码有个隐蔽的 Bug**：原写法是
> `if (month == 2 && (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0))`
> 因为 `&&` 优先级高于 `||`，它实际等价于 `(month == 2 && 闰年) || (year % 400 == 0)`。
> 当 `month == 12` 且 `year % 400 == 0`（如 2000 年 12 月）时，会错误地返回 **29 天**，日期计算出错。
> 正确写法就是上面那样：**把"是 2 月"和"是闰年"两个条件都放进同一层括号里**。
>
> 另外 `monthDayArray` 用 `static` 修饰，避免每次调用都重新在栈上建立数组 —— 函数内的 `static` 变量只初始化一次，存放在静态区。

#### `+=`：不进位直接加，进位用循环

```cpp
// d1 += 100
Date& Date::operator+=(int day)
{
    if (day < 0)                 // 4-26 新增：加负数 = 减正数，避免另写一套逻辑
    {
        return *this -= -day;
    }

    _day += day;

    // 进位
    while (_day > GetMonthDay(_year, _month))
    {
        _day -= GetMonthDay(_year, _month);
        ++_month;
        if (_month == 13)
        {
            _month = 1;
            ++_year;
        }
    }

    return *this;    // 支持 d1 += 100 += 20 这样的链式调用
}
```

板书上的手推过程（`2025/4/24 + 100`）：

```text
2025 4 24
        100
-----------
2025 4 124     124 > 30(4月天数) → 124 - 30 = 94，月份 5
2025 5 94      94 > 31(5月天数) → 94 - 31 = 63，月份 6
2025 6 63      63 > 30(6月天数) → 63 - 30 = 33，月份 7
2025 7 33      33 > 31(7月天数) → 33 - 31 = 2，月份 8
2025 8 2       ← 结果：2025 年 8 月 2 日
```

#### `+`：不能改自己，先拷贝一份

```cpp
// d1 + 100
Date Date::operator+(int day) const
{
    Date tmp(*this);     // 拷贝构造出一份临时对象
    tmp += day;          // 复用 +=，避免重复写进位逻辑
    return tmp;          // 传值返回
}
```

**`+` 与 `+=` 的关键差别**：`+` 不能修改原对象，所以要先 `Date tmp(*this)`；`+=` 直接改自己并 `return *this`。

> 工程写法：`+` 复用 `+=`、`-` 复用 `-=`、`<` 复用 `==`……**用已实现的运算符实现其他运算符**，代码量减半，也不容易出现逻辑不一致。

#### `-=` / `-`：向"借位"

```cpp
Date& Date::operator-=(int day)
{
    if (day < 0)                 // 减负数 = 加正数
    {
        return *this += -day;
    }

    _day -= day;

    while (_day <= 0)
    {
        --_month;
        if (_month == 0)
        {
            _month = 12;
            --_year;
        }
        _day += GetMonthDay(_year, _month);
    }

    return *this;
}

// 日期 - 天数
// d1 - 100
Date Date::operator-(int day) const
{
    Date tmp(*this);
    tmp -= day;
    return tmp;
}
```

注意这里必须**先回退月份再借天数**：如果写成"先加天数再减月份"，遇到 3 月 31 日往前借位时就会用错月份的天数。

板书上手推的借位过程（`2025/4/26 - 100`）：

```text
2025 4 26
        -100
--------------
2025 4 -74     -74 <= 0 → 借 3 月：-74 + 31 = -43，月份 4-1 = 3
2025 3 -43     -43 <= 0 → 借 2 月：-43 + 28 = -15，月份 2（2025 非闰年，2 月 28 天）
2025 2 -15     -15 <= 0 → 借 1 月：-15 + 31 = 16，月份 1
2025 1 16      ← 结果：2025 年 1 月 16 日
```

> 课堂上还写过一版"偷懒"的 `-=`：`*this = *this - day; return *this;`，即用 `-` 实现 `-=`。
> 它能跑通，但**会产生 2 次拷贝**（一次构造 `tmp`、一次赋值回来），效率不如直接改 `_day`。
> 课堂原话：**"学习/解决问题 → 减少走捷径"**，先自己把逻辑写对，再考虑复用。

#### `日期 - 日期`：最难的一个

```cpp
int Date::operator-(const Date& d) const
{
    Date max = *this;
    Date min = d;
    int flag = 1;

    if (max < min)
    {
        max = d;
        min = *this;
        flag = -1;
    }

    int n = 0;
    while (min != max)   // 一天一天数，小日期追上大日期
    {
        ++min;
        ++n;
    }

    return n * flag;     // 保证结果是"有符号"的
}
```

这是最直观但效率最低的写法（O(天数)）。工程里更常用"转化为儒略日 / 时间戳相减"，但思路本质一样：**先确定谁大谁小，再决定结果正负**。

#### 比较运算符与自增自减

```cpp
bool Date::operator<(const Date& d) const
{
    if (_year != d._year)  return _year < d._year;
    if (_month != d._month) return _month < d._month;
    return _day < d._day;
}

bool Date::operator==(const Date& d) const
{
    return _year == d._year && _month == d._month && _day == d._day;
}

bool Date::operator<=(const Date& d) const { return *this < d || *this == d; }
bool Date::operator>(const Date& d) const  { return !(*this <= d); }
bool Date::operator>=(const Date& d) const { return !(*this < d); }
bool Date::operator!=(const Date& d) const { return !(*this == d); }

Date& Date::operator++()      // 前置
{
    *this += 1;
    return *this;
}

Date Date::operator++(int)    // 后置
{
    Date tmp(*this);
    *this += 1;
    return tmp;               // 返回自增前的旧值
}

Date& Date::operator--()      // 前置 --d
{
    *this -= 1;
    return *this;
}

Date Date::operator--(int)    // 后置 d--
{
    Date tmp(*this);
    *this -= 1;
    return tmp;
}
```

> 注意：`d1++` 调用的是 `d1.operator++(1)` 这种形式（编译器给那个 `int` 形参传一个占位值），你只写 `d1++` 就行。

### 9.3 测试 `Test.cpp`

```cpp
#include "Date.h"

void TestDate1()
{
    Date d1(2025, 4, 24);
    // Date d2(d1 + 100);
    Date d2 = d1 + 100;
    d1.Print();          // 2025/4/24  —— d1 不变（operator+ 是 const）
    d2.Print();          // 2025/8/2

    Date d3 = d1 += 100;
    d1.Print();          // 2025/8/2   —— d1 被改了
    d3.Print();          // 2025/8/2
}

void TestDate2()
{
    Date d1(2025, 4, 26);
    Date d2 = d1 -= 10000;      // 往前退一年半多，跨年份借位
    d1.Print();
    d2.Print();

    Date d3(2025, 4, 26);
    Date d4 = d3 - 10000;
    d3.Print();
    d4.Print();

    Date d5(2025, 4, 26);
    d5 += -100;                 // 加负数 → 内部转成 -=
    d5.Print();                 // 2025/1/16
}

void TestDate3()
{
    Date d1(2025, 4, 26);
    Date d2 = d1++;             // 后置：d2 拿到旧值
    // Date d2 = d1.operator++(0);
    d1.Print();                 // 2025/4/27
    d2.Print();                 // 2025/4/26

    Date d3(2025, 4, 26);
    Date d4 = ++d3;             // 前置：d4 拿到新值
    d3.Print();                 // 2025/4/27
    d4.Print();                 // 2025/4/27
}

void TestDate4()
{
    Date d1(2025, 4, 26);
    Date d2(2025, 7, 1);

    cout << (d1 < d2) << endl;
    cout << (d2 < d1) << endl;
    cout << (d1 == d2) << endl;

    cout << d1 - d2 << endl;    // -66

    Date d3(2005, 4, 26);
    Date d4(2100, 1, 1);
    cout << d3 - d4 << endl;    // 负数：约 -34620
}

void TestDate5()
{
    Date d1(2100, 2, 29);       // 2100 不是闰年 → 构造里打印"非法日期"
    Date d2(2000, 2, 29);       // 2000 是闰年 → 合法

    // 流插入：cout << d1 << d2
    cout << d1 << d2;
    // operator<<(cout, d1);

    // 流提取：非法输入会一直要求重输
    cin >> d1 >> d2;
    cout << d1 << d2;

    cout << d1 - d2 << endl;
}

void TestDate6()
{
    Date d1(2100, 2, 28);
    d1.Print();

    d1 + 100;      // 不影响 d1
    d1 += 100;     // 影响 d1

    const Date d2(2100, 2, 28);
    d2.Print();

    d2 + 100;      // OK：operator+ 是 const 成员函数
    // d2 += 100;  // ✗ 不支持：operator+= 会修改对象，不能是 const

    cout << &d1 << endl;    // 默认 operator&：返回 &d1
    cout << &d2 << endl;    // 默认 const operator&
}

int main()
{
    TestDate1();
    return 0;
}
```

> 这 6 个测试函数就是课堂的验证清单，每个都盯着一个考点：`+`/`+=` 的差别、跨年借位、`+=` 负数、前后置自增、比较与日期差、非法日期与流运算符、`const` 对象的可调范围。

### 9.4 流运算符的完整实现（含输入校验）

```cpp
ostream& operator<<(ostream& out, const Date& d)
{
    out << d._year << "年" << d._month << "月" << d._day << "日" << endl;
    return out;
}

istream& operator>>(istream& in, Date& d)
{
    while (1)
    {
        cout << "请依次输入年月日:>";
        in >> d._year >> d._month >> d._day;

        if (d.CheckDate())
        {
            break;
        }
        else
        {
            cout << "非法日期，请重新输入!" << endl;
        }
    }
    return in;
}
```

`operator>>` 里那个 `while (1)` 循环是重点：**输入非法就重来**，把校验逻辑收敛到一处，调用者不用管。这也是"把不变量交给类自己维护"的思路。

> 关于 `const` 成员函数（`Print`、`operator+`、`operator==` 为什么都带 `const`，以及 `const Date` 对象为什么只能调 const 成员函数），详见第八章 8.6 节。

---

## 十、初始化列表

### 10.1 再探构造函数

之前我们实现构造函数时，初始化成员变量主要使用**函数体内赋值**。构造函数初始化还有一种方式，就是**初始化列表**。

初始化列表的使用方式是以**一个冒号开始**，接着是一个**以逗号分隔的数据成员列表**，每个"成员变量"后面跟一个**放在括号中的初始值或表达式**：

```cpp
class Time
{
public:
    Time(int hour)
        : _hour(hour)          // ← 初始化列表
    {
        cout << "Time()" << endl;
    }
private:
    int _hour;
};

class Date
{
public:
    Date(int& x, int year = 1, int month = 1, int day = 1)
        : _day(day)
        , _month(month)
        , _ref(x)
        , _i(10)
        , _t(1)                // 自定义类型成员也在这里初始化（调用 Time(int)）
    {
        // _year = year;        // 也可以在函数体内赋值，但不如初始化列表
    }
private:
    // 声明 + 缺省值（C++11）
    int _year = 2000;
    int _month;
    int _day = 1;

    int& _ref;                 // 引用成员
    const int _i = 100;        // const 成员
    Time _t;                   // 没有默认构造的自定义类型成员
};
```

### 10.2 三条硬规则

1. 每个成员变量在初始化列表中**只能出现一次**，语法理解上初始化列表可以认为是每个成员变量**定义初始化**的地方。
2. **引用成员变量、`const` 成员变量、没有默认构造的类类型变量，必须放在初始化列表位置进行初始化**，否则会编译报错。
3. C++11 支持在成员变量**声明位置给缺省值**，这个缺省值主要是给**没有显示在初始化列表初始化**的成员使用的。

#### 为什么这三种必须放初始化列表？

因为函数体内赋值是"**先定义（默认初始化）再赋值**"，而这三类成员**没有"默认初始化"这条路**：

| 成员类型 | 为什么必须用初始化列表 |
| :--- | :--- |
| 引用 `int& _ref` | 引用**必须在定义时绑定**，不能先定义再改绑 |
| `const int _i` | `const` 变量**必须在定义时初始化** |
| 无默认构造的自定义类型 `Time _t` | 函数体内无法"调用不存在的默认构造"，只能在初始化列表里指定构造参数并调用 |

### 10.3 成员初始化顺序（选择题高频）

> **初始化列表中按照成员变量在类中声明顺序进行初始化，跟成员在初始化列表出现的先后顺序无关。建议声明顺序和初始化列表顺序保持一致。**

```cpp
class A
{
public:
    A(int a)
        : _a1(a)          // 先执行（因为 _a2 声明在前？）
        , _a2(_a1)
    {}

    void Print() { cout << _a1 << " " << _a2 << endl; }

private:
    int _a2 = 2;          // ← 声明在前
    int _a1 = 2;          // ← 声明在后
};

int main()
{
    A aa(1);
    aa.Print();           // 输出：1 随机值  （不是 1 1，也不是 2 2）
}
```

**为什么？** 真正决定初始化顺序的是**声明顺序**：`_a2` 先声明，所以 `_a2(_a1)` 先执行 —— 但此时 `_a1` **还没有被初始化**（它是内置类型、声明处又没有缺省值），于是 `_a2` 拿到一个随机值；随后 `_a1` 才被赋为 `a`（1）。

所以输出是 `1 随机值`。这道题的答案也提醒我们：**初始化列表的书写顺序最好和声明顺序一致**，否则读代码的人（甚至你自己）都会被误导。

### 10.4 尽量使用初始化列表

> **尽量使用初始化列表初始化**，因为那些你**不在初始化列表初始化的成员也会走初始化列表**。如果这个成员在声明位置给了缺省值，初始化列表会用这个缺省值初始化；如果没有给缺省值，对于没有显示在初始化列表初始化的内置类型成员是否初始化取决于编译器，C++ 并没有规定；对于没有在初始化列表初始化的自定义类型成员会调用这个成员类型的默认构造函数，如果没有默认构造函数会编译错误。

一张图理清所有情况（**每个成员都要走初始化列表**，按声明顺序走）：

```text
每个成员变量
    │
    ├─ ① 在初始化列表中出现了 ─────────▶ 用括号里的值/表达式初始化
    │
    └─ ② 没在初始化列表出现
            │
            ├─ 类中声明位置有缺省值 ──▶ 按缺省值初始化
            │
            └─ 类中声明位置没有缺省值
                    ├─ 内置类型成员 ──▶ 可能初始化为随机值，也可能为默认值 0，具体看编译器
                    └─ 自定义类型成员 ─▶ 调用该成员的默认构造函数
                                         若没有默认构造 ──▶ 编译报错
```

其中 **引用成员 / const 成员 / 没有默认构造函数的类类型成员** 这一类，**必须在初始化列表初始化**，不能落到 ② 分支里。

### 10.5 初始化列表的实际应用：Stack 与链表

```cpp
typedef int STDataType;

class Stack
{
public:
    Stack(int n = 4)
        : _a((STDataType*)malloc(sizeof(STDataType) * n))
        , _top(0)
        , _capacity(n)
    {
        if (nullptr == _a)
        {
            perror("malloc申请空间失败");
            return;
        }
    }
    // ...
private:
    STDataType* _a;
    size_t _capacity;
    size_t _top;
};

class MyQueue
{
public:
    MyQueue(int n = 4)
        : _pushst(n)
        , _popst(n)      // 有默认构造的成员，也可以显式在列表里指定参数
    {}
private:
    Stack _pushst;
    Stack _popst;
    int _size = 0;       // 缺省值
};

// 单链表：头指针给缺省值空，省掉初始化
struct SListNode
{
    SListNode* _next;
    int _val;
};

class SList
{
private:
    SListNode* _head = nullptr;    // 缺省值，最省事
};

// 双向循环链表：头结点需要 malloc，只能写进初始化列表
struct DListNode
{
    DListNode* _next;
    DListNode* _prev;
    int _val;
};

class DList
{
public:
    DList()
        : _head((DListNode*)malloc(sizeof(DListNode)))
    {
        if (_head == nullptr)
        {
            // 处理失败
        }
        _head->_prev = _head;
        _head->_next = _head;
    }
private:
    DListNode* _head;
};
```

> 小结：**能用缺省值就别写初始化列表，能用初始化列表就别放函数体**。函数体内赋值是最后手段，且对引用/const/无默认构造成员**完全无效**。

### 10.6 `memset` 大法（顺带一提）

```cpp
class AA
{
public:
    AA()
    {
        memset(_ptr, 0, 40);      // 一把梭：把整个数组清零
    }
private:
    int _ptr[10];
};
```

对纯内置类型数组来说 `memset` 确实方便，但对**含有指针、虚表指针、自定义类型成员的类绝对不能用 `memset`**，会把虚表指针等一并清零，程序直接崩。初始化列表 + 缺省值才是普适方案。

---

## 十一、类型转换与 `explicit`

### 11.1 C 语言与 C++ 的类型转换

C 语言支持的转换关系：

1. 整形之间
2. 整形和浮点数之间
3. 整形和指针之间（`int` ↔ `int*`）
4. 指针和指针之间

C++ 在此基础上多了一条**双向通道**：

```text
内置类型 ──▶ 类类型     需要"以该内置类型为参数的构造函数"支持
类类型   ──▶ 类类型     需要相应的构造函数支持（如 B(const A& a)）
```

- C++ 支持**内置类型隐式类型转换为类类型对象**，需要有**相关内置类型为参数的构造函数**。
- 构造函数前面加 `explicit` 就**不再支持隐式类型转换**。
- **类类型的对象之间也可以隐式转换**，需要相应的构造函数支持。

### 11.2 隐式转换与编译器优化

```cpp
class A
{
public:
    // explicit A(int a1)        // 加了 explicit 后不再支持隐式类型转换
    A(int a1)
        : _a1(a1)
    {
        cout << "A(int a1)" << endl;
    }

    A(int a1, int a2)
        : _a1(a1), _a2(a2)
    {}

    A(const A& aa)
        : _a1(aa._a1), _a2(aa._a2)
    {
        cout << "A(const A& aa)" << endl;
    }

    void Print() { cout << _a1 << " " << _a2 << endl; }
    int Get() const { return _a1 + _a2; }

private:
    int _a1 = 1;
    int _a2 = 2;
};

class B
{
public:
    B(const A& a) : _b(a.Get()) {}     // A → B 的隐式转换靠这个
private:
    int _b = 0;
};

class Stack
{
public:
    void Push(const A& aa) {}
};

int main()
{
    // 1 构造一个 A 的临时对象，再用这个临时对象拷贝构造 aa1
    // 编译器遇到"连续构造 + 拷贝构造" → 优化为直接构造
    A aa1 = 1;
    aa1.Print();              // 1 2

    const A& r1 = 1;          // 临时对象生命周期被延长到 r1 的生命周期

    A aa3 = { 1, 1 };         // 多参数版本的隐式转换（C++11 花括号）
    const A& r2 = { 2, 2 };

    Stack st;
    A aa2(2);                 // 有名对象
    st.Push(aa2);

    st.Push(2);               // 2 隐式转换成 A 的临时对象（类型转换）
    st.Push({ 3, 4 });        // 类型转换

    B b1 = aa1;               // A → B 的隐式转换
    const B& r3 = aa1;

    return 0;
}
```

**编译器优化的证据**（Linux 下 `g++` 实测）：

```bash
[root@bite ~]# g++ test.cpp
[root@bite ~]# ./a.out
A(int a1)
1 2

[root@bite ~]# g++ test.cpp -fno-elide-constructors   # 关闭优化
[root@bite ~]# ./a.out
A(int a1)              ← 构造临时对象
A(const A& aa)         ← 用临时对象拷贝构造 aa1
1 2
```

对比可以看到：**默认情况下"构造 + 拷贝构造"被合并成了"直接构造"**（拷贝省略 / copy elision），这就是为什么 `A aa1 = 1;` 只打印一次 `A(int a1)`。

> 关键结论：
> - `A aa1 = 1;` **语义上是**"先隐式转换出临时对象，再拷贝构造"，**实际执行上是**直接构造（编译器优化）。
> - 这个优化是**允许但非强制**的（C++17 起部分场景变为强制），所以不能写依赖"拷贝构造被调用几次"的代码。
> - 这也是为什么 `Stack st; st.Push(2);` 能工作：`2` 被隐式转成一个 `A` 的临时对象，绑定到 `const A&` 形参上。
> - **加 `explicit` 就能整体禁用这条通路**，工程中单参数构造建议一律加 `explicit`。

### 11.3 `explicit` 的适用面

- 只对**单参数**（或除第一个外都有缺省值）的构造函数有意义。
- C++11 之后 `explicit` 也可以用在多参数构造函数上（如 `explicit A(int, int)`），但此时 `A aa = {1, 2};` 这种拷贝列表初始化会被禁止。
- **拷贝构造也可以加 `explicit`**，但那样 `void f(A a)` 传值调用就会失败，一般不这么写。

```cpp
class A
{
public:
    explicit A(int a1) : _a1(a1) {}
private:
    int _a1;
};

A a1 = 1;      // ✗ 编译报错：explicit 禁止隐式转换
A a2(1);       // ✓ 显式构造
A a3 = A(1);   // ✓ 显式构造出的临时对象再拷贝（优化后就是直接构造）
```

### 11.4 匿名对象（临时对象的一种）

上面 `A(1)`、`A()` 这种**没有名字的对象**就叫**匿名对象**，它的生命周期**只在当前这一行**，语句一结束立刻析构。

```cpp
class A
{
public:
    A(int a = 0) : _a(a) { cout << "A(int a)" << endl; }
    ~A() { cout << "~A()" << endl; }
private:
    int _a;
};

class Solution
{
public:
    int Sum_Solution(int n) { /* ... */ return n; }
};

int main()
{
    // 有名对象：生命周期在所在作用域内
    A aa1;
    A aa2(12);

    // 匿名对象：生命周期只在当前一行，下一行就被析构
    A();
    A(11);

    // 只调用一次成员函数，没必要专门起个名字
    Solution s1;
    cout << s1.Sum_Solution(10) << endl;        // 有名对象
    cout << Solution().Sum_Solution(10) << endl; // 匿名对象，用完即死

    // 匿名对象最常见的用法：给算法传一个"临时策略"
    int values[] = { 40, 10, 100, 90, 20, 25 };
    sort(values, values + 6);                    // 默认升序（内部用 less<int>）
    sort(values, values + 6, greater<int>());    // 降序：greater<int>() 就是一个匿名对象

    return 0;
}
```

关于 `sort` 的第三个参数，课堂演示了三种写法：

```cpp
// greater<int> 是标准库里的仿函数模板（一个重载了 operator() 的类模板）
greater<int> gt;                          // ① 有名对象
sort(values, values + 6, gt);

sort(values, values + 6, greater<int>()); // ② 匿名对象（推荐，省一行变量）
```

> 关键点：
> - 匿名对象**只活一行**，所以 `A(1);` 这一行结束就析构，想拿它做参数必须在**同一行**用掉。
> - 想让临时对象活久一点：用 **`const` 引用**绑定它，生命周期会被延长到引用的作用域结束（`const A& r = A(1);`）。
> - 匿名对象是后面"编译器优化"的主角：**凡是"构造一个临时对象 + 再拷贝一份"的场景，编译器都可能把两次合并成一次**。

### 11.5 对象拷贝时的编译器优化（5-6 课专题）

> 一句话：**现代编译器为了尽可能提高程序效率，在不影响正确性的前提下，会尽量减少传参和传返回值过程中的拷贝。**
>
> 但这种优化 **C++ 标准并没有严格规定**，各编译器自行处理。当前主流的较新编译器，一般会对"连续表达式"中的连续拷贝进行合并；一些更新更"激进"的编译器还会进行**跨行表达式**的合并优化。

为了"看见"拷贝，一定要让拷贝构造和析构都打印日志。下面这个 `A` 类就是 5-6 课的实验用类：

```cpp
class A
{
public:
    A(int a = 0)
        : _a1(a)
    {
        cout << "A(int a)" << endl;
    }

    A(const A& aa)
        : _a1(aa._a1)
    {
        cout << "A(const A& aa)" << endl;
    }

    A& operator=(const A& aa)
    {
        cout << "A& operator=(const A& aa)" << endl;
        if (this != &aa)
        {
            _a1 = aa._a1;
        }
        return *this;
    }

    ~A()
    {
        cout << "~A()" << endl;
    }

private:
    int _a1 = 1;
};

void f1(A aa) {}    // 传值传参
```

关闭优化的开关：`g++ test.cpp -fno-elide-constructors`（`elide` 就是"省略"，copy elision = 拷贝省略）。

#### 场景一：传值传参 —— 无法优化

```cpp
int main()
{
    A aa1(1);   // A(int a)
    f1(aa1);    // A(const A& aa)  ← 形参 aa 是实参 aa1 的拷贝，跑不掉
    return 0;
}
```

```bash
[root@bite ~]# g++ 5-6_test.cpp -fno-elide-constructors
[root@bite ~]# ./a.out
A(int a)
A(const A& aa)
~A()
~A()

[root@bite ~]# g++ 5-6_test.cpp        # 默认（开着优化）
[root@bite ~]# ./a.out
A(int a)
A(const A& aa)     ← 完全一样，说明这个场景本来就优化不了
~A()
~A()
```

**结论：实参本身就是个"有名对象"时，传值传参必须拷一份形参，没有优化空间。** 所以板书把它标成 ❌ **不推荐**（大对象尤其亏，应该写 `void f1(const A& aa)`）。

#### 场景二：传值传参 + 匿名对象 —— 构造 + 拷贝构造 → 直接构造

```cpp
int main()
{
    f1(A(1));   // ① 显式传一个匿名对象
    f1(1);      // ② 单参数构造支持隐式转换，1 被转成匿名对象
    return 0;
}
```

```bash
[root@bite ~]# g++ 5-6_test.cpp -fno-elide-constructors && ./a.out
A(int a)            ← 先构造临时对象
A(const A& aa)      ← 再用临时对象拷贝构造形参 aa
~A()
~A()

[root@bite ~]# g++ 5-6_test.cpp && ./a.out
A(int a)            ← 只剩一次构造：形参 aa 直接用 1 构造出来
~A()
```

**结论：`构造 + 拷贝构造` 被优化成 `直接构造`。** 语义上依然是"先造临时对象再拷贝"，但实际执行时临时对象被**省略**了，形参直接在原地构造。板书标成 ✅ **推荐**。

#### 场景三：传值返回 —— 最容易被问懵的一块

```cpp
A f2()
{
    A aa;
    return aa;      // 传值返回：语义上要用 aa 拷贝构造一个"返回的临时对象"
}

int main()
{
    f2();           // ① 返回值没人接收
    A aa2 = f2();   // ② 用返回值初始化一个新对象
    A aa3;
    aa3 = f2();     // ③ 返回值赋给已存在的对象
    return 0;
}
```

**① `f2();`（返回值不要）**

```bash
# VS2022 debug 观察结果
A()
~A()
```

> 板书的批注：**分析语义场景，`f2()` 没有接收返回值，觉得拷贝也白拷贝了，直接就不拷贝了。**

**② `A aa2 = f2();`（返回值初始化新对象）**

```bash
[root@bite ~]# g++ 5-6_test.cpp -fno-elide-constructors && ./a.out
A()                 ← f2 里的 aa 构造
A(const A& aa)      ← return aa：拷贝构造出返回的临时对象
~A()                ← 局部 aa 析构
A(const A& aa)      ← 用返回值临时对象拷贝构造 aa2
~A()                ← 返回值临时对象析构
~A()                ← aa2 析构

[root@bite ~]# g++ 5-6_test.cpp && ./a.out
A()
~A()                ← 两次拷贝全被优化掉，只剩一次构造
```

不同编译器的"激进程度"不一样：

| 编译器 / 配置 | `A aa2 = f2();` 的输出 | 优化程度 |
| :--- | :--- | :--- |
| `g++ -fno-elide-constructors` | `A()` `A(const A&)` `~A()` `A(const A&)` `~A()` `~A()` | 无优化（2 次拷贝） |
| VS2019 debug | `A()` `A(const A&)` `~A()` `~A()` | 一个表达式内连续拷贝构造 → 合并成 1 次拷贝构造 |
| VS2022 debug / g++ 默认 | `A()` `~A()` | 更激进：跨行合并，**直接变成一次构造** |

课堂还专门验证了一件很有意思的事：在 `f2()` 里和 `main()` 里分别打印对象地址：

```cpp
A f2()
{
    A aa;
    aa += 1;
    cout << "aa  " << &aa << endl;
    return aa;          // 本质就是"让 aa 直接建在 aa2 的位置上"
}

int main()
{
    A aa2 = f2();
    cout << "aa2 " << &aa2 << endl;   // 两个地址完全一样！
    return 0;
}
```

> 板书的批注：**本质只创建了 aa2，底层让 `aa` 变成 `aa2` 的引用。因为 `return aa` 与 `aa2` 在同一块空间，所以就不用拷贝了。**
>
> 这就是常说的 **RVO / NRVO（返回值优化）**：编译器把函数内的局部对象**直接构造在调用者的接收空间**上。

**③ `A aa3; aa3 = f2();`（返回值赋给已有对象）**

```bash
[root@bite ~]# g++ 5-6_test.cpp -fno-elide-constructors && ./a.out
A()                        ← aa3 默认构造（注意 _a1 的缺省值是 1）
A()                        ← f2 里的 aa
A(const A& aa)             ← return aa 拷贝出返回临时对象
~A()
A& operator=(const A& aa)  ← 临时对象赋值给 aa3
~A()

[root@bite ~]# g++ 5-6_test.cpp && ./a.out
A()
A()
A(const A& aa)
~A()
A& operator=(const A& aa)
~A()
```

> 这是一个 **"一个表达式中，连续拷贝构造 + 赋值重载"** 的场景，**无法优化**（VS2019 debug 也是如此）。
> 只有 VS2022 debug 这种更激进的编译器才对它做了优化。
> 板书批注：因为**已经存在 `aa3` 这块空间**了，函数返回的临时对象没法"建在 `aa3` 身上"，必然要拷贝/赋值一次。

**四种写法的推荐度（板书原样）**

| 写法 | 拷贝情况 | 板书评价 |
| :--- | :--- | :--- |
| `A aa1(1); f1(aa1);` | 传值传参，一次拷贝构造 | ❌ 不推荐（大对象改用 `const A&`） |
| `f1(A(1));` / `f1(1);` | 构造 + 拷贝构造 → **优化为直接构造** | ✅ 推荐 |
| `A aa2 = f2();` | 两次拷贝 → 优化成 1 次拷贝 / 直接构造 | ✅ 推荐 |
| `f2();` | 返回值无人接收 → 直接不拷贝 | —（只做实验用） |
| `A aa3; aa3 = f2();` | 拷贝构造 + 赋值重载，**不可优化** | ❌ 不推荐 |

#### 专题小结（工程结论）

1. **拷贝省略（copy elision）是"允许但非强制"的优化**，不能用它来保证程序行为。C++17 起，用**纯右值（prvalue）直接初始化对象**时的省略是**强制**的（guaranteed copy elision），而**命名返回值优化（NRVO）依然是可选优化**。
2. **同一段代码在不同编译器、不同版本、debug/release 下表现都可能不同**，所以：
   - 不要写"依赖拷贝构造被调用几次"的代码（比如把业务逻辑塞进拷贝构造里）；
   - 不要在拷贝构造里做资源"登记"类的隐含副作用，指望它一定执行。
3. **三个"最容易被优化掉"的组合**：`构造 + 拷贝构造`（传参/返回初始化）、`拷贝构造 + 拷贝构造`（传值返回）。
   **最难优化的组合**：`拷贝构造 + 赋值重载`（一个表达式里同时出现返回临时对象和赋值）。
4. **推荐写法**：
   - 传参：能传 `const T&` 就别传值；传值的场景尽量直接传匿名对象/字面值（能触发优化）。
   - 返回：优先 `T f()` 返回具名局部对象，靠 NRVO；千万别 `T& f()` 返回局部对象（野引用）。
   - 接收：**定义时直接初始化**（`A aa2 = f2();`），别先定义再赋值（`A aa3; aa3 = f2();`）。
   - C++11 之后进一步升级为**移动语义**：实在要拷贝的场景，写带移动构造/移动赋值的类，比指望编译器优化更靠谱。

---

## 十二、static 成员、友元与内部类

### 12.1 静态成员变量与静态成员函数

用 `static` 修饰的成员变量称为**静态成员变量**，用 `static` 修饰的成员函数称为**静态成员函数**。静态成员变量**必须在类外面定义并初始化**：

```cpp
class Sum
{
public:
    Sum()
    {
        _ret += _i;
        ++_i;
    }

    static int GetRet()      // 静态成员函数：没有 this 指针
    {
        return _ret;
    }
private:
    static int _i;
    static int _ret;
};

// 类外定义（声明时不能给初始值）
int Sum::_i = 1;
int Sum::_ret = 0;
```

特点：

- 静态成员变量属于**整个类**（所有对象共享），**不属于某个具体对象**，因此**不计入 `sizeof(类)`**。
- 静态成员变量**必须在类外定义**（`static int Sum::_i = 1;`），否则链接错误。
- 静态成员函数**没有 this 指针**，因此**不能访问非静态成员**，只能访问静态成员。
- 静态成员既可以用 `类名::成员` 访问，也可以用 `对象.成员` 访问。

### 12.2 经典题：统计创建了多少个类对象

> 题目（牛客 JZ64）：求 `1+2+3+...+n`，要求不能使用乘除法、`for`、`while`、`if`、`else`、`switch`、`case` 等关键字及条件判断语句（A?B:C）。

思路：循环 ✗、递归 ✗、等差数列公式 ✗（要用乘法）。那就能用**"创建 n 个对象，构造函数自动累加"** —— 让编译器帮你循环。

```cpp
class Sum
{
public:
    Sum()
    {
        _ret += _i;     // 累加
        ++_i;           // 下一个
    }

    static int GetRet() { return _ret; }

private:
    static int _i;
    static int _ret;
};

int Sum::_i = 1;
int Sum::_ret = 0;

class Solution
{
public:
    int Sum_Solution(int n)
    {
        Sum* arr = new Sum[n];     // 创建 n 个对象 → n 次构造 → 累加完成
        return Sum::GetRet();
    }
};
```

> 注意课堂上第一版写的是 `Sum arr[n];`（**变长数组**，C99 特性，C++ 标准不支持，VS 下会编译报错），后来改成 `Sum* arr = new Sum[n];`。
> 另外 `new Sum[n]` 之后**没有 `delete[]`**，这里只是做题，实际工程必须配对释放。

### 12.3 友元

**友元提供了一种突破类访问限定符封装的方式。** 友元分为：**友元函数**和**友元类**，在函数声明或者类声明的前面加 `friend`，并且把友元声明放到一个类的里面。

```cpp
class A
{
    // 友元类声明：B 是 A 的友元，B 的成员函数可以访问 A 的私有成员
    friend class B;

    // 友元函数声明
    friend void func(const A& aa);

private:
    int _a1 = 1;
    int _a2 = 2;
};

class B
{
public:
    void func1(const A& aa)
    {
        cout << aa._a1 << endl;    // OK：友元类成员可以访问 A 的私有成员
        cout << _b1 << endl;
    }
private:
    int _b1 = 3;
    int _b2 = 4;
};
```

特性（考点）：

1. **外部友元函数可访问类的私有和保护成员**，友元函数仅仅是**一种声明，他不是类的成员函数**。
2. 友元函数可以在类定义的**任何地方声明，不受类访问限定符限制**。
3. 一个函数可以是**多个类的友元函数**（比如 `operator<<` 同时是 `Date` 的友元）。
4. 友元类中的成员函数都可以是另一个类的友元函数，都可以访问另一个类中的私有和保护成员。
5. **友元类的关系是单向的，不具有交换性**，比如 `A` 类是 `B` 类的友元，但是 `B` 类不是 `A` 类的友元。
6. **友元类关系不能传递**，如果 `A` 是 `B` 的友元，`B` 是 `C` 的友元，但 `A` 不是 `C` 的友元。
7. **友元有时提供了便利，但是友元会增加耦合度，破坏了封装，所以友元不宜多用。**

> 为什么要用到友元？两个典型场景：
> 1. **流运算符重载**必须写成全局函数，又要访问私有成员 → `friend`；
> 2. **两个类需要紧密协作**（如容器与迭代器）→ `friend`。

### 12.4 内部类

> **如果一个类定义在另一个类的内部，这个内部类就叫做内部类。**

- 内部类是一个**独立的类**，跟定义在全局相比，他只是**受外部类类域限制和访问限定符限制**，所以外部类定义的**对象中不包含内部类**（`sizeof` 不含内部类）。
- **内部类默认是外部类的友元类。**
- 内部类本质也是一种封装，**当 A 类跟 B 类紧密关联，A 类实现出来主要就是给 B 类使用，那么可以考虑把 A 类设计为 B 的内部类**；如果放到 `private`/`protected` 位置，那么 A 类就是 B 的专属内部类，其他地方都用不了。

```cpp
// 刘备 客卿：A 是外部类，B 是内部类（客卿）
class A
{
private:
    static int _k;
    int _h = 1;

public:
    class B      // B 默认就是 A 的友元
    {
    public:
        void foo(const A& a)
        {
            cout << _k << endl;      // OK：访问外部类的静态成员
            cout << a._h << endl;    // OK：访问外部类对象的私有成员
        }

    private:
        int _b1 = 1;
        int _b2 = 1;
    };
};

int A::_k = 1;

int main()
{
    cout << sizeof(A) << endl;    // 4（只有 _h，不含内部类 B）
    A aa1;
    A::B bb1;                     // 内部类要用 A::B 访问

    return 0;
}
```

**用内部类重写 JZ64**（把 `Sum` 变成 `Solution` 的专属内部类）：

```cpp
class Solution
{
    class Sum     // Sum 是 Solution 的内部类，默认是 Solution 的友元
    {
    public:
        Sum()
        {
            _ret += _i;
            ++_i;
        }
    };
public:
    int Sum_Solution(int n)
    {
        Sum* arr = new Sum[n];
        return _ret;        // 作为友元，直接访问外部类的私有静态成员
    }

private:
    static int _i;
    static int _ret;
};

int Solution::_i = 1;
int Solution::_ret = 0;

int main()
{
    Solution s;
    Solution::Sum a[3];          // 外部也能用，因为 Sum 在 Solution 的 public 区
    cout << s.Sum_Solution(10) << endl;   // 1+2+...+10 = 55

    return 0;
}
```

> 对比一下：`Sum` 放在全局时能正常工作，但它"只服务于 `Solution`"；做成内部类后，**名字被藏进 `Solution` 的类域**，不污染全局命名空间 —— 这就是内部类的封装价值。
> 如果把 `class Sum` 放到 `private` 区，那么 `Solution::Sum a[3];` 这行就编译不过了，即"只给 `Solution` 自己用"。

---

## 十三、从 C 到 C++：Stack 的演进与括号匹配

### 13.1 C 版本

```c
typedef char STDataType;

typedef struct Stack
{
    STDataType* a;
    int top;
    int capacity;
} ST;

void STInit(ST* pst);
void STDestroy(ST* pst);
void STPush(ST* pst, STDataType x);
void STPop(ST* pst);
STDataType STTop(ST* pst);
bool STEmpty(ST* pst);
int  STSize(ST* pst);

void STInit(ST* pst)
{
    assert(pst);
    pst->a = NULL;
    pst->top = 0;        // top 指向栈顶数据的下一个位置
    pst->capacity = 0;
}

void STDestroy(ST* pst)
{
    assert(pst);
    free(pst->a);
    pst->a = NULL;
    pst->top = pst->capacity = 0;
}

void STPush(ST* pst, STDataType x)
{
    assert(pst);
    if (pst->top == pst->capacity)          // 满了，扩容
    {
        int newcapacity = pst->capacity == 0 ? 4 : pst->capacity * 2;
        STDataType* tmp = (STDataType*)realloc(pst->a, newcapacity * sizeof(STDataType));
        if (tmp == NULL)
        {
            perror("realloc fail");
            return;
        }
        pst->a = tmp;
        pst->capacity = newcapacity;
    }
    pst->a[pst->top] = x;
    pst->top++;
}

void STPop(ST* pst)
{
    assert(pst);
    assert(pst->top > 0);
    pst->top--;
}

STDataType STTop(ST* pst)
{
    assert(pst);
    assert(pst->top > 0);
    return pst->a[pst->top - 1];
}

bool STEmpty(ST* pst)
{
    assert(pst);
    return pst->top == 0;
}

int STSize(ST* pst)
{
    assert(pst);
    return pst->top;
}
```

### 13.2 C 版本的括号匹配

```c
bool isValid(char* s)
{
    ST st;
    STInit(&st);
    while (*s)
    {
        // 左括号入栈
        if (*s == '(' || *s == '[' || *s == '{')
        {
            STPush(&st, *s);
        }
        else // 右括号取栈顶左括号尝试匹配
        {
            if (STEmpty(&st))
            {
                STDestroy(&st);
                return false;       // 只有右括号，数量不匹配
            }

            char top = STTop(&st);
            STPop(&st);

            // 顺序不匹配
            if ((top == '(' && *s != ')')
                || (top == '{' && *s != '}')
                || (top == '[' && *s != ']'))
            {
                STDestroy(&st);
                return false;
            }
        }
        ++s;
    }

    // 栈不为空，说明左括号比右括号多，数量不匹配
    bool ret = STEmpty(&st);
    STDestroy(&st);
    return ret;
}
```

**注意 C 版本的两个痛点**：

1. 每个返回路径上都必须**手动 `STDestroy`**，漏一处就内存泄漏；
2. `STInit`、`STPush`、`STDestroy` 全靠程序员记得调，忘了就出事。

### 13.3 C++ 版本：让对象自己管自己的生命周期

```cpp
class Stack
{
public:
    Stack(int n = 4)                    // 构造：自动申请资源
    {
        _a = (STDataType*)malloc(sizeof(STDataType) * n);
        if (nullptr == _a)
        {
            perror("malloc申请空间失败");
            return;
        }
        _capacity = n;
        _top = 0;
    }

    // 深拷贝：别让两个对象抢同一块内存
    Stack(const Stack& s)
    {
        _a = (STDataType*)malloc(sizeof(STDataType) * s._capacity);
        if (nullptr == _a)
        {
            perror("malloc申请空间失败");
            return;
        }
        memcpy(_a, s._a, sizeof(STDataType) * s._top);
        _top = s._top;
        _capacity = s._capacity;
    }

    void Push(STDataType x)
    {
        // 课堂简化版未写扩容，实际工程必须判断 _top == _capacity 后 realloc
        _a[_top++] = x;
    }

    STDataType Top() const { return _a[_top - 1]; }
    bool Empty() const { return _top == 0; }

    ~Stack()                            // 析构：自动释放资源
    {
        free(_a);
        _a = nullptr;
        _capacity = _top = 0;
    }

private:
    STDataType* _a;
    size_t _capacity;
    size_t _top;
};
```

同样功能的括号匹配，用类写出来是这样：

```cpp
bool isValid(const char* s)
{
    Stack st;               // ← 自动构造，init 不用写
    while (*s)
    {
        // 左括号入栈
        if (*s == '(' || *s == '[' || *s == '{')
        {
            st.Push(*s);
        }
        else
        {
            // 右括号比左括号多，数量不匹配
            if (st.Empty())
                return false;

            char top = st.Top();
            // st.Pop();  // 课堂版本用 Top() 取值后未 Pop，实际应补上

            // 顺序不匹配
            if ((*s == ')' && top != '(')
                || (*s == '}' && top != '{')
                || (*s == ']' && top != '['))
            {
                return false;
            }
        }
        ++s;
    }

    // 栈为空返回 true，说明数量匹配；不为空说明左括号多
    return st.Empty();      // ← 自动析构，destroy 不用写
}
```

对比 C 版本，最直观的收益：

| | C 版本 | C++ 类版本 |
| :--- | :--- | :--- |
| 初始化 | 手工 `STInit(&st)` | 构造自动调用 |
| 清理 | 每个 `return` 前都要 `STDestroy` | 析构自动调用（**包括提前 return 的路径**） |
| 传参 | 必须传 `ST*`，还要 `assert(pst)` | 直接用对象 |
| 忘记清理 | 内存泄漏 | 不可能忘 |

这就是 C++ 最重要的理念 —— **RAII（Resource Acquisition Is Initialization）**：**在构造函数中获取资源，在析构函数中释放资源，把资源生命周期绑定到对象生命周期上。只要对象能正常销毁，资源就绝不泄漏。**

### 13.4 顺带一提：指向成员函数的指针

```cpp
void f() { cout << "f()" << endl; }

class A
{
public:
    void func() { cout << "A::func()" << endl; }
};

int main()
{
    void (*func1)() = f;             // 普通函数指针
    (*func1)();

    void (A::*func2)() = &A::func;    // 成员函数指针
    A aa;
    (aa.*func2)();                   // 必须绑定对象才能调用

    return 0;
}
```

成员函数指针必须**绑定一个对象**才能调用（因为需要 this），这也是 `d1.operator==(d2)` 这种"显式调用运算符重载函数"能成立的原因。

---

## 十四、速查表与易错清单

### 📊 面向对象生命周期速查表

| 特性 | 构造函数 | 析构函数 |
| :--- | :--- | :--- |
| **作用** | 初始化、申请资源 | 清理资源、防止内存泄漏 |
| **参数** | 可带参，可重载 | 无参，不可重载 |
| **是否必须写** | 建议必写（防随机值） | 只有需要释放资源时才写 |
| **虚函数** | 不能是虚函数（多态未建立） | 基类**必须**是虚函数（`virtual ~Base()`） |
| **编译器生成** | 不写则生成（不安全，内置类型不初始化） | 不写则生成（不处理自定义资源） |

### 📊 六大默认成员函数一表看清

| 成员函数 | 签名 | 编译器默认行为 | 何时必须自己写 |
| :--- | :--- | :--- | :--- |
| 构造函数 | `Date(int y = 1, int m = 1, int d = 1);` | 内置类型不初始化，自定义类型调其默认构造 | 需要定制初始化/校验（如 `CheckDate`） |
| 析构函数 | `~Date();` | 内置类型不处理，自定义类型调其析构 | 有 `malloc`/`new`/文件等资源 |
| 拷贝构造 | `Date(const Date& d);` | 内置类型逐字节浅拷贝，自定义类型调其拷贝构造 | 有资源指针（需深拷贝） |
| 赋值重载 | `Date& operator=(const Date& d);` | 同拷贝构造 | 有资源指针（需深拷贝） |
| 取地址重载 | `Date* operator&();` | `return this;` | 几乎永远不写 |
| const 取地址重载 | `const Date* operator&() const;` | `return this;` | 几乎永远不写 |

### 📊 三大函数（对象大小与函数归属）

| 问题 | 答案 |
| :--- | :--- |
| `sizeof(类)` 包含成员函数吗？ | **不包含**，成员函数在公共代码段 |
| `sizeof(类)` 包含静态成员、内部类吗？ | **都不包含** |
| 空类 `sizeof` 是多少？ | **1**，占位表示对象存在 |
| `class A { char _ch; int _i; }` 大小？ | **8**（内存对齐，以空间换时间） |
| `d1.Print()` 实际调用形式？ | `Print(&d1)`，this 由编译器隐式传入 |

### 📊 初始化列表 vs 函数体赋值

| 成员类型 | 能否用函数体赋值 | 说明 |
| :--- | :--- | :--- |
| 内置类型（`int`/`char`/指针） | 可以 | 但初始化列表效率更好 |
| 引用成员 `int&` | **不行** | 必须在定义时绑定 |
| `const` 成员 | **不行** | 必须在定义时初始化 |
| 无默认构造的自定义类型 | **不行** | 函数体内没有可调用的默认构造 |
| 有默认构造的自定义类型 | 可以 | 初始化列表里直接指定参数更优 |

### 📊 编译器默认生成 vs 需要自己实现

| 成员函数 | 什么时候不用自己写 | 什么时候必须自己写 |
| :--- | :--- | :--- |
| 构造 | 成员全是自定义类型且都有默认构造 | 成员含内置类型，或需要定制初始化 |
| 析构 | 类里没有需要释放的资源 | 有 `malloc`/`new`/文件句柄等资源 |
| 拷贝构造 | 成员不含资源指针（深拷贝无意义） | **有资源指针**（否则浅拷贝 → double free） |
| 赋值重载 | 同上 | 同上（浅拷贝 → double free） |

### 💼 工程实践补充

1. **RAII 是 C++ 资源管理的核心**：构造函数获取资源，析构函数释放资源。智能指针、`std::vector`、`std::string` 全都建立在这个机制上。
2. **单参数构造加 `explicit`**：防止 `MyClass obj = 10;` 这种隐蔽的隐式转换。
3. **低层 const**：不修改对象状态的成员函数一律加 `const`，否则 `const` 对象用不了。
4. **虚析构（必考工作题）**：如果一个类准备作为**基类**被继承，它的析构函数**必须**声明为 `virtual`。否则用基类指针 `delete` 派生类对象时只调基类析构，派生类资源泄漏。
5. **三/五法则**：只要你的类需要写析构函数（意味着自己管理了动态资源），通常也必须同时提供**拷贝构造**和**拷贝赋值**，否则会有**浅拷贝 / Double Free** 的风险。C++11 之后扩展到"五法则"（再加移动构造、移动赋值）。
6. **运算符重载要克制**：只重载**语义上说得通**的运算符（`Date - Date` 有意义，`Date + Date` 没意义）。
7. **构造里做校验**：像 `CheckDate` 一样把"合法性"收进类内部，调用者不必每次都手动验证。
8. **友元与内部类都是"降低耦合"的工具**：友元不宜多用（破坏封装），内部类适合"只服务外部类"的辅助类（如 JZ64 里的 `Sum`）。

### ⚠️ 易错清单（考前最后一遍）

1. `Date d4 = d1;` 是**拷贝构造**，不是赋值运算符（新对象初始化）。
2. 拷贝构造必须**传引用**，传值会引发**无穷递归**，编译器直接报错。
3. 编译器生成的拷贝构造是**浅拷贝（逐字节值拷贝）**，对有资源的类会**双重释放**。
4. `Date func();` 是**函数声明**，不是对象定义。
5. 无参构造 + 全缺省构造同时存在 → 无参调用**歧义**。
6. `p = nullptr; p->Print();` **不会崩**（只要函数体内不访问成员变量）；一旦访问成员就崩。
7. `this` 不能修改（`Date* const`），`this = nullptr;` 编译不过。
8. 不能重载的 5 个运算符：`. :: sizeof ?: .*`。
9. 运算符重载**至少要有一个类类型参数**。
10. 后置 `++` 靠一个**`int` 占位形参**与前置 `++` 区分。
11. 传引用返回要保证返回对象在函数结束后**仍然存在**，否则是野引用。
12. 闰年判断别忘了括号：`month == 2 && ((year % 4 == 0 && year % 100 != 0) || (year % 400 == 0))`。
13. **赋值运算符重载必须是成员函数**，且要返回 `Date&` 才支持 `d1 = d2 = d3`。
14. **成员初始化顺序只看声明顺序**，与初始化列表的书写顺序无关（`_a1/_a2` 那道题答案是 `1 随机值`）。
15. **引用成员、const 成员、无默认构造的类类型成员**必须放初始化列表，否则编译报错。
16. **静态成员变量必须在类外定义**（`int Sum::_i = 1;`），且不计入 `sizeof`。
17. **静态成员函数没有 this 指针**，不能访问非静态成员。
18. **友元关系是单向的、不能传递**，且会增加耦合。
19. **内部类默认是外部类的友元**，但外部类对象**不包含**内部类对象。
20. `Sum arr[n];` 是**变长数组**（C99 特性），C++ 标准不支持，应写 `new Sum[n]`。
21. 加 `explicit` 后 `A a1 = 1;` 会编译报错，但 `A a1(1);` 仍然可用。
22. `A aa1 = 1;` 语义是"构造临时对象 + 拷贝构造"，**实际执行是直接构造**（编译器优化，`-fno-elide-constructors` 可关闭）。
23. **匿名对象**（`A()`、`A(1)`、`greater<int>()`）的**生命周期只在当前这一行**；用 `const A& r = A(1);` 可以把它的生命周期延长到 `r` 的作用域结束。
24. `f1(aa1)` 这种**实参是有名对象**的传值传参**无法优化**，一次拷贝构造跑不掉；而 `f1(A(1))`、`f1(1)` 的"构造 + 拷贝构造"会被优化成**直接构造**。
25. `A aa3; aa3 = f2();` 是一个表达式中的 **"连续拷贝构造 + 赋值重载"，基本无法优化**；想避免就写成 `A aa3 = f2();`。
26. **拷贝省略不是语言保证**：编译器版本、debug/release 都会影响结果；C++17 起只有"用纯右值直接初始化对象"的省略是强制的，**NRVO 仍是可选优化**。所以别在拷贝构造/析构里塞依赖调用次数的业务逻辑。
