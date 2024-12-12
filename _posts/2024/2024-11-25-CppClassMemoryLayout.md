---
layout: post
title:  "C++ 中类的内存布局"
date:   2024-11-25 16:26:00 +800
category: C/C++
---

- [1. 空的类](#1-空的类)
- [2. 静态成员变量](#2-静态成员变量)
- [3. 虚函数](#3-虚函数)
- [4. 单继承](#4-单继承)
- [5. 多继承](#5-多继承)
  - [5.1. 多继承中的菱形继承问题](#51-多继承中的菱形继承问题)
- [6. 子类中重写了父类的虚函数](#6-子类中重写了父类的虚函数)
- [7. 父类中没有虚函数，而子类中有虚函数](#7-父类中没有虚函数而子类中有虚函数)


本篇笔记使用 Clang 编译器的：

```zsh
clang -Xclang -fdump-record-layouts -c source_file.cpp
```

来查看类的内存布局，本机设备是 Apple M1 Max Mackbook。

## 1. 空的类

**空的类在内存中占用 1 个字节** ，用以确保每个对象都有一个唯一的内存地址。

```cpp
#include <iostream>

class A { };

int main() {
    std::cout << sizeof(A) << std::endl;
    return 0;
}
```

空类 A 的内存布局：

```zsh
*** Dumping AST Record Layout
         0 | class A (empty)
           | [sizeof=1, dsize=1, align=1,
           |  nvsize=1, nvalign=1]
```

其中：

- `sizeof=1`: 表示类的总大小为 1 字节
- `dsize=1`: 表示动态大小（dsize）为 1 字节。对于没有虚函数的普通类，`sizeof` 和 `dsize` 通常是相同的。
- `align=1`: 表示类的对齐要求为 1 字节。这意味着类的实例可以在任何字节边界上开始。
- `nvsize=1`: 表示非虚大小（nvsize）为 1 字节。对于没有虚继承的类，`nvsize` 和 `sizeof` 通常是相同的。
- `nvalign=1`: 表示非虚对齐（nvalign）为 1 字节。这是对类在内存中对齐的要求。

## 2. 静态成员变量

**静态成员变量存储在静态内存中，不在每个对象的内存布局中** 。

```cpp
#include <iostream>

class A {
public:
    int i_a;
    static int s_a;
};

int main() {
    std::cout << sizeof(A) << std::endl;
    return 0;
}
```

类 A 的内存布局：

```zsh
*** Dumping AST Record Layout
         0 | class A
         0 |   int i_a
           | [sizeof=4, dsize=4, align=4,
           |  nvsize=4, nvalign=4]
```

## 3. 虚函数

**类中如果有虚函数，则在前 4 个字节（ 32 位系统）或前 8 个字节（ 64 位系统）有一个虚指针（ vtable pointer ）** 。

```cpp
#include <iostream>

class A {
public:
    int i_a;
    static int s_a;
    virtual void VFunc_A() { }
};

int main() {
    std::cout << sizeof(A) << std::endl;
    return 0;
}
```

类 A 的内存布局：

```zsh
*** Dumping AST Record Layout
         0 | class A
         0 |   (A vtable pointer)
         8 |   int i_a
           | [sizeof=16, dsize=12, align=8,
           |  nvsize=12, nvalign=8]
```

## 4. 单继承

内存布局顺序：

- 虚指针（如果父类有虚函数的话）
- 父类的成员变量（按声明顺序）
- 子类的成员变量（按声明顺序）

```cpp
#include <iostream>

class A {
public:
    int i_a;
    static int s_a;
    virtual void VFunc_A() { }
};

class B : public A {
public:
    int i_b;
};

int main() {
    std::cout << sizeof(B) << std::endl;
    return 0;
}
```

子类 B 的内存布局：

```zsh
*** Dumping AST Record Layout
         0 | class B
         0 |   class A (primary base)
         0 |     (A vtable pointer)
         8 |     int i_a
        12 |   int i_b
           | [sizeof=16, dsize=16, align=8,
           |  nvsize=16, nvalign=8]
```

## 5. 多继承

内存布局顺序：

- 多个父类中有虚函数的父类在前，都有虚函数则按照子类声明的继承顺序
- 父类 1 的虚指针（ 如果父类 1 有虚函数的话 ）
- 父类 1 的成员变量（按声明顺序）
- 父类 2 的虚指针 （ 如果父类 2 有虚函数的话 ）
- 父类 2 的成员变量（按声明顺序）
- 子类的成员变量（按声明顺序）

```cpp
#include <iostream>

class A {
public:
    int i_a;
    static int s_a;
    virtual void VFunc_A() { }
};

class B {
public:
    int i_b;
    virtual void VFunc_B() { }
};

class C : public A, public B {
public:
    float f_c;
};

int main() {
    std::cout << sizeof(C) << std::endl;
    return 0;
}
```

子类 C 的内存布局：

```zsh
*** Dumping AST Record Layout
         0 | class C
         0 |   class A (primary base)
         0 |     (A vtable pointer)
         8 |     int i_a
        16 |   class B (base)
        16 |     (B vtable pointer)
        24 |     int i_b
        28 |   float f_c
           | [sizeof=32, dsize=32, align=8,
           |  nvsize=32, nvalign=8]
```

### 5.1. 多继承中的菱形继承问题

当一个类从两个父类继承，而这两个父类又从同一个父类继承时，会导致菱形继承的问题。如下面的例子，类 `B` 和类 `C` 都继承自类 `A` ，类 `D` 继承自类 `B` 和类 `C` ：

```cpp
#include <iostream>

class A {
public:
    int a;
};

class B : public A {
public:
    int b;
};

class C : public A {
public:
    int c;
};

class D : public B, public C {
public:
    int d;
};

int main() {
    std::cout << sizeof(D) << std::endl;
    return 0;
}
```

那么类 `D` 的内存布局如下：

```zsh
*** Dumping AST Record Layout
         0 | class D
         0 |   class B (base)
         0 |     class A (base)
         0 |       int a
         4 |     int b
         8 |   class C (base)
         8 |     class A (base)
         8 |       int a
        12 |     int c
        16 |   int d
           | [sizeof=20, dsize=20, align=4,
           |  nvsize=20, nvalign=4]
```

可以看到，在类 `D` 的内存布局中，存在两份父类 `A` 的成员变量 `int a` ，**菱形继承会导致冗余数据和二义性的问题**。通过在继承中通过 `virtual` 关键字使用**虚继承**可以解决菱形继承的问题：

```cpp
...

class B : virtual public A {
public:
    int b;
};

class C : virtual public A {
public:
    int c;
};

...
```

类 `D` 的内存布局如下：

```zsh
*** Dumping AST Record Layout
         0 | class D
         0 |   class B (primary base)
         0 |     (B vtable pointer)
         8 |     int b
        16 |   class C (base)
        16 |     (C vtable pointer)
        24 |     int c
        28 |   int d
        32 |   class A (virtual base)
        32 |     int a
           | [sizeof=40, dsize=36, align=8,
           |  nvsize=32, nvalign=8]
```

## 6. 子类中重写了父类的虚函数

内存布局顺序和 [多继承](#5-多继承) 中一致。

看下面的例子：

```cpp
#include <iostream>

class A {
public:
    int i_a;
    static int s_a;
    virtual void VFunc_A() { }
};

class B {
public:
    int i_b;
    virtual void VFunc_B() { std::cout << "B" << std::endl; }
};

class C : public A, public B {
public:
    float f_c;
    virtual void VFunc_B() override { std::cout << "C" << std::endl; }
};

int main() {
    std::cout << sizeof(C) << std::endl;
    C* c = new C();
    c->VFunc_B();
    return 0;
}
```

其中，子类 C 继承了父类 A 和父类 B ，并且重写了父类 B 中的虚函数 `VFunc_B` 。子类 C 的内存布局如下：

```zsh
*** Dumping AST Record Layout
         0 | class C
         0 |   class A (primary base)
         0 |     (A vtable pointer)
         8 |     int i_a
        16 |   class B (base)
        16 |     (B vtable pointer)
        24 |     int i_b
        28 |   float f_c
           | [sizeof=32, dsize=32, align=8,
           |  nvsize=32, nvalign=8]
```

当通过子类 C 的实例化对象调用 `VFunc_B` 函数时，通过父类 B 的虚指针找到父类 B 的虚函数表，从其中找到被子类 C 覆盖的 `VFunc_B` 函数地址并调用。

## 7. 父类中没有虚函数，而子类中有虚函数

父类中没有虚函数，而子类中有虚函数时，内存布局顺序：

- 子类虚指针
- 父类的成员变量（按声明顺序）
- 子类的成员变量

```cpp
#include <iostream>

class A {
public:
    int i_a;
    static int s_a;
};

class C : public A {
public:
    float f_c;
    virtual void VFunc_C() { }
};

int main() {
    std::cout << sizeof(C) << std::endl;
    return 0;
}
```

子类 C 的内存布局：

```zsh
*** Dumping AST Record Layout
         0 | class C
         0 |   (C vtable pointer)
         8 |   class A (base)
         8 |     int i_a
        12 |   float f_c
           | [sizeof=16, dsize=16, align=8,
           |  nvsize=16, nvalign=8]
```
