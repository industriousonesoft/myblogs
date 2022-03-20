title: 《Effective Modern C++》条款18：使用std::unique_ptr管理专属的资源
date: 2019-5-7 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

std::unique_ptr是C++11用表达专属所有权的方式，一个非空的std::unique_ptr总是拥有其所指的资源。因此，std::unique_ptr不允许复制，即只能通过移动操作将所有权转移。

<!-- more -->

### 三个优点

使用std::unique_ptr相对于其他智能指针或裸指针而已，具有一下三个优点：

- 防止内存泄漏
    
    std::unique_ptr内封装了一个原始指针，通过构造函数和析构函数实现对原始指针的内存管理。即在构造函数中通过new申请原始指针的内存，在析构函数中调用delete释放原始指针。
    
    因此，使用std::unique_ptr时不再需要担心内存泄漏。
    
- 快速小巧
    
    在默认情况下，即使用默认析构器，std::unique_ptr和裸指针占用的内存空间基本相同。
    
    由于std::unique_ptr重载了“→”和“*”运算符，所以可以像使用裸指针一样使用它。
    
    因此，std::unique_ptr几乎和裸指针一样，足够小且足够快。这就意味着在任何使用裸指针的场合都可以使用裸指针替代，包括内存和时钟周期紧张的场合。
    
- 可高效地转成std::shared_ptr
    
    std::unique_ptr可以方便且高速地转换成std::shared_ptr。
    
    ```cpp
    std::unique_ptr<Widget> CreateAWidget();
    
    // 转换成std::shared_ptr
    std::shared_ptr<Widget> sp = CreateAWidget(); 
    ```
    

### 自定义析构器

自定义析构器相较于使用默认析构器的好处在于：可以在原始指针释放之前做一些操作，比如添加日志等。

在使用默认析构器的前提下，std::unique_ptr和裸指针占用的内存大小基本相同。但是，如果std::unique_ptr使用自定义析构器，情况则有所不同。

自定义析构器的实现主要分以下三种类型：

- 无捕获的lambda
    
    由于lambda在没有捕获变量的情况下可以直接转换成函数指针。因此，可使用无捕获的lambda来实现std::unique_ptr的析构器，其所占用内存大小与使用默认析构器一样，即不会增加内存。
    
    ```cpp
    auto deletor = [](Widget* pWidget) {
    		// Make some logs.
    		delete pWidget;
    };
    
    std::unique_ptr<Widget, decltype(deletor)> sp(new Widget(), deletor);
    ```
    
- 函数指针
    
    std::unique_ptr使用函数指针来实现析构器时，其占用内存一般至少增加一个函数指针的大小。
    
    ```cpp
    void deletor(Widget* pWidget) {
    		// Make some logs.
    		delete pWidget;
    }
    
    std::unique_ptr<Widget, void(*)(Widget*)> sp(new Widget(), deletor);
    ```
    
- 函数对象
    
    如果一个类或结构体重载了“()”运算符，则称之为函数类。这个类的对象即为函数对象，该对象可以被当做普通函数进行调用。
    
    函数对象的优势：
    
    - 编译器可以内联执行函数对象的调用。
    - 函数对象可以有自己的状态，在函数对象被多次调用时可共享状态。普通函数则只能使用全局变量来共享状态。
    - 函数对象有自己特有的类型，而普通函数则无清晰的类型界限。因此，在使用模板函数时，可以通过传递类型函数来实例化相应的模板。
    
    使用无状态的函数对象来说实现析构器，与使用无状态的lambda一样不会占用额外内存。因此，如果析构器是函数对象，其带来的内存变化与该函数对象中存储状态的数量成正比。
    
    ```cpp
    class Deletor {
    public:
    		void operator()(Widget* pWidget) {
    				// Make some logs.
    				delete pWidget;
    		}
    };
    
    std::unique_ptr<Widget, Deletor> sp(new Widget());
    ```