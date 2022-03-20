title: 《Effective Modern C++》条款24：区分万能引用和右值引用
date: 2019-5-18 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

万能引用和右值引用都用”T&&“表示。

- 万能引用
    
    所谓万能引用，即它既可以绑定左值，又可以绑定右值。一般用于表示型别推导的结果。
    
- 右值引用
    
    右值引用，顾名思义就是只能绑定到右值，它的主要作用是作为可移动对象的标识。
    
<!-- more -->

### 详解

- 万能引用
    
    万能引用有两种最常见的场景，这两种场景都涉及型别推导，它们分别是：
    
    - 模板函数的形参
        
        ```cpp
        template<typename T>
        void f(T&& param);  // param是个万能引用
        
        Widget w;
        
        f(w);  // param的型别是左值引用，即Widget&
        
        f(std::move(w));  // param的型别是右值引用，即Widget&&
        ```
        
    - 用于auto声明
        
        ```cpp
        Widget w;
        
        auto&& w1 = w;  // w1是个左值引用，即Widget&
        
        auto&& w2 = std::move(w);  // w2是个右值引用，即Widget&&
        ```
        
    
     万能引用首先是一个引用，所以初始化是必须的，且初始化的对象决定了它代表的是左值还是右值引用。
    
    几个要点：
    
    - 万能引用的声明形式必须是”T&&“，且型别推导必须是涉及param本身：
        - 模板函数的形参并不一定涉及型别推导
            
            ```cpp
            template<typename T>
            void f(std::vector<T>&& param);  // param是个右值引用，
            															   // 因为型别已确定是std::vector<T>&&，
            																 // 而非T&&
            std::vector<int> v;
            f(v);             // 编译错误！不能给一个右值引用绑定一个左值
            f(std::move(v));  // 编译通过
            ```
            
        - 位于模板内并不能保证涉及型别推导
            
            ```cpp
            // vector的类声明
            template<class T, class Allocator = allocator<T>>
            class vector {
            public:
              ...
            	void push_back(T&& param);  // param是个右值引用，虽然形参的声明形式为”T&&“，
            															// 但是调用该函数并不涉及型别推导，
            															// 因为T的类型在创建vector时已确定。
            
            	template<class... Args>
            	void emplace_back(Args... args);  // args是个万能引用
              ...
            };
            
            std::vector<Widget> v;   // 此时push_back的形参型别已确定为Widget,
            												 // 但是emplace_back的形参需要推导。
            ```
            
    - 使用const修饰的”T&&”一定是右值引用
        
        ```cpp
        template<typename T>
        void f(const T&& param);   // param是个右值引用
        ```