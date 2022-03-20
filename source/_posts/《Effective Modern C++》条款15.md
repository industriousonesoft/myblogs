title: 《Effective Modern C++》条款15：只要可能使用constexpr，就使用它
date: 2019-4-27 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

constexpr是C++11引入的关键字，主要目的是为了解决const关键字的二义性，constexpr既可用于修饰对象，又可用于修饰函数。

<!-- more -->

- constexpr对象
    
    所谓const二义性是指const既用于表示变量的只读性，又常用于修饰常量。但是const并不能代表“常量”，它只是变量的一个修饰，告诉编译器这个变量只能被初始化但不能被直接修改（实际上可以通过堆栈溢出等方式修改）。因此这个变量的值，可以在运行时也可以在编译期指定。比如：
    
    ```cpp
    // 形参array_size是一个只读变量，其值在运行期指定
    void func(const int array_size) {
    	std::array<int, array_size> arr;  // 编译错误！因为array_size的在编译期未知，即非编译器常量									
    }
    
    void func2() {
    	// 实参array_size是一个只读变量，其值在编译期指定，因此是一个常量
    	const int array_size = 5;
    	std::array<int, array_size> arr;  // 编译通过，因为array_size的值编译期已知，即编译期常量。
    }
    ```
    
    因此，const并不能保证对象是编译期已知，而constexpr可以保证其修饰的对象一定是编译期常量。如下所示：
    
    ```cpp
    int sz;  // 普通变量
    
    const auto array_size = sz;       // 编译通过，array_size是sz的一个const副本
    constexpr auto array_size2 = sz;  // 编译错误！sz的值在编译期未知
    ```
    
    简而言之，所有的constexpr对象都是const对象，而并非所有的const对象都是constexpr对象。
    
    因此，C++11 标准中，建议将 const 和 constexpr 的功能区分开，即凡是表达只读语义的场景都使用 const，凡是要求编译期常量的语境中都使用 constexpr。
    

- constexpr函数
    
    使用constexpr修饰函数的好处在于可以拓展函数的使用语境，即既可在编译期调用，又能在运行时调用。
    
    - 编译期
        
        如果constexpr函数调用的语境是在编译期，那么必须确保传入的实参都是编译期常量，且内部调用的函数只能是constexpr函数，否则编译失败。
        
        好处是程序运行更快，坏处是编译时间更长。
        
    - 运行期
        
        如果constexpr函数调用的语境在运行时，那么其运行方式与普通函数无异。
        
    
    由于constexpr函数必须在传入编译期常量时能返回编译期结果，它们的实现就必须加以限制。且C++11和C++14中的限制还有所差异：
    
    - C++11的限制条件
        - 保证constexpr函数返回值和参数必须是字面值（即可在编译期决议的值），
        - 只能有且只有一行return代码，通常使用三目运算符（替代if-else语句）或递归（替代循环语句）来扩展表达力。
    - C++14的限制条件
        
        只要保证返回值和参数是字面值，函数体中可有多条语句，这样更为方便灵活。
        
    
    注：在C++11中除void以外的所有内建型别都是字面值，而C++14中所有内建型别都是字面值，包括void。
    
参考资料：

[C++11/14 constexpr 用法](https://www.jianshu.com/p/34a2a79ea947)

[C++11 constexpr和const的区别详解](http://c.biancheng.net/view/7807.html)