title: 表达式：x=x&(x-1)的含义
date: 2019-3-15 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

表达式x = x & (x - 1)的含义是：

每执行一次，就会把x二进制格式下最右边的1变为0，换言之就是该表达式有效的执行次数等于x二进制格式下1的个数。

<!-- more -->

以下是常见的几个应用：

1. 统计x在二进制格式下1的个数
    
    ```cpp
    int Func(int x) {
    	int countX = 0;
    	while (x) {
    		x = x & (x - 1);
    		countX++;
    	}
    	return countX;
    }
    ```
    
2. 判断一个整数是否是2的n次幂
    
    ```cpp
    int Func(int x) {
      // 如果x是2的n次幂，那么x的二进制表达式只有一个1，其余都是0
    	if (x & (x - 1) == 0) {
    		return true;
    	} else {
    		return false;
    	}
    }
    ```