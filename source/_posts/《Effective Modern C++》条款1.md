title: 《Effective Modern C++》条款1：理解模板型别推导
date: 2019-4-10 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

一个函数模板的声明和调用大致形如：

```cpp
// 声明
template<typename T>
void f(ParamType param);   // ParamType表示形参型别

// 调用
f(expr);  // expr表示调用时传入的实参
```

模板型别推导的结果，不仅依赖于传入实参的型别，还依赖函数形参的型别。

<!-- more -->

具体可分三种情况：

1. 函数形参ParamType是一个指针或普通引用（非万能引用）
    1. ParamType是引用
        1. ParamType是一个普通引用，传入实参的引用性会被忽略，但保留其常量性。
            
            ```cpp
            // 模板函数
            template<typename T>
            void f(T& param);     // param是个普通引用
            
            // 型别推导
            int x = 27;         // x的型别是int
            const int cx = x;   // cx的型别是const int
            const int& rx = x;  // rx的型别是const int的引用
            
            // 推导过程：保留实参的常量性（constness），忽略实参的引用性（reference-ness）
            f(x);     // T的型别是int, param的型别是int&
            f(cx);    // T的型别是const int, param的型别是const int&
            f(rx);    // T的型别是const int, param的型别是const int&
            ```
            
        2. ParamType是一个const引用，传入实参的引用和常量性都会被忽略。
            
            ```cpp
            // 模板函数
            template<typename T>
            void f(const T& param);     // param是个const引用
            
            int x = 27;         // x的型别是int
            const int cx = x;   // cx的型别是const int
            const int& rx = x;  // rx的型别是const int的&
            
            // 推导过程：传入实参的常量性（constness）和引用性（reference-ness）都会被忽略
            f(x);     // T的型别是int, param的型别是const int&
            f(cx);    // T的型别是int, param的型别是const int&
            f(rx);    // T的型别是int, param的型别是const int&
            ```
            
            因为ParamType已经是一个const类型，因此T的类型推导没必要再包含const。
            
        3. 传入实参是一个数组或函数，ParamType会被推导成数组引用
            
            ```cpp
            // 模板函数
            template<typename T>
            void f(T& param);     // param是个普通引用
            
            const char name[] = "iceberg";   // name的型别是const char[7]
            
            f(name);   // T的型别是数组本身，即const char[7]，
            				   // param的型别数组引用，是const char(&)[7]
            
            void someFunc(int, double);  // someFunc是个函数，其型别是void(int, double)
            f(someFunc);                 // T的型别是函数本身，即void(int, double)，
            														 // param的型别是函数引用，即void(&)(int, double)
            ```
            
            扩展：可用于在编译期计算数组大小
            
            ```cpp
            // 以编译器常量形式返回数组的大小
            template<typename T, std::size_t N>
            constexpr std::size_t array_t arraySize(T (&)[N]) noexcept {
            	return N;  // 返回数组大小
            }
            
            const int keys = {1, 3, 7, 9 , 11, 22, 43};
            std::array<int, arraySize(keys)> mapped;  // 编译期就可以确定mapped数组的大小
            ```
            
    2. ParamType是指针
        1. ParamType是一个普通指针，传入实参的常量性会被保留
            
            ```cpp
            // 模板函数
            template<typename T>
            void f(T* param);      // param是指针
            
            int x = 27;           // x的型别是int
            int* px1 = &x;        // px1的型别是int*
            const int* px2 = &x;  // px2的型别是const in*
            
            // 推导过程：传入实参的常量性会被保留
            f(x);      // T的型别是int，param的型别是int*
            f(px1);    // T的型别是int，param的型别是int*
            f(px2);    // T的型别是const int，param的型别是const int*
            ```
            
        2. ParamType是一直const指针，传入实参的常量性会被忽略
            
            ```cpp
            // 模板函数
            template<typename T>
            void f(const T* param);      // param是指针
            
            int x = 27;           // x的型别是int
            int* px1 = &x;        // px1的型别是int*
            const int* px2 = &x;  // px2的型别是const in*
            
            // 推导过程：传入实参的常量性会被忽略
            f(x);      // T的型别是int，param的型别是const int*
            f(px1);    // T的型别是int，param的型别是const int*
            f(px2);    // T的型别是int，param的型别是const int*
            ```
            
2. 函数形参ParamType是万能引用
    
    万能引用既可以绑定左值，也可以绑定右值。因此，当ParamTyep是万能引用时，传入实参即可以是左值，也可以是右值。
    
    1. 传入实参是左值，T和ParamType都会被推导为左值引用
        
        NOTE：这是在所有的模板型别推导中， T被推导为引用型别的唯一情形。
        
        ```cpp
        // 模板声明
        template<typename T>
        void f(T&& param);      // param是万能引用
        
        int x = 27;         // x的型别是int
        const int cx = x;   // cx的型别是const int
        const int& rx = x;  // rx的型别是const int&
        
        f(x);               // x是个左值，T和Param的型别均为int&
        f(cx);              // cx是个左值，T和Param的型别均为const int&
        f(rx);              // rx是个左值，T和Param的型别均为const int&
        ```
        
    2. 传入实参是右值，则使用情况1中的规则。
        
        ```cpp
        // 模板声明
        template<typename T>
        void f(T&& param);      // param是万能引用
        
        int x = 27;          // x的型别是int
        const int cx = x;    // cx的型别是const int
        int& rx = x;         // rx的型别是int&
        const int& crx = x;  // crx的型别是const int&
        
        f(27);               // 27是个右值，T的型别是int，Param的型是int&&
        f(std::move(x));     // T的型别是int，Param的型是int&&
        f(std::move(cx));    // 编译错误！右值引用不能绑定到const型别
        f(std::move(rx));    // T的型别是int，Param的型别int&&
        f(std::move(crx));   // 编译错误！右值引用不能绑定到const型别
        ```
        
3. 函数形参ParamType是既不是指针，也非引用，即值传递
    1. 由于是值传递，因此不管传入实参是左值还是右值，其的引用性和常量性都会被忽略
        
        ```cpp
        // 模板声明
        template<typename T>
        void f(T param);      
        
        int x = 27;         // x的型别是int
        const int cx = x;   // cx的型别是const int
        int& rx = x;  // rx的型别是const int&
        
        f(x);               // x是个左值，T和Param的型别均为int
        f(cx);              // cx是个左值，T和Param的型别均为int
        f(std::move(rx));   // 传入右值，T和Param的型别均为int
        ```
        
    2. 当传入实参是一个数组，数组会退化成首元素指针
        
        ```cpp
        // 模板函数
        template<typename T>
        void f(T param);      
        
        const char name[] = "iceberg";   // name的型别是const char[7]
        
        f(name);    // T和Param的型别均为const char*
        
        void someFunc(int, double);  // someFunc是个函数，其型别是void(int, double)
        f(someFunc)                  // T的型别是函数本身，即void(int, double)，
        														 //  param的型别是函数指针void(*)(int, double)
        ```