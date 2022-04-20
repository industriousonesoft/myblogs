title: C++之引用和指针的区别
date: 2022-4-20 22:17:07
mathjax: false
tags:
- C++
categories:
- 编程
- C++
keywords:

---

> 面试时被问到的一个问题，回答得不够全面，算是查漏补缺。
> 

### 简述

我一直都有一个疑问，为什么在C++中有了指针还要引用？在[Stroustrup: C++ Style and Technique FAQ](https://link.zhihu.com/?target=http%3A//www.stroustrup.com/bs_faq2.html%23pointers-and-references)中给出了回复，原文如下：

> C++ inherited pointers from C, so I couldn’t remove them without causing serious compatibility problems. References are useful for several things, but the direct reason I introduced them in C++ was to support operator overloading.
> 

翻译：C++中指针继承自C语言，出于兼容的原因保留了下来。引用的用处很多，但是C++引入它最直接的原因是为了支持运算符重载。

<!-- more -->

举例：

```cpp
// 使用指针
A operator+(const A* lhs, const A* rhs) {
		return *lhs + *rhs;  // ugly
}
// 调用
A a = &b + &c;  // ugly

// 使用引用
A operator+(const A& lhs, const A& rhs) {
		return lhs + rhs;  // better
}
// 调用
A a = b + c;  // better
```

### 比较

以下是C++中指针和引用的一些主要区别：

1. 变量绑定
    - 指针可多次绑定同一类型的不同变量。
        
        ```cpp
        int a = 3;
        int b = 7;
        int* ptr;
        ptr = &a;
        ptr = &b;
        *p = 9;
        assert(a == 3);
        assert(b == 9);
        ```
        
    - 引用必须在初始化时完成绑定，且不可再次绑定其他任何变量。
        
        ```cpp
        int a = 3;
        int b = 7;
        int& ref; // Error! A reference MUST be bound at initialization.
        int& ref = x; // OK.
        ```
        
2. 初始值
    - 指针可初始化为nullptr。
    - 引用理论上则必须绑定某个非空的对象。引用其实也可强制绑定到nullptr，但属于行为未知的操作，如下：
        
    ```cpp
    /* The code above is undefined. Your compiler may optimise it
     * differently: emit warnings or refuse to compile it. 
    */
    // Having a reference to a pointer whose value is nullptr.
    int &ref = *static_cast<int *>(nullptr);
    ```
        
3. 内存分配
    - 指针本质是一个变量，同普通变量一样被分配内存空间，且可通过一元操作符&获取内存地址，可通过sizeof获得内存大小。
    - 引用可理解为一个变量的别名，同临时变量一样其内存地址和空间大小是不可见的。对引用的操作都会直接作用于其所绑定的变量上，比如：使用一元操作符&获取的其绑定变量的内存地址，使用sizeof获得的是其绑定变量的内存大小。
        
    ```cpp
    int a = 1;
    int& ref = a;
    int* ptr = &a;
    int* ptr2 = &ref;
    
    assert(ptr == ptr2); // &a == &ref
    assert(&ptr != &ptr2); // Each pointer variable has its own memory address.
    ```
        
4. 嵌套使用
    - 指针可无限嵌套使用，即一级指针，二级指针等。
    - 引用不可嵌套使用，即只有一级引用。
        
    ```cpp
    int a = 1;
    int b = 2;
    int* ptr = &a;
    int* ptr2 = &b;
    int** pptr = &ptr;
    
    **pptr = 3;
    pptr = &ptr2;  // *pptr is ptr2 now.
    **pptr = 4;
    
    assert(a == 3);
    assert(b == 4);
    ```
        
5. 自我迭代
    - 指针可作为数组的迭代器，其步幅是指针本身的长度，与指针指向的类型无关。比如可通过自增操作符++指向数组中下一个元素，亦或通过+3操作跳转到第4个元素。
    - 引用的内存地址是访问的，因此对引用的任何操作最终都会作用于其绑定的对象上。
6. 使用方式
    - 访问指针所指向的内存空间需使用一元操作符“*”，操作指针所指向的对象需使用一元操作符“→”。
    - 访问引用所绑定的对象可直接访问，操作引用所绑定的对象需使用一元操作符“.”。
7. 数组存储
    - 指针可作为数组元素。
    - 引用则不能作为数组元素。
8. 右值绑定
    - 指针不可指向右值内存空间。
    - 引用可绑定右值，比如临时对象。
    
    ```cpp
    const int& ref = int(10); // Legal in C++.
    int* ptr = &int(10);  // Illegal to take the address of temporaty.
    ```
    

### 参考资料

[What are the differences between a pointer variable and a reference variable in C++?](https://stackoverflow.com/questions/57483/what-are-the-differences-between-a-pointer-variable-and-a-reference-variable-in)

[Stroustrup: C++ Style and Technique FAQ](https://www.stroustrup.com/bs_faq2.html#pointers-and-references)