title: 《Effective Modern C++》条款8：表示空指针优先选用nullptr，而非0或NULL
date: 2019-4-24 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

- 字面常量0的型别是int，而非指针。C++会在只能使用指针的语境中才勉强将其解释为空指针。
- NULL是一个宏定义，而非指针。同样，C++会在只能使用指针的语境中才勉强将其解释为空指针。
    
    ```cpp
    /* Define NULL pointer value */
    #ifndef NULL
        #ifdef __cplusplus
            #define NULL    0
        #else  /* __cplusplus */
            #define NULL    ((void *)0)
        #endif  /* __cplusplus */
    #endif  /* NULL */
    ```
    
- nullptr是C++关键字，不具备整型型别。实际上，它也不具备指针型别，但是它可以隐式转换成任意指针型别，这也就是为什么nullptr可以用于初始化所有指针型别的原因。

<!-- more -->

### 使用nullptr可避免多义性

由于字面量0和NULL本质上是一个整型，因此用它们表示空指针时会出现多义性，而使用nullptr可避免多义性。

- 避免重载决议中的意外
    
    ```cpp
    class Widget {
    public:
    	....
    
    	void Func(int);  // Func的三个重载版本
    	void Func(bool);
      void Func(void*);
    };
    
    Widget w;
    w.Func(0);        // 调用的是Func(int)，而非F(void*)
    w.Func(NULL);     // 可能编译错误，但一般会调用的Func(int)，
                      // 而非F(void*)
    w.Func(nullptr);  // 调用的是Func(void*)
    ```
    
- 提升代码清晰度
    
    比如涉及auto型别推导时：
    
    ```cpp
    auto ret = FindRecord(/* 实参 */);
    if (result == 0) {  // 很难判断result的型别是整型还是指针
    ...
    }
    
    auto ret = FindRecord(/* 实参 */);
    if (result == nullptr) { // 一目了然
    ...
    }
    ```
    

### 总结

- 相对于0NULL，优先选用nullptr
- 使用nullptr可避免在整型和指针型别之间出现重载意外