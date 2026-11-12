---
title: Temlate
date: 2026-11-07 22:48:38
tags: [c++, 模板, 泛型编程]
description: 'C++ 模板入门：函数模板、模板的推演与实例化、模板参数匹配原则、类模板，以及 explicit/隐式实例化'
---

> 内容来源：
> - `25年-05月-06日--类和对象+内存管理`：`std::sort` 的本质是**函数模板** + `greater<int>()` 传仿函数
> - `25年-05月-08日--内存管理+模板`：泛型编程引入、**函数模板**、隐式/显式实例化、模板与非模板函数的匹配原则、**类模板**
>
> 本文只覆盖"模板入门"。特化、非类型模板参数、模板分离编译等进阶内容属于 `25年-07月-24日-模板进阶`，另文整理。

## 目录

- **一、泛型编程：为什么需要模板**
- **二、函数模板的语法与原理**
- **三、隐式实例化与显式实例化**
- **四、模板参数的类型转换与匹配原则**
- **五、类模板（重点：`typedef` 解决不了的问题）**
- **六、模板使用中的几个硬规则**
- **七、速查表与易错清单**

---

## 一、泛型编程：为什么需要模板

先看没有模板时的"函数重载地狱"——想交换 `int`、`double`、`char`，就得写三份几乎一样的代码：

```cpp
void Swap(int& left, int& right)
{
    int temp = left;
    left = right;
    right = temp;
}

void Swap(double& left, double& right)
{
    double temp = left;
    left = right;
    right = temp;
}

void Swap(char& left, char& right)
{
    char temp = left;
    left = right;
    right = temp;
}
```

问题很明显：

1. **代码复用率低**：类型一多就要复制粘贴，逻辑完全一样。
2. **维护成本高**：逻辑改了（比如加个日志、加个断言）要改 N 个函数。
3. **可扩展性差**：自定义类型（`Date`、`ListNode`）没法预先写好重载。

> 想同时支持 `int` 和 `double`，用 `typedef double DataType;` 改一改行不行？
>
> ```cpp
> typedef double DataType;
>
> namespace bit
> {
>     class stack
>     {
>     private:
>         DataType* _a;
>         int _top;
>         int _capacity;
>     };
> }
>
> int main()
> {
>     bit::stack st1;   // 想要 int 版本
>     bit::stack st2;   // 想要 double 版本   ← 做不到！
>     return 0;
> }
> ```
>
> `typedef` 是**编译期的类型别名**，一个 `typedef` 只能指一个类型，`st1` 和 `st2` 永远是同一个类型。**想在同一份代码里同时存在"int 的 Stack"和"double 的 Stack"，只能靠类模板。**

于是有了**泛型编程**：写一份"与类型无关"的逻辑（模具），把类型本身交给编译器在使用时填入。

**模板就是泛型编程的基础。**

## 二、函数模板的语法与原理

### 2.1 语法

```cpp
// 函数模板：T 是"模板参数"，代表一个还没确定的类型
template<typename T>     // template<class T> 完全等价
void Swap(T& left, T& right)
{
    T temp = left;
    left = right;
    right = temp;
}

int main()
{
    int i = 0, j = 1;
    double x = 1.1, y = 2.2;

    Swap(i, j);    // 编译器按 int   生成一份 Swap
    Swap(x, y);    // 编译器按 double 生成一份 Swap

    return 0;
}
```

- `template<typename T>` 或 `template<class T>` 都行——**这里 `class` 不代表"类"，只是"类型"的意思**，早期标准用的是 `class`，现代写法推荐 `typename`。
- `T` 是**模板参数**（类型参数），名字随便起，习惯用 `T`、`T1`、`T2`；有多个时 `template<class T1, class T2>`（如 `Add(T1 left, T2 right)`）。
- **函数模板不是函数**，是编译器用来生成函数的"模具"。

### 2.2 原理：模板是一张蓝图

> 板书原话：**函数模板是一个蓝图，它本身并不是函数，是编译器用使用方式产生特定具体类型函数的模具。所以其实模板就是将本来应该我们做的重复的事情交给了编译器。**

```text
             实际调用                        推演 + 实例化                     生成的具体函数
┌────────────────────────┐                                    ┌─────────────────────────┐
│ double d1 = 2.0;       │                                    │ void Swap(double& a,    │
│ double d2 = 5.0;       │──┐                                 │            double& b)   │
│ Swap(d1, d2);          │  │    ┌──────────────────────┐     │ { double temp = a; ... }│
└────────────────────────┘  ├───▶│ template<title T>    │◀────┤                         │
┌────────────────────────┐  │    │ void Swap(T& a, T& b)│     └─────────────────────────┘
│ int i1 = 10;           │  │    │ { T temp = a; ... }  │     ┌─────────────────────────┐
│ int i2 = 20;           │──┼───▶│                      │◀────┤ void Swap(int& a,       │
│ Swap(i1, i2);          │  │    └──────────────────────┘     │            int& b)      │
└────────────────────────┘  │        ↑ 模具（蓝图）            │ { int temp = a; ... }   │
┌────────────────────────┐  │                                 └─────────────────────────┘
│ char a = '0';          │──┘                                 ┌─────────────────────────┐
│ char b = '9';          │                         ────────▶ │ void Swap(char& a,      │
│ Swap(a, b);            │                                    │            char& b)     │
└────────────────────────┘                                    └─────────────────────────┘
                          ▲ 实参类型推演            ▲ 编译器实例化（半自动化）
```

- 编译阶段：**根据实参的类型推演模板参数**，然后**实例化**出具体类型的函数。
- 模板把"复制粘贴 + 改类型"的活交给了编译器，所以板书叫它**半自动化**——逻辑你写一次，具体函数编译器替你生成。
- 实例化出来的函数**只生成一次**（同一份模板 + 同一套类型 → 同一份代码，不会重复膨胀）。

## 三、隐式实例化与显式实例化

```cpp
template<class T>
T Add(const T& left, const T& right)
{
    return left + right;
}

template<class T1, class T2>
void func(const T1& left, const T2& right)
{}

int main()
{
    int a1 = 10, a2 = 20;
    double d1 = 10.1, d2 = 20.1;

    // ── 隐式实例化：编译器根据实参类型推演模板参数
    cout << Add(a1, a2) << endl;          // T = int
    cout << Add(d1, d2) << endl;          // T = double

    // cout << Add(a1, d2) << endl;       // ✗ 编译报错：T 推演成 int 还是 double？歧义

    cout << Add(a1, (int)d2) << endl;     // ✓ 用户自己强转，两份实参类型一致
    cout << Add((double)a1, d2) << endl;  // ✓

    // ── 显式实例化：显式指定模板参数，语法 Add<类型>(...)
    cout << Add<int>(a1, d2) << endl;     // T = int：d2 隐式转换成 int
    cout << Add<double>(a1, d2) << endl;  // T = double：a1 隐式转换成 double
    cout << Add<int>(d1, d2) << endl;     // T = int：两个 double 都转成 int

    return 0;
}
```

| 方式 | 语法 | 模板参数来源 | 实参是否允许隐式转换 |
| :--- | :--- | :--- | :--- |
| **隐式实例化** | `Add(a1, a2)` | 编译器**从实参类型推演** | **不允许**（`Add(a1, d2)` 直接报错） |
| **显式实例化** | `Add<int>(a1, d2)` | **尖括号里显式指定** | **允许**（普通函数调用的形参转换规则） |

> 关键差别就一句：**显式实例化之后，形参类型已经定了，剩下的就是普通的传参隐式类型转换；隐式实例化则必须让编译器能从实参唯一地推出 `T`，冲突就报错。**

## 四、模板参数的类型转换与匹配原则

同一个名字既有普通函数又有模板时，编译器怎么选？

```cpp
// 专门处理 int 的加法函数
int Add(int left, int right)
{
    return left + right;
}

// 通用加法函数（故意 +10，方便观察到底调的哪个）
template<class T>
T Add(T left, T right)
{
    return left + right + 10;
}

int main()
{
    int a1 = 10, a2 = 20;
    double d1 = 10.1, d2 = 20.1;

    cout << Add(a1, a2) << endl;      // 调普通函数 → 30
    cout << Add(d1, d2) << endl;      // 调模板实例化 → 40.2 + 10 = 50.2
    cout << Add<int>(a1, a2) << endl; // 显式指定 → 强制走模板 → 40
    return 0;
}
```

**匹配原则（背下来）：**

1. **对于非模板函数和同名的函数模板，如果其他条件都相同，编译器会优先调用非模板函数**，而不会从模板产生一个实例出来（`Add(a1, a2)` 走普通函数）。
2. **如果模板可以产生一个"更匹配"的函数**（例如 `Add(d1, d2)`，普通函数的 `int` 形参需要类型转换，而模板能实例化出精确的 `double` 版本），那么就选择模板。
3. **可以用 `Add<int>(...)` 这种显式指定模板参数的方式，强制只从模板生成**。
4. 反过来，也可以写 `Add<>(a1, a2)` 强制用模板（空尖括号，让编译器推演）。

> 一般不建议把"同名普通函数 + 同名模板"混着写，容易让调用方搞不清走了哪条路。真需要特殊处理某个类型，模板进阶里用**模板特化**才是正解。

## 五、类模板（重点：`typedef` 解决不了的问题）

### 5.1 语法

```cpp
namespace bit
{
    // 类模板：整个类里的 T 由实例化时指定
    template<class T>
    class stack
    {
    public:
        stack(int n = 4)
            : _a(new T[n])       // 类型已是 T，开空间直接用 new
            , _top(0)
            , _capacity(n)
        {
            cout << "stack(int n = 4)" << endl;
        }

        void push(const T& x)
        {
            // 上课只演示了逻辑，工程里这里必须判满扩容，否则越界写
            if (_top == _capacity)
            {
                size_t newCapacity = _capacity == 0 ? 4 : _capacity * 2;
                T* tmp = new T[newCapacity];
                for (size_t i = 0; i < _top; ++i)
                {
                    tmp[i] = _a[i];
                }
                delete[] _a;
                _a = tmp;
                _capacity = newCapacity;
            }

            _a[_top] = x;
            ++_top;
        }

        ~stack()
        {
            delete[] _a;         // 与 new[] 配对
            _a = nullptr;
            _top = 0;
            _capacity = 0;
            cout << "~stack()" << endl;
        }

    private:
        T* _a;
        int _top;
        int _capacity;
    };
}

int main()
{
    try
    {
        // 类模板必须【显式实例化】——尖括号里给类型，不能靠推演
        bit::stack<int> st1;        // int 版
        bit::stack<double> st2;     // double 版（同一个模板，两份完全独立的类）

        st1.push(1);
        st1.push(2);
        st1.push(3);

        st2.push(1.1);
        st2.push(2.2);
        st2.push(3.3);
    }
    catch (const exception& e)
    {
        cout << e.what() << endl;
    }

    return 0;
}
```

### 5.2 类模板 vs `typedef`（本节的核心对比）

```cpp
typedef double DataType;      // 只能指定一种

bit::stack st1;   // 只能是 double 版
bit::stack st2;   // 想换成 int 版？做不到
```

```cpp
template<class T>
class stack { /* ... */ };

bit::stack<int>    st1;   // ✓ int 版
bit::stack<double> st2;   // ✓ double 版
```

> **这就是类模板存在的意义：让"类"本身也变成可以带参数的模具。** 标准库里的 `vector<T>`、`list<T>`、`map<K, V>`、`stack<T>`、`priority_queue<T>` 全是类模板。

### 5.3 类模板的几个细节

1. **类模板必须显式实例化**（`stack<int>`）。因为类模板不像函数模板有实参可供推演，语法上就必须写尖括号。
2. **`stack<int>` 和 `stack<double>` 是两个完全独立的类**，各有各的静态成员，互不相干。
3. **类模板的成员函数是按需实例化的**：只有真正被调用的成员函数才会被生成，没调用的不生成（所以类模板里某些成员函数对某类型编译不过，只要不调用也能过）。
4. **类模板的声明和定义不能分离到 `.h` / `.cpp`**（会链接错误），通常整个模板写在头文件里（进阶内容细讲）。
5. 在类外定义类模板的成员函数，要带上 `template<class T>` 并且写成 `stack<T>::`：
   ```cpp
   template<class T>
   void stack<T>::push(const T& x) { /* ... */ }
   ```
6. 类模板的**静态成员**要在类外定义时，也是 `template<class T> T stack<T>::_count = 0;` 这种形式。

## 六、模板使用中的几个硬规则

1. **模板参数可以多个**：`template<class T1, class T2>`，用于实参类型不同、又不想强转的场景（例如 `void func(const T1& l, const T2& r)`）。
2. **模板不是"万能"的**：`T` 需要支持函数体里用到的操作。比如 `Add` 里写了 `left + right`，那么实例化时 `T` 必须有可用的 `+`；不支持就会编译报错（**错误信息在实例化时才出现**，所以模板报错通常又长又难读）。
3. **`std::sort` 就是函数模板**：`sort(begin, end)` 默认用 `less<>`，`sort(begin, end, comp)` 的第三个参数可以是**仿函数对象 / 函数指针 / lambda**：
   ```cpp
   int values[] = { 40, 10, 100, 90, 20, 25 };

   sort(values, values + 6);                    // 升序：内部用 less<int>
   sort(values, values + 6, greater<int>());    // 降序：greater<int>() 是匿名仿函数对象

   greater<int> gt;
   sort(values, values + 6, gt);                // 也可以先起个名字
   ```
   - `greater<int>` 是标准库的**仿函数类模板**（类里重载了 `operator()`），`greater<int>()` 是它的**匿名对象**。
   - 所以 `sort` 的"可调用对象"参数，本质上是由**类模板 + 模板参数**驱动的策略注入——这也是泛型编程优雅的地方。
4. **`typename` 的双重身份**：声明模板参数时和 `class` 等价；但在模板内部表示"依赖类型"时必须用 `typename`（如 `typename T::iterator it;`）。进阶再细讲。

## 七、速查表与易错清单

### 📊 函数模板 vs 类模板

| 对比项 | 函数模板 | 类模板 |
| :--- | :--- | :--- |
| 声明 | `template<class T> void f(T x)` | `template<class T> class C { ... };` |
| 实例化方式 | **隐式**（实参推演）/ **显式**（`f<int>(x)`） | **只能显式**（`C<int> obj;`） |
| 生成的产物 | 一个具体类型的函数 | 一个具体类型的类（独立的类型） |
| 成员函数 | — | **按需实例化**（不调用不生成） |
| 常见例子 | `std::sort`、`std::max`、`Add` | `std::vector<T>`、`std::list<T>`、`std::map<K,V>`、`std::stack<T>` |
| 能否分离编译 | 一般不行（放头文件） | 一般不行（放头文件） |

### 📊 模板参数来源

| 写法 | 名称 | 推演/指定 | 实参隐式转换 |
| :--- | :--- | :--- | :--- |
| `Add(a, b)` | 隐式实例化 | 编译器推演 | ✗ 不允许 |
| `Add<int>(a, b)` | 显式实例化 | 尖括号指定 | ✓ 允许 |
| `Add<>(a, b)` | 强制用模板 | 编译器推演 | ✗ 不允许 |
| `stack<int> st;` | 类模板显式实例化 | 尖括号指定（**必须**） | — |

### ⚠️ 易错清单

1. **模板本身不是函数/类**，是编译器生成具体函数/类的**模具（蓝图）**，只有被使用（实例化）时才有代码。
2. `template<class T>` 和 `template<typename T>` **完全等价**，`class` 在这里不代表"类"。
3. `Add(a1, d2)`（两个实参类型不同）在**隐式实例化**下**编译报错**——要么强转成同类型，要么写 `Add<int>(a1, d2)` 显式指定。
4. **同名普通函数和模板共存时，优先调普通函数**；只有模板能给出更精确匹配时才走模板；`Add<int>(...)` 可强制走模板。
5. **类模板必须显式实例化**（一定要写 `<>`），不存在"推演"这一说；`stack<int>` 与 `stack<double>` 是两个**毫无关系**的类。
6. 类模板的**成员函数按需实例化**，没被调用的成员函数不会生成，也不参与检查。
7. 类模板的声明和定义**别拆到 .h/.cpp**，否则链接时报"找不到符号"（模板进阶里有专门的"模板分离编译"）。
8. 模板中 `T` 必须支持函数体里用到的所有操作，否则**实例化时才报错**，报错信息又长又难读——这不代表编译器坏了。
9. `greater<int>()` 是**匿名对象**（`greater<int>` 是类模板），它作为 `sort` 的第三个参数传入；写成 `greater<int>`（不带括号）会报语法错误。
10. `typedef` 只能固定一个类型，**无法让同一个类同时支持多种元素类型**，这正是类模板的立足点。
