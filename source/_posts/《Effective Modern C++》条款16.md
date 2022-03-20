title: 《Effective Modern C++》条款16：保证const成员函数的线程安全性
date: 2019-4-30 7:00:07
tags:
- C++
categories:
- 编程
- C++
keywords:

---

### 简述

使用const修饰对象的成员函数的本意在于告诉调用者调用该函数不会修改对象的成员变量，即在其中只能对成员变量进行只读操作。

<!-- more -->

### 确保线程安全

由于在多线程环境下执行读操作是安全的，因此const成员函数被认为是线程安全的。

换言之，从调用者的角度来说调用一个const成员函数应该是线程安全的，但这只是一种字面上的约定，因为使用const修饰的成员函数并不能保证一定是线程安全。

比如可以在const成员函数中可对使用mutable修饰的成员变量进行写操作。如下：

```cpp
class Widget {
public:
	int CachedValue() const {
			if (cache_valid_) {
				return counter_;
			} else {
        first_cache_ = true;  // 编译错误！first_cache_被视为const对象
				cache_valid_ = true;  // 编译通过，mutable修饰的成员变量可被修改
				cache_value_ = DoExpensiveComputaion();  // 同上
			}
	}

private:
  bool first_cache_{false};
	mutable bool cache_valid_{false};
	mutable int cache_value_{0};
};

Widget w;
/*----线程1-----*/
int cached_value = w.CachedValue();  // 可能出现data race，存在未定义行为
/*----线程2-----*/
int cached_value = w.CachedValue();  // 同上
```

解决方案有两个，分别是：

- 使用互斥量mutex
    
    如果需要同时修改多个成员变量，使用mutex更方便。
    
    ```cpp
    class Widget {
    public:
    	int CachedValue() const {
    			std::lock_guard<std::mutex> lock(mutex_);  // 加上互斥量
    			if (cache_valid_) {
    				return counter_;
    			} else {
    				cache_valid_ = true;  // 编译通过，mutable修饰的成员变量可被修改
    				cache_value_ = DoExpensiveComputaion();  // 同上
    			}
    	} // 解除互斥量
    
    private:
      mutable std::mutex mutex_;
    	mutable bool cache_valid_{false};
    	mutable int cache_value_{0};
    };
    ```
    
    NOTE：由于std::mutex是个只移型别（move-only type），将mutex加入Widget的副作用就是使Widget失去了可复制性，但仍可移动。
    
- 使用原子数据型别std::atomic
    
    如果修改的成员变量只有一个，使用std::atomic的开销更低（是否真的成本更低取决于硬件以及互斥量的实现）。
    
    ```cpp
    class Widget {
    public:
    	int Counter() const {
    			++counter_;  // 带原子性的自增操作
    	} // 解除互斥量
    
    private:
    	mutable std::atomic<int> counter_{0};
    };
    ```
    
    NOTE：与std::mutex一样，std::atomic也是只移型别。因此Widget也变成了只移动型别。