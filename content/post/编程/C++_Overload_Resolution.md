---
title: "C++ Overload Resolution"
description: 
date: 2026-06-21T21:20:22+08:00
categories:
    - "编程"
tags:
    - "C++"
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

### 构建候选函数集 (Candidate Functions)

当编译器遇到一个函数调用 f(args...) 时，首先通过**名字查找（Name Lookup）**（包括常规查找和 ADL 依赖于实参的名字查找）收集所有同名的函数。

- **模板推导与 SFINAE**：如果候选集中有函数模板，编译器会尝试进行模板实参推导。如果推导失败或替换失败（SFINAE），该模板会被静默剔除，不会报错。
- **C++20 重写候选 (Rewritten Candidates)**：对于比较运算符（如 == 或 <=>），编译器还会自动生成参数反转的候选函数（例如 a == b 会生成 b == a 的候选）



### 筛选可行函数 (Viable Functions)

从候选函数集中，编译器会筛选出**可行函数**。一个函数要成为可行函数，必须满足：

1. **参数数量匹配**：参数个数必须与调用时的实参个数一致（考虑默认参数和可变参数包）。
2. **可转换性**：每个实参都必须能够通过**隐式转换序列（ICS）**转换到对应的形参类型。
3. **C++20 约束检查 (Constraints)**：如果函数带有 requires 子句（Concepts），则实参必须满足该约束。不满足约束的函数会被直接剔除。



### 决出最佳可行函数 (Best Viable Function) 

**基本博弈原则**：函数 F1 比 F2 更好，当且仅当对于**所有**参数，F1 的隐式转换序列都不比 F2 差，并且**至少有一个**参数的转换序列 F 严格优于 F2。如果转换序列一样好，则进入**平局打破规则（Tie-breakers）**

以下是完整的优先级降序排列（从最强到最弱）：

#### 规则 1：隐式转换序列 (ICS) 级别比拼

对于单个参数，转换序列的优先级严格如下：
**标准转换序列 (SCS) > 用户定义转换序列 (UDS) > 省略号转换序列 (...)**

**1.1 标准转换序列 (SCS) 内部的优先级：**

- **精确匹配 (Exact Match)**：类型完全一致。仅发生左值到右值 (Lvalue-to-Rvalue)、数组到指针 (Array-to-Pointer)、函数到指针 (Function-to-Pointer) 的退化。仅添加顶层 const/volatile 限定符。
- **提升 (Promotion)**：整型提升：short, char, bool `→→` int。浮点提升：float `→→` double。
- **转换 (Conversion)**：整型转换：int `→→` short，int `→→` unsigned int。浮点转换：double `→→` float。浮点与整型互转：int `↔↔` double。指针转换：Derived* `→→` Base\*，T* `→→` void*。布尔转换：指针/整型 `→→` bool。

**1.2 标准转换序列的“平局打破” (Tie-breakers within SCS)：**
如果两个函数都属于“精确匹配”或都属于“转换”，编译器会继续细分：

- **引用绑定规则 (Reference Binding)**：**右值优先绑定到右值引用**：对于右值实参，绑定到 T&& 优于绑定到 const T&（C++11 引入，移动语义的基础）。**左值优先绑定到左值引用**：对于左值实参，绑定到 T& 优于绑定到 const T&。
- **CV 限定符规则**：绑定到**更少** cv 限定符的类型更好。例如：实参为 int，形参 int& 优于 const int&；int* 优于 const int*。
- **类继承体系转换 (Derived-to-Base)**：转换到**血缘更近**的基类更好。例如：Derived `→→` Base1 `→→` Base2。实参为 Derived 时，形参 Base1* 优于 Base2*。



#### 规则 2：函数级别的平局打破规则

如果所有参数的 ICS 级别都完全一样好（或者无法分出胜负），编译器将动用函数级别的 Tie-breakers。**以下规则按顺序依次判定，一旦分出胜负即停止**：

**Tie-breaker 1: 非模板 优于 模板 (Non-template > Template)**

如果 F1 是普通函数，F2 是函数模板的特化版本，普通函数 F1 获胜。

```C++
void foo(int);           // F1: 普通函数
template<class T> void foo(T); // F2: 模板函数
foo(42); // 调用 F1
```

**一个黑魔法**：由于编译器在某些情况会自动生成特殊成员函数，最终如果我们只提供了模版特殊成员函数那么就可能调用这个默认版本（非模板优于模板），C++11之后当然我们都是用delete显式禁用默认版本了，不过在之前我们可以显式声明一个非模版特殊成员函数并且参数带上volatile来抑制编译器自动生成特殊成员函数，因为绝大部分情况都是nonvolatile类型，所以不用担心该版本会被调用（因为模版实例化后没有volatile更精确而由于非模版），除非你真的使用了volatile参数。

**Tie-breaker 2: 更特化的模板 优于 较泛化的模板 (More Specialized Template)**

如果 F1 和 F2 都是模板特化，编译器会执行**模板偏序（Partial Ordering）**，更特化的模板获胜。

```C++
template<class T> void ptr_func(T*);       // F1: 接受任意指针
template<class T> void ptr_func(const T*); // F2: 接受 const 指针
int x = 0;
const int* p = &x;
ptr_func(p); // 调用 F2，因为 const T* 比 T* 更特化
```

**Tie-breaker 3: 更受约束 优于 较少约束 (More Constrained) —— C++20**

如果 F1 和 F2 具有相同的参数列表，但使用了 C++20 的 Concepts，编译器会通过**约束包容（Constraint Subsumption）**规则，选择约束更严格的函数。

```C++
template<typename T> requires std::integral<T>
void print(T); // F1

template<typename T> requires std::integral<T> && (sizeof(T) == 4)
void print(T); // F2

print(42); // 调用 F2，因为 F2 的约束包容了 F1 的约束（更严格）
```

**Tie-breaker 4: 非重写候选 优于 重写候选 (Non-rewritten > Rewritten) —— C++20**

在 C++20 中，为了支持 <=> 和 == 的自动推导，引入了重写候选。如果显式定义的函数和编译器重写的函数发生冲突，显式定义的（非重写）获胜

**Tie-breaker 5: 非反转参数 优于 反转参数 (Non-reversed > Reversed) —— C++20**

如果两个都是重写候选，参数顺序没有被反转的候选优于被反转的候选。

```C++
struct A {
    bool operator==(const A&) const; // F1
};
// a == b 时，F1 (a.op==(b)) 优于 反转候选 (b.op==(a))
```

**Tie-breaker 6: 派生类构造函数 优于 继承的基类构造函数**

如果 F1 是类本身的构造函数，而 F2 是通过 using Base::Base; 继承来的构造函数，类本身的构造函数获胜。

**C++23 (显式对象参数 Deducing this)**：
C++23 允许使用 this auto&& self 作为成员函数的第一个参数。在重载决议中，这个显式对象参数被**完全视为一个普通参数**参与 ICS 排名，消除了以往“隐式对象参数（Implicit Object Parameter）不参与某些类型转换”的特殊历史遗留规则，使得成员函数和非成员函数的重载决议逻辑彻底统一。

(注：constexpr *和* consteval 关键字不参与重载决议的优先级排名。它们只在决议完成后，检查所选函数在当前上下文中是否合法。)



### 列表初始化

当使用大括号 {} 进行初始化时，C++ 标准规定了一套极其特殊的重载决议规则。

**std::initializer_list 具有最高优先级**：
如果实参是大括号列表，且存在形参为 std::initializer_list\<T> 的构造函数/函数，只要列表中的元素能隐式转换到 T，该函数**无条件获胜**（即使其他构造函数看起来是更完美的精确匹配）

```C++
struct Widget {
    Widget(int, int); // F1
    Widget(std::initializer_list<int>); // F2
};
Widget w{1, 2}; // 绝对调用 F2！这是 Modern C++ 中最容易踩坑的地方。
```

如果不存在 initializer_list 匹配，编译器才会退化去尝试匹配普通的构造函数（此时 {1, 2} 会被拆解为两个独立的参数参与常规重载决议）。



### 二义性 (Ambiguity)

如果编译器走完了上述**所有**规则（从 ICS 级别到所有的 Tie-breakers），发现依然有两个或多个函数并列第一（例如：一个函数在参数 A 上转换更好，另一个函数在参数 B 上转换更好），编译器将放弃挣扎，抛出 **Ambiguous call（二义性调用）** 编译错误。此时开发者必须通过显式类型转换（static_cast）或显式指定模板参数来手动打破僵局。







