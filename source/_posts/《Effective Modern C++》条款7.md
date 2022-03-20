title: 《Effective Modern C++》条款7：在创建对象时注意区分()和 {}
date: 2019-4-18 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

在C++11中有三种指定初始化值得方式，分别是使用小括号()、等号=和大括号{}。其中，前两种在C++98中就已经存在，而使用大括号的初始化语法则是C++11引入的。

<!-- more -->

这三种初始化语法只有大括号适用于所有初始化场景，比如：

1. 初始化非静态成员变量
    
    可使用等号=或大括号{}，不能使用小括号()
    
    ```cpp
    class Widget {
    	...
    private:
    	int x{ 0 };  // 可行
    	int y = 0;   // 可行
    	int z(0);    // 不可行！
    };
    ```
    
2. 初始化不可复制对象
    
    可使用小括号()或大括号{}，不能使用等号=
    
    ```cpp
    std::atomic<int> a_1{0};    // 可行
    std::atomic<int> a_2(0);    // 可行
    std::atomic<int> a_1 = 0;   // 不可行，无赋值函数
    ```
    

大括号初始化语法的三个独有的特性：

1. 禁止隐式窄化类别转换（narrowing conversion）
    
    ```cpp
    double x, y ,z;
    int sum1{x + y + z};  // 报错！double之和可能无法用int表达，存在窄化转换
    int sum2(x + y + z);  // 可行，double之和被截断（窄化）为int
    int sum3 = x + y + z; // 可行，同上
    ```
    
2. 对C++解析语法免疫
    
    > C++规定：任何能够解析为声明的语句都要解析为声明。
    > 
    
    因此，当使用默认构造方式来创建对象时会被解析为一个函数声明，如下：
    
    ```cpp
    Widget w1(10);  // 正确，调用Widget的带形参的构造函数，w1被解析为对象
    Widget w2();    // 错误！本意是想调用Widget的默认构造函数创建w2对象，
    								// 结果w2被解析为函数声明，且返回一个Widget对象
    Widget w3{}     // 正确，由于函数声明不能使用大括号来指定形参列表，
    								// 所以w3不会被解析为函数声明，而是调用默认构造函数创建w3对象
    ```
    
3. 优先选择使用std::initializer_list模板为形参的构造函数，甚至劫持复制或移动构造函数
    
    根据std::initializer_list模板中的型别，具体可分四种情况：
    
    1. 构造函数的参数可被隐式强制转换成std::initializer_list模板中的型别，如下int和bool都可以强制转换成long double。
        
        ```cpp
        class Widget {
        public:
          Widget(); // 默认构造函数
        	Widget(int i, bool b);   // 第一个构造函数
          Widget(int i, double d);   // 第二个构造函数
          Widget(std::initializer_list<long double> i); // 第三个构造函数
        
          operator float() const;  // 强制转换成float
          ...
        };
        
        Widget w1(10, true);  // 调用第一个构造函数
        Widget w2{10, true};  // 调用第三个构造函数，10和true均被强制转换成long double
        
        Widget w3(10, 5.0);   // 调用第二个构造函数
        Widget w4{10, 5.0};   // 调用第三个构造函数，10和5.0均被强制转换成long double
        
        Widget w5(w4);  // 调用复制构造函数
        Widget w6{w4};  // 调用第三个构造函数，w4的返回值被强制转换成float，
                        // 随后又被强制转换成long double
        
        Widget w7(std::move(w4));  // 调用移动构造函数
        Widget w8{std::move(w4)};  // 调用第三个构造函数，同w6
        ```
        
    2. 构造函数的参数可被隐式窄化转换成std::initializer_list模板中的型别
        
        比如上列中的std::initializer_list模板中的型别是bool，即std::initializer_list<bool>。由于int或double转换成bool属于窄化转换，而窄化转换在大括号初始化中是被禁止的，因此会编译报错。
        
    3. 构造函数的参数无法隐式强制转换成std::initializer_list模板中的型别
        
        比如上列中的std::initializer_list模板中的型别是std::string，即std::initializer_list<std::string>。由于int或double无法被隐式转换成std::string，因此编译器会退而求其次去找其他的构造函数。
        
    4. 空大括号
        
        空大括号{}表示的语义是”没有实参“，而非”空的std::initializer_list“，因此使用空大括号调用的是默认构造函数。
        
        ```cpp
        Widget w1;    // 调用默认构造函数
        Widget w2{};  // 同上，调用默认构造函数
        Widget w2();  // 被解析为函数声明
        
        Widget w4({})  // {}作为实参表示一个空的std::initializer_list，
        				   // 因此调用std::initializer_list为形参的构造函数
        Widget w5{{}}  // 同上
        ```