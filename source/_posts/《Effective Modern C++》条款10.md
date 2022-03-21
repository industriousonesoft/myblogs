title: 《Effective Modern C++》条款10：优先选用enum class，而非enum
date: 2019-4-21 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

众所周知，使用C++98中的枚举型别定义的枚举量，会泄漏到枚举型别所在的作用域中，这也意味着在此作用域内不能再有其他实体取相同的名称。因此，C++98中的枚举型别又称之为：不限作用域的枚举型别。

C++11为了解决这一问题，引入了限定作用域的枚举型别。由于限定作用域的枚举型别是通过”enum class“声明的，所以有时它们也被称为枚举类。

<!-- more -->

### 三个优点

C++11中限定作用域的枚举型别相较于C++98中不限定作用域的枚举型别，有三个明显的优点，分别是：

1. 解决名称冲突
    
    使用C++11限定作用域的枚举型别定义的枚举量，会被限定在枚举型别内：
    
    ```cpp
    // C++98
    enum Color { red, green, blue };   // red、green、blue的作用域和Color相同
    auto red = false;   // 编译错误！red已在当前作用域内被声明过了
    
    // C++11
    enum class Color { red, green, blue };   // red、green、blue的作用域被限定在Color内
    auto red = false;   // 编译成功，当前作用域内无“red“声明
    ```
    
2. 拒绝隐式转换
    
    使用C++11中限定作用域的枚举型别定义的枚举量，在于其他类型比较时不会被隐式转换，只能是同类型的比较。
    
    ```cpp
    // C++98
    enum Color { red, green, blue };
    Color c = red;
    if (c < 2.5) {     // 将Color型别与double型别比较时会发生隐式转换，结果为true
    	....
    }
    
    // C++11
    enum enum Color { red, green, blue };
    Color c = Color::red;
    if (c < 2.5) {     // 编译错误！不能将Color型别与double型别比较
    	....
    }
    
    // 可通过强制转换进行比较
    if (static_cast<double>(c) < 2.5) {    // 编译成功，结果为true
    	...
    }
    ```
    
3. 可进行前置声明
    
    C++11中限定作用域的枚举型别可以进行前置声明，即型别的名字可以比较其中枚举量先声明。
    
    ```cpp
    enum Color;    // 编译错误！
    enum class Color;   // 编译成功
    ```
    
    一切枚举型别在C++眼里都会由编译器来选择一个整数型别作为其底层型别。
    
    因此，C++11中限定作用域的枚举型别之所以能进行前置声明，本质上是因为其底层型别是已知的，默认为int。
    
    而C++98中不限定作用域的枚举型别处于空间优化的目的，并未为其指定底层型别。换言之，如果想让C++98中不限定作用域的枚举型别支持前置声明，只需要在前置声明时为其指定底层型别即可。
    
    ```cpp
    enum Color: uint8_t;    // 指定不限定作用域的枚举型别的底层型别为uint8_t
    ```