title: memset函数的使用陷阱
date: 2019-3-10 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:
- memset

---

memset是C/C++编程中常用的函数，一般用于申请内存后的初始化操作，比如：

```cpp
char* buffer = (char*)malloc(1024);
memset(buffer, 0x0, 1024);
```

memset接收三个参数：

- 第一个参数是开始填充的地址
- 第二个参数是填充的byte
- 第三个参数要填充的字节数（注意是字节数）

这个函数看似简单，但是如果使用不当则会导致一些未知的bug。以下是目前遇到的几个陷阱。

<!-- more -->

- 陷阱一：
    
    memset是按字节赋值，虽然第二个参数是int类型，但有效值只取低位的一个字节。这也是为什么第三个参数强调的是填充的字节数而非填充长度，如下。
    
    ```cpp
    int a[2]; 
    memset(a, 0x1203, 2);
    // expect: a = {0x00000003, 0x00000003}
    // actual: a = {0x03030303, 0x03030303}
    ```
    
    使用建议：
    
    1. 最好只用于单字节数组的初始化，e.g., char[]，uint8_t[]。这样不需要担心第二个或第三个参数的取值。
    2. 用于非单字节的数组的初始化时，第二个参数只能是0（b00000000）或-1（b11111111）。
- 陷阱二：
    
    memset函数的第三个参数慎用sizeof的结果。
    
    因为静态数组作为参数传入某个函数的时候，就会退化成指针，也就是该数组的首地址，其数组的长度信息就丢失。如下：
    
    ```cpp
    int a[] = {1,2,3,4,5};
    // sizeof(a) = 20
    // 静态数组a退化成指针，因此sizeof的结果为4.
    memset(s, 0, sizeof(a)/*the result = 4*/);
    // expect: a = {0, 0, 0, 0, 0}
    // actual: a = {0, 2, 3, 4, 5}
    ```
    
 参考资料
 	1. [性能杀手："潜伏"的memset_华仔-技术博客-CSDN博客_memset性能](https://blog.csdn.net/yunhua_lee/article/details/6381866)