title: 《Effective Modern C++》条款17：理解特种成员函数的生成机制
date: 2019-5-5 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

在C++的官方用语中，所谓特种成员函数是指那些C++编译器会自行生成的成员函数。这些函数仅在需要时才会生成，即某些代码使用了它们，但在类中并未显式声明。

<!-- more -->

C++11中有以下六种特种函数，其中前四种同属于C++98，后两种则是C++11新加入的成员：

- 默认构造函数
    
    默认构造函数仅当类中没有声明任何构造函数时才会被生成。
    
- 析构函数
    
    除了析构函数其他的特种函数都一定是非虚的。当基类的析构函数是一个虚函数时，编译器为派生类生成的析构函数也是一个虚函数。
    
    C++11中析构函数默认为noexcept。
    
- 复制构造函数和复制赋值运算符
    
    两种复制操作是彼此独立的：即只显示地声明其中一个不会阻止编译器生成另外一个。
    
    NOTE：尽管在已存在其中一个复制操作或析构函数的前提下，编译器仍生成另外一个复制操作，但是这成为被废弃行为。因此建议遵守大三律原则。
    
- 移动构造函数和移动赋值运算符
    
    两种移动操作并非彼此独立：即显式地声明了其中一种会阻止编译器生成另外一个。理由在于：一旦显示地声明了其中一种移动操作，编译器会理解为你需要的移动操作实现与编译器默认生成的实现会存在差异，因此希望你自己实现另外一个以统一移动逻辑。
    
    NOTE：调用移动构造或移动赋值并不能保证移动操作真的会发生，更像是一种移动请求。对于不可移动的型别，比如C++98中遗留的型别，将通过其复制操作实现”移动“。
    

### 大三律指导原则

大三律是指：如果声明了复制构造函数、复制赋值运算符或析构函数中的任一个，就得同时声明两位两个。

理由：

如果有改写复制操作的需求或声明了析构函数，往往意味着该类需要执行某种资源管理。即：

1. 如果在一种复制操作中进行了任何资源管理，那么在另外一种复制操作也极有可能需要进行
2. 该类的析构函数也极有可能会参与到资源管理中，比如内存释放操作等。

推论：

如果用户声明了析构函数，那么复制操作就不应该被自动生成，因此他们的行为极可能出错。但可惜地是C++98未支持该推论，即显示声明析构不会阻止编译器生成复制操作。C++11出于兼容C++98的原因也未加以限制，否则会破坏太多的遗留代码。

但是由于移动操作是C++11新引入的函数，因此，基于该推论，C++11规定：只要显示地声明了析构函数，编译器便不再生成移动操作。

### 总结

- 复制操作和移动操作相互压抑，即显示地声明了任意一种复制操作都阻止编译器生成移动操作，反之亦然。
- 复制操作彼此相互独立，即声明其中一个复制操作不会阻止编译器生成另外一个。
- 移动操作相互抑制，即声明其中一个复制操作会阻止编译器生成另外一个。
- 编译器生成移动操作需要同时满足三个条件：
    - 该类未显示声明任何复制操作
    - 该类未显示声明任何移动操作
    - 该类未显示声明任何析构函数
- C++11中可使用”=default“要求编译器自动生成某个特种函数：
    
    ```cpp
    class Widget {
    public:
    	~Widget();  // 用户定义的析构函数
    
    	Widget(Widget&&) = default;  // 使用编译器生成的移动函数
    	Widget& operator=(Widget&&) = default;  // 使用编译器生成的移动赋值运算符
    }
    ```
    
- 成员函数模板在任何情况下都不会阻止特种成员函数的生成。