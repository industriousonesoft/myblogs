title: 《Effective Modern C++》条款23：理解std::move和std::forward
date: 2019-5-13 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

- std::move
    
    std::move，即移动语义。它的设计初衷在于让编译器用一种低成本的移动操作替换昂贵的复制操作。与之相关的两个函数分别是移动构造函数和移动赋值运算符。
    
    此外，std::move使得创建一个只可移动不可复制的对象成为可能，比如std::unique_ptr、std::future和std::thread等。
    
- std::forward
    
    std::forward，即完美转发。它的设计初衷是使对任何一个函数模板，都可以将当前函数所接受的实参原封不动地转发给其它函数，且目标函数接受到的实参与传入当前函数的实参完全相同，包括实参的左右值属性。
    
- 形参总是左值
    
    所谓左值，简单地说就是可寻址变量，而右值则是不可寻址的临时变量。
    
    ```cpp
    int a = 3;  // a是左值，3是右值
    a = a + 1;  // 等号左边的a是右值，等号右边的a是右值
    					  // CPU取a的值存入临时变量，即寄存器，然后+1，再将寄存器的值赋值给a
    ```
    
    实参既可以是左值，也可以是右值。而形参总是左值，原因在于形参是为了传递实参的值或指针或引用而出现的，因此必须是可被赋值的左值。即便形参的类别是右值引用，如下：
    
    ```cpp
    void fun(Widget&& w);  // 形参w是一个左值
    ```
    
<!-- more -->

### 详解

1. std::move
    
    std::move本质上就是使用std::remove_reference_t将传入实参的引用属性移除（但不包括const属性，移除const属性需要使用std::remove_const_t），再将其强制转换成右值引用。以下是C++14中std::move的实现：
    
    ```cpp
    template<typename T>
    decltype(auto) move(T&& param) {
    	using ReturnType = std::remove_reference_t<T>&&; // 即便param是一个左值引用，移除引用后变成左值类型
    	return static_cast<ReturnType>(param);  // 强制转换成右值引用
    }
    ```
    
    鉴于std::move不会移除实参中的const属性，因此只能保证其返回值是一个右值，但是不能保证返回值的可移动能力。比如：
    
    ```cpp
    class A {
    public:
    	A(const std::string& s);   // 复制构造函数
      A(std::string&& s);        // 移动构造函数
    };
    
    const std::string a = "hello";  // a是一个常量左值
    std::string b = "hello";  // a是一个非常量左值
    A(std::move(a));  // 调用复制构造函数
    A(std::move(b));  // 调用移动构造函数
    ```
    
2. std::forward
    
    std::forward是一种有条件的强制型别转换：仅当传入的实参是一个右值时，才会执行右值型别的强制转换。
    
    std::forward本质上就是转发实参的左右值属性。如果传入的实参是左值，那么转发之后仍是左值。如果传入的实参是右值，那么转发之后仍是右值。比如：
    
    ```cpp
    void process(const Widget& lval);  // 函数1，处理左值
    void process(Widget&& rval);  // 函数2，处理右值
    
    template<typename T>
    void logAndProcess(T&& param) {
    	..... // add a log
      process(std::forward<T>(param));  // param是一个形参，由于所有形参都是左值，理论上只会调用函数1。
    																		// 解决机制：param所接收的实参的左右属性被编码进目标参数T中，
    																		// 当param传递给std::forward后，std::forward再将编码的信息
    																		// 解码，获取传入实参的左右属性。
    }
    
    Widget w;
    logAndProcess(w);  // 传入左值，调用函数1
    logAndProcess(std::move(w));  // 传入右值，调用函数2
    ```
    

### 总结

std::move实际上不会进行任何移，而std::forward实际上也不会进行任何转发。二者都是编译期的行为，在运行期不会有任何行为，即不会生成任何可执行代码。

- 相同点
    
    二者本质上都仅是执行强制类别转换的函数模板。
    
- 不同点
    
    std::move是无条件地将实参强制转换成右值，而std::forward则是在满足特定条件下才执行同类别强制转换（即转发）。