title: Effective Objective-C 2.0 学习笔记
date: 2017-10-07 23:06:07
tags: 
- Objective-C
categories: 
- 编程
- Objective-C
keywords: 
- Objective-C
description: 读书笔记
---

这篇文章是在阅读《Effective Objective_C》一书时的学习笔记，这本书对Objective-C语言的特性进行有深入浅出的分析和讲解，让我受益匪浅。

<!-- more -->

### 第1章 熟悉Objective-C

#### 1: 了解Objective-C语言的起源

* Objective-C语言是由Smalltalk演化而来，后者是消息型语言的鼻祖。
* 消息机构的语言，不论是否多态，其运行时所执行的代码都由运行环境来决定；而使用函数调用的语言则由编译器决定，但调用函数是多态的，则是在运行时通过查询“虚方法表（virtual table）”决定具体执行函数。
* 如果只需要保存int、flot、double、char等非对象类型，通常使用CGRect这种结构体，因为结构体可以使用栈空间，而不用分配和释放堆空间，避免额外开销。

#### 2: 在类的头文件中尽量少引用其他头文件

* 除非确实有必要，否则不要引入头文件，尽量使用向前声明，这样不但可以缩短编译时间和降低类之间的耦合。
* 如果无法使用向前声明，比如要声明某个类遵循一项协议。尽量把所遵循的协议移至实现文件中的匿名分类。如果必须在头文件引入协议头文件，则把协议单独放在一个头文件中。

#### 3: 多用字面量语法，少用与之对等的方法

* 使用字面量语法创建字符串、数值、数组、字典更为简单扼要。
* 使用取下标操作访问数组下标或者字典中的键对应的元素。
* 使用字面量语言更为安全，遇到nil对象会抛出异常。
* 字面量语法的限制：除了字符串以外，所创建出来的对象必须属于Foundation框架才行。

#### 4: 多用类型常量，少用#define预处理指令

* 若常量局限于某个实现文件之内，则前面加字面k，若常量在类之外，则通常以类名为前缀。
* static修饰符意味着变量仅在定义此变量的编译单元中可见，在Objective-C的语境下，编译单元通常指每个类的实现文件，即以.m为后缀名。
* 常量必须使用const修饰符声明，常量定义从右至左解读，下面例子中定义了一个常量指针，const修饰的是指针，指向NSString对象。
```objc
extern NSString *const ECOStringConstant;
NSString *const ECOStringConstant = @"VALUE";
```
* 如果一个变量同时声明为static和const,那么编译器根本不会创建符号，而是会像#define预处理指令一样，把所有遇到的变量都替换为常值。
* 在头文件中使用extern来声明全局常量，并在相关实现文件中定义其值。这种常量要出现在全局符号中，因此通常与之相关的类名做前缀加以区分。

#### 5：用枚举表示状态、选项、状态码

* C++11标准修订了枚举的某些特性，其中包括可以指定何种“底层数据类型”来保存枚举类型的变量。这样做的好处是可以向前声明枚举变量了，如果编译器不清楚底层数据类型就不知道分配空间大小。
* 如果枚举类型是多个选项且同时使用，那么久将各选项值定义为2的幂，以便按位或操作进行组合。
* 用NS_ENUM与NS_OPTION宏来定义枚举类型，并指明其底层数据类型，这样确保枚举是用开发者所选的底层数据类型实现出来的，而不会采用编译器所选的类型。
* 在处理枚举类型的switch语句中不要实现default分析，这样加入新的枚举之后，编译器就会提示开发者：switch语句并未处理所有枚举。

### 第2章 对象、消息、运行期

#### 第6条： 理解“属性”这一概念

* 使用@property语法编译器会自动创建一套存取方法，因此访问属性实质上就是调用存取函数，走消息派发流程。
* @synthesize语言可以指定属性实例变量的名字，默认的属性名前加下划线。
* @dynamic关键字可以阻止属性创建实例变量和存取方式，编译器即使在编译过程中没有发现该属性的存取方法也不会保存，而是相信这些方法在运行期能够找到。
* 原子性只能确保每次都能获取属性的有效值，即确保属性修改完成再被其他线程访问，但是不能确保线程安全。
* 处于性能考虑，iOS程序所有属性都是nonatomic，Mac OS X程序使用atomic属性通常不会有性能瓶颈。
* 对应一个属性定义的变量来说，直接访问实例变量会绕开指定的属性特质和消息派发流程。

#### 第7条：在对象内部尽量直接访问实例变量

* 使用“点语法”（本质上使用实例变量存取方法）与直接访问实例变量的区别：
	* 直接访问实例变量不经过Objective-C的消息派发流程，因此速度比点语法快。
	* 直接访问实例变量不会调用属性配置的存取方法，从而绕过了属性指定的相关特质。
	* 直接访问实例变量不会触发“键值观测KVO”通知，因为键值观测是建立在存取方法之上的。
* 在初始化方法和dealloc中尽量直接访问实例变量，因为子类可能覆写父类属性的存取方法，从而无法输出预期。
* 在对象内部读取数据时，应该直接通过实例变量来读，除惰性初始化技术之外。而写入数据时则应通过属性来写。

#### 第8条：理解“对象等同性”这一概念

* 检测对象的等同性，必须提供“isEqual:”和hash方法。相等的对象hash值必须相同，但是has值相同的对象未必相等。
* 计算hash值时应考虑减少碰撞。因为collection检索哈希表时会用对象的哈希值作为索引。hash方法的高效与低碰撞率可以使collection减少开销。
* 等同性判定的执行深度根据具体的对象决定，不一定将整个对象进行判定，有时候只需要判定代表对象唯一性的值即可。
* 在容器中放入可变类对象时，确保对象加入后就不再改变哈希值，否则容易造成未知行为。

#### 第9条：以“类簇模式”隐藏实现细节

* 类簇模式可以把实现细节隐藏在一套简单的公共接口后面。
* 系统框架中经常使用类簇，比如NSArray、NSNumber等。
* 从类簇的公共抽象基类中继承子类时，遵循几条规则：
	* 子类应该继承自类簇中的抽象基类。
	* 子类应该定义自己的数据存储方式。
	* 子类应当覆写超类文档中指明需要覆写的方法。

#### 第10条：在既有类中使用关联对象存放自定义数据

* 使用关联对象可以将两个对象以属性的方式关联起来，并指定类似于@property的内存管理语义和存储策略。
* 设置关联对象的键值是个“不透明指针”，若想令两个键值匹配到同一个值，二者必须是完全相同的指针才行。鉴于此，设置关联对象值时通常使用静态全局变量做键值。
* 由于关联对象容易引入难于排查的bug，所以不要轻易使用。

#### 第11条：理解objc_msgSend的作用

* objc_msgSend通过在运行期搜索接收者类的方法列表实现“动态绑定”，以选择子为key值搜索函数指针。
* 函数原型与objc_msgSend函数很像，在objc_msgSend函数内部搜索到选择子对应的函数并在函数最后return语句调用，利用“尾调用优化”进行优化。

#### 第12条：理解消息转发机制

* 消息转发分两大阶段：动态方法解析和完整的消息转发机制（分两阶段）
* 动态方法解析：实例对象收到无法解析的消息会触发类方法：+（BOOL）resolveInstanceMethod:(SEL)selector，如果是类对象对应的类方法为：+（BOOL）resolveClassMethod:(SEL)selector。在这个方法中可以通过class_addMethod动态插入方法，所添加的方法是用纯C函数（IMP指针）。
* 完整的消息转发机制第一阶段：调用函数-(id)forwardingTargetForSelector:(SEL)selector，看是否能把消息转发其他对象处理。通过此方案可以用“组合”来模拟”多重继承“的某些特性。
* 完整的消息转发机制第二阶段：- (void)forwardInvocation:(NSInvocation *)invocation;此阶段可以在转发消息前修改消息内容。若本类不处理，则需调用父类同名方法。

###第4章 协议与分类

#### 23：通过委托与数据源协议进行对象间通信

* 委托模式为对象提供了一套接口，使其可由此将相关事件告知其他对象
* 当某对象需要从另外一个对象中获得数据时，可以使用委托模式【数据源协议】。
* 若有必要，可实现含有位段的结构体，将委托对象是否能响应相关协议方法这一信息缓存至其中。

#### 24：将类的实现代码分散到便于管理的数个分类之中

* 使用分类机制将类的实现代码划分成易于管理的小块
* 将应该视为“私有”的方法归入名叫private的分类中，以隐藏实现细节。

#### 25：总是为第三方类的分类名称加前缀

* 分类机制通常用于向无源代码的既有类中增加新功能，且分类中的方法会覆盖既有类的同名方法，后一个分类会覆盖前一个分类的同名方法。
* 为了减少同名函数覆盖的概率，以命名空间来区分各个分类的名称与其中所定义的方法，在Objective-C中实现命名空间的方法就是给分类名和函数名添加专用的前缀。

#### 26：勿在分类中声明属性

* 属性是封装数据的方式，尽管可以通过关联对象的方式合成实例变量，但是建议最好全部在主接口中实现，分类的作用在于扩展类的功能，而非封装数据
* 除了“class-continuation分类【匿名分类】”之外可以定义属性，其他分类最好只定义方法

#### 27：使用“class-continuation分类”隐藏实现细节

* 匿名分类可以定义方法和实例变量，原因在于“稳固的ABI”机制（详见第6条）
* 在实现块中添加匿名分类可隐藏实现细节，即私有变量和方法。
* 编译objective-C++时，在匿名分类中定义c++相关的变量，从而避免了因为在头文件中声明C++变量，进而导致凡是引用该类的其他类都必须改为Objective-C++类。
* 利用匿名分类扩展头文件中声明为“只读”的属性为“可读写”状态
* 利用匿名分类隐藏该类遵循的协议。

#### 28：通过协议提供匿名对象

* 使用协议可将具体的对象类型谈化成遵循某种协议的id类型，协议中规定了对象需实现的方法
* 如果对象类型不重要，重要的是对象是否实现了某些方法，此时可用“匿名对象”来实现这一概念，与Python的“鸭子类型”有点相似。

###第5章 内存管理

#### 第29条：理解引用计数

* 悬挂指针：对象在release之后，内存被放回“可用内存池”，但是不一定保证马上被回收，此时指针属于悬挂指针，容易导致crash。
*  为了避免不经意间使用了悬挂指针，在调用完release之后清空指针
* autorelease能延长对象生命周期，使其在跨越方法调用边界后依然跨越存活一段时间，释放操作会在清空最外层自动释放池时执行，即在当前线程进入下一次事件循环时释放。
* 通常采用“弱引用”来避免循环引用发生，从而避免内存泄漏

#### 第30条：以ARC简化引用计数

* Clang的静态分析器（static analyzer）不但可以指明程序中引用计数出现问题的地方，还能根据需要预先加入适当的retain和release操作以避免这些问题。自动引用计（ARC）数的思路也是源于此。
* ARC在执行retain、release和autorelease等操作时，不是通过普通的Objective-C消息派送机制，而是直接调用其对应的C语言版本，这样效率更高。
* ARC通过命名约定将内存管理标准化，方法名以下列词语开头，其返回的对象归调用者所有：
	* alloc
	* new
	* copy
	* mutableCopy
	否则，返回对象会自动释放，即相当于执行autorelease操作.
* 在编译期和运行期，ARC都把能够相互抵消的retain、release、autorelease操作约简。
* 运行期，为了优化代码，在方法返回自动释放的对象时，调用objc_autoreleaseReturnValue,此函数会检视当前函数调用的代码是否需要对返回对象执行retain操作，如果是则设置一个全局标志位。而不执行autorelease操作；与只对应的是在调用代码如果要保留对象，则不执行retain操作，而是调用objc_retainAutoreleasedRetuenValue.此函数检测之前设置的全局标志位，如果已经置位，则不执行retain操作。
* ARC环境优化方式具体实现由编译器决定，比如将全局标志位存储在STL(Thread Local Storage:线程局部存储，以key-value的形式读写)中，STL只适用于调用和被调用方都是ARC模式的情况，使用__builtin_return_address可以在被调用函数中获得调用函数的栈空间，进而可以推算出调用方后续操作是否调用了objc_retainAutoreleasedReturnValue，如果调用则是ARC环境，反之使用没优化的老逻辑。
* 变量的内存管理的边界问题，在设置变量值时，需要先保留新值，释放旧值，最后设置实例变量，确保即便是新值与旧值是同一对象也不能引发错误。在ARC情况下，无需考虑这种“边界情况”
* ARC下清理实例变量是借用Objective-C++的析构函数实现的，不需要重载dealloc，如果存在CoreFoundation等非Objective-C对象时，只需在dealloc函数中执行CFRetain/CFRelease等释放操作，而不需要调用超类的dealloc方法。

#### 第31条：合理使用dealloc方法

* 在dealloc中只释放对其他对象的引用，解除监听和取消订阅的KVO等，不要做其他事情
* 不要在dealloc中释放开销大或系统内稀缺资源，如文件描述符、套接字以及大块内存等，因为这些资源可能被其他对象持有，不宜保留过长时间，而是实现一个专门用于清理的函数，如close等
* 出于优化效率的目的，系统不能保证每一个对象的dealloc都会执行
* 不应在dealloc中调用执行异步任务的方法或只能在正常状态下执行的方法，因为dealloc所在的线程会执行final release。

#### 第32条：编写“异常安全代码”时留意内存管理问题

* MRC环境下，在@try中创建的对象应在@finaly中释放而非@try中，以避免因抛出异常导致内存泄漏
* ARC环境下，出于对运行期的性能考虑默认情况下是不会处理异常捕获过程中出现的内存泄漏情况。
* ARC环境下，可以通过-fobjc-arc-exceptions这个编译标志开启安全处理异常功能，默认情况是关闭的，但是出于Objectve-C++模式下会自动打开。

#### 第33条：以弱引用避免保留环

* 虽然垃圾回收机制可以检测并回收保留环，但是Mac OS X 10.8之后以及iOS平台不支持这个功能
* MRC环境下，使用unsafe_unretained（表明属性不安全且不归实例所拥有）或者weak属性来避免保留环，且效果等同
* ARC环境下，weak属性在修饰对象被回收后自动清空，更为安全，避免访问悬挂指针。

#### 第34条：以“自动释放池块”降低内存峰值

* GCD或主线程都默认自带自动释放池。
* 自动释放池是以栈的形式存在的，对象收到autorelease消息后，会被放入最近的自动释放池的栈顶。
* 合理运用自动释放池，用以降低应用程序的内存峰值，如for循环中。
* ARC环境下的@autoreleasepool比MRC环境下NSAutoreleasepool更为轻便与安全。

#### 第35条：用“僵尸对象”调试内存管理问题

* 向已回收的对象发送消息是不安全与不稳当的，如果内存已经被复用且复用的对象不能响应此消息则会crash，如果复用对象能够响应消息也许输出不能达到预期，如果内存部分存活则可能消息可能依然有效。
* “僵尸对象”是调试内存管理问题最佳方式。
* 僵尸类是从名为_NSZombie_的模板类复制而来，通过创建一个名为_NSZombie_原类名的新类，再将已回收对象的指针指向新类，原对象的类变了，但是内存结构不变，便于调试。示例代码：

```objc
Class cls = object_getClass(self);
const char *clsName = class_getName(cls);
const char *zombieClsName = "_NSZombie_" + clsName;
Class zombieCls = objc_lookUpClass(zombieClsName);
if (!zombieCls) {
	class baseZomeCls = objc_lookUpClass("_NSZombie_");
	zombieCls = objc_duplicateClass(baseZombieCls, zombieClsName, 0);
}
//Perform normal destruction of the object being deallocated
objc_destructInstance(self);
//Set the class of the object being deallocated to the zombie class
objc_setClass(self, zombieCls);
```
代码的关键在于：对象的内存没有释放，因此这块内存不可被其他对象复用，虽然会造成内存泄漏，但是出于调试的目的可以忽略。

* 僵尸类的作用是通过消息转发机制体现，因为僵尸类没有实现任何方法，和Object一样是根类，只有一个实例变量isa。在消息转发机制通过类名检测到当前对象是一个僵尸对象时会进行特殊处理：打印原类的相关信息，然后终止程序。示例代码：

```objc
Class cls = object_getClass(self);
const char *clsName = class_getName(cls);
//If so, this is a zombie
if (string_has_prefix(clsName, "_NSZombie_")) {
	const char *originClsName = substring_from(clsName, 10);
	const char *selectorName = sel_getName(_cmd));
	
	Log("*** - [%s %s]: message sent to deallocated instance %p" , originalClsName, selectorName, self);
}
```
#### 第36条：不要使用retainCount

* retainCount返回的保留计数只是某个给定时间点上的值，并未考虑到对象加入自动释放池的情况，因此不能反映真实的保留计数。
* 有时系统出于优化的目的，retainCount可能永远都不返还0，在保留计数为1的时候就被回收了。

###第6章 块与大中枢派发

####第37条 理解“块”这一概念

* 块在定义与使用方面与函数类似，但是块本身是一个对象，有引用计数。
* 在块中直接访问实例变量，虽没有显性使用self，但是self变量还是会被块捕获。
* 块的内存结构中，最重要的是invoke函数指针，指向块的实现代码，第一个void *参数指代块，用于访问块对象所捕获的变量。此外descriptor变量指向结构体指针，每个块里都包含此结构体，其中声明了块对象的总大小和copy与dispose两个辅助函数的指针。
* 根据内存位置分为全局块、栈块和堆块。
* 栈块只在定义氛围有效，下述代码存在一个比较隐蔽的错误，存在危险：块的内存都分配在if及else范围内，在离开相应范围后如果编译器覆写了分配给块的内存则会导致crash，否则正常运行。

```objc
	void (^block)();
	if (/* some condition */) {
		block = ^{ NSLog(@"I am block A"); };
	}else { 
		block = ^{ NSLog(@"I am block B"); };
	}
	//解决方式：将块拷贝到堆中，成为堆块
	void (^block)();
	if (/* some condition */) {
		block = [^{ NSLog(@"I am block A"); } copy];
	}else { 
		block = [^{ NSLog(@"I am block B"); } copy];
	}
```
6. 在全局范围声明的块及为全局块，其所使用的内存区域也在编译器全部确定。

#### 第38条 为常用的块类型创建typedef

* 以typedef重新定义块类型，可令块变量用起来更简单
* 定义新类型时应遵循现有的命名习惯，勿使其名称与别的类型相冲突。
* 可以为同一个块签名定义多个类型的别名，便于理解类型的用途。

#### 第39条 用handler块降低代码分散程度

* 异步任务执行完成之后，可使用委托协议或者内联块。相比于委托代理，块更为简洁和聚合。
* 建议使用同一个块来处理成功与失败的情况。
* 设计API时如果用的了handler块，可以增加一个参数，使调用者可通过此参数来决定块执行的队列。

#### 第40条 用块引用其所属对象时不要出现保留环

* 如果块所捕获的对象直接或者间接地保留了块本身，那么得注意是否存在保留环的问题。
* 一定要找个适当的时机解除保留环，而不能把责任推给API调用者。
* 网络下载器可在任务启动时将自己加入全局的容器对象中，在任务结束后移除，从而保证自己在任务执行期间存活的同时不需要API调用方引用，大部分网络通信库都是采用这办法，如Twitter框架的TWRequest对象。

#### 第41条 多用派发队列，少用同步锁

* 同步块@synchronized(obj)会根据给定对象自动创建一个锁，锁在代码块执行完成释放。由于给定对象相同，那么意味着它们使用的同一个锁，代码块需按顺序执行。如果在两个或多个没有逻辑关联的代码块给同一个对象加锁，会影响执行效率。
* 滥用@synchronized(self)很危险，因为所有同步块都会彼此抢夺同一个锁。示例：使用@synchronized实现属性的原子性（atomic）：

```objc
	- (NSString *)someString {
		@synchronized(self) {
			return _someString;
		}
	}
	
	- (void)setSomeString:(NSString *)someString {
		@synchronized(self) {
			_someString = someString;
		}
	}
```
如果有很多属性都是类似写法，那么每个属性的同步块都要等其他同步块执行完成才能执行。理想情况应该是属性各自独立地同步。

* 使用GCD实现属性原子性：

```objc
_syncQueue = dispatch_queue_create("com.effectiveobjectivec.syncQueue", NULL);
//同步队列+同步派发
- (NSString *)someString {
	__block NSString *localSomeString;
	dispatch_sync(_syncQueue, ^{
		localSomeString = _someString;
	});
	return localSomeString;
}
//同步队列+同步派发
- (void)setSomeString:(NSString *)someString {
	dispatch_sync(_syncQueue, ^{  
		_someString = someString;
	});
} 
```

* GCD版本优化：设置方法不需要返回值，所以并不一定非得同步；获取方法可以并发执行，且设置方法与获取方法之间不能并执行。

```objc
_syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
//异步队列+同步派发
- (NSString *)someString {
	__block NSString *localSomeString;
	dispatch_sync(_syncQueue, ^{
		localSomeString = _someString;
	});
	return localSomeString;
}
//异步队列+异步派发（同步派发可能效率更高，原因在于异步派发需要拷贝块，如果拷贝时间超过代码执行时间，则得不偿失。异步派发适合较为复杂的任务）+栅栏块
- (void)setSomeString:(NSString *)someString {
	//栅栏块会单独执行，执行前会等待当前所有并发块执行完成，避免出现读写竞赛
	dispatch_barrier_async(_syncQueue, ^{  
		_someString = someString;
	});
} 
```
#### 第42条 多用GCD，少用performSelector系列方法

* performSelector系列方法在内存管理方面容易有疏忽，由于无法确认将要执行的选择子是什么，因而ARC编译器无法插入适当的内存管理方法。
* performSelector系列方法所能处理的选择子有局限性，选择子的参数个数与类型以及返回类型都受到限制。

#### 第43条 掌握GCD及操作队列的使用时机

* 对于只需要执行一次的代码来说，GCD的dispatch_once是首选，但是执行后台任务则可以考虑NSOperationQueue。
* 在iOS4与Mac OSX 10.6开始，操作队列在底层是用GCD来实现的。
* GCD是纯C的API，任务使用轻量级数据结构块来表示；操作队列则是Objective-C的对象，采用更为重量级的NSOperation对象执行任务。
* 使用操作队列的优势：
	* 可取消还未启动的任务
	* 可指定操作间的依赖关系
	* 可通过键值观测机制监控NSOperation对象的属性
	* 可指定操作的优先级，而GCD只能指定队列的优先级
	* 可重用NSOperation对象
* 是否使用底层实现方案还是高层API，可通过实际性能测试来确定。

#### 第44条 通过Dispatch Group机制，根据系统资源状况来执行任务

* 一系列的任务可归入一个dispatch group中，开发者可以在所有任务完成后获得通知
* 利用dispatch group并发地执行多项任务。
* dispatch_apply是持续阻塞的，直到所有任务都执行完成。

#### 第45条 使用dispatch_once来执行只需要运行一次的线程安全代码

* dispatch_once采用“原子访问”来判断块中的代码是否已经执行过，而非使用重量级的同步机制，相比于@sychronized更为高效，如果想了解@synchronized的实现机制可以看看[这篇博客](http://icebergcwp.com/%E5%89%96%E6%9E%90@synchronized%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0.html)。
* 标记dispatch_once_t应该声明为static，确保每次调用都复用同一个变量。

#### 第46条 不要使用dispatch_get_current_queue

* dispatch_get_current_queue已被废弃，只应做调试使用
* 派发队列是按层级来组织的，子队列是包含于父队列的，所以无法单用某个队列对象来描述“当前队列”这一概念
* dispatch_get_current_queue用于解决由不可重入代码所引发的死锁，可以使用“队列特定数据”来解决：dispatch\_queue\_set\_specific(dispatch\_queue\_t queue,const void \*key, void \*context, dispatch\_function\_t destructor);

### 第7章 系统框架

* 标准根类NSObject属于Foundation框架，而非语言本身。如果不适应Foundation框架，则需要自己实现根类

#### 第47条 熟悉系统框架
* 在众多框架中，Foundation和CoreFoundation这两个框架最为重要，提供了许多核心功能
* Objective-C编程经常会使用纯C实现的框架，比如CoreFoundation，里面用到底层C语言级API，这样可以绕过运行时系统，提升速度，但是需要手动管理内存。

#### 第48条 多用块枚举，少用for循环
* 遍历collection有四种方式。最基础的是for循环，其次是NSEnumerator遍历和NSFastEnumeration协议下的快速遍历，最快、最先进的方式是“块枚举发”
* “块枚举法”本身能够通过GCD来并发执行遍历，无须另行代码，其他遍历方式则不能做到这一点
* 如果提前知道遍历collection中的对象，应修改块签名，指出对象的具体对象。

#### 第49条 对自定义其内存管理语义的collection使用无缝桥接

* Foundation框架中的collection类都有与之对应的CoreFoundation框架版的C语言API
* 桥接符号__bridge表示ARC保留对Objective对象的所有权，而__bridge__retain则刚好相反，需要使用CFRelease释放内存。
* 使用CoreFoundation框架可以创建出Foundation框架所不具备的功能，比如NSDictionary的键值内存管理语义是“copy”，即键值必须支持Copying协议，使用CoreFoundaition创建一个键值内存管理语义为“Retain”的CFDictionary。

#### 第50条 构建缓存时选用NSCache而非NSDictionary

* 实现缓存时应选NSCache而非NSDictionary对象。NSCache提供了优雅的自动删减功能，而且线程安全。此外，它与字典不同，不会拷贝键值，而是retain一次。
* 可以给Cache对象设置上限：缓存对象总个数和缓存总大小，这些设置定义了缓存删减其中对象的时机。但是这些设置仅对Cache起指导作用，并非一定在系统资源紧张时删减Cache中的某个对象，因此不能通过设置上限来迫使Cache优先删减某个对象。
* 将NSPurgeableData与NSCache搭配使用，可实现自动清除数据功能，也就是说当NSPurgeableData对象所占内存被系统丢弃时，对象也会从缓存中移除。
* 缓存的设计初衷是为了提高响应速度，只有那些“重新计算起来费劲”的数据才值得放入缓存，比如网络获取或者磁盘读取的数据。     

#### 第51条 精简+initialize与+load的实现代码

* 在加载过程中，如果类实现了+load方法，那么系统会通过函数指针调用+load。调用顺序是父类->子类->分类。因为程序加载时是使用函数指针调用而非消息机制，所以分类中的+load不会覆盖子类中的+load方法。         
* 程序启动时，运行期处于“脆弱状态（fragile state）”，如果+load方法中使用了其他非系统库（系统库的类在这之前已经加载好了）的类，那么这些类的+load方法也在此时被调用。如果子类没有实现+load方法，那么各级超类是否实现此方法都不被系统调用。
* +load方法务必实现的精简一些，因为整个程序在+load方法时都会阻塞。其真正的用途在于调试程序。
* +initialize方法会在程序首次调用该类之前调用，只调用一次，如果某个类未被使用，那么其+initialize方法一直不会被调用。
* +initialize方法被调用时，运行期系统已经处于正常状态，理论上可以在其中调用任何类的任意公开方法，且是线程安全的。
* +initialize方法与其他消息一样，如果子类未实现它而其超类实现了，那么会子类也会调用一次超类的实现方法。 
* 无法再编译器设定的全局变量，可以放在+initilize方法中初始化。  

#### 第52条 别忘了NSTimer会保留其目标对象

* NSTimer对象会保留其目标，知道计时器调用invalidate方法后失效为止。另外，一次性的计数器在触发任务后就会立即失效。
* 反复执行任务的计时器，很容易引入保留环，可通过扩充NSTimer功能，用块来打破保留环。代码如下：

```objc
@interface NSTimer (ECOBlocksSupport)

+ (NSTimer *)eoc_scheduledTimerWithTimeInterval:(NSTimerInterval)interval  block:(void(^)())block repeats:(BOOL)repeats;
+ 
@end

@implementation NSTimer (ECOBlocksSupport)

+ (NSTimer *)eoc_scheduledTimerWithTimeInterval:(NSTimerInterval)interval  block:(void(^)())block repeats:(BOOL)repeats {
	return [self scheduledTimerWithTimerInterval:interval target:self selector:@selector(eoc_blockInvoke:) userInfo:[block copy] repeats:repeats];
}

+ (void)eco_blockInvoke:(NSTimer *)timer {
	void(^block)() = timer.userInfo;
	if (block) {
		block();
	}
}

@end
```
这个方法仍然存在保留环，计时器现在的target是NSTimer类对象，但是因为类对象无需回收，所以不用担心。

* 上述方法本身不能解决问题，但是提供了解决问题的工具。使用分类中的eoc_scheduledTimerWithTimeInterval来创建计时器：

```objc
- (void)startPolling {
	__weak type(self) weakSelf = self;
	_pollTimer = [NSTimer eoc_scheduledTimerWithTimeInterval:1.0 block:^{ 
	EOCClass *strongSelf = weakSelf;
	[strongSelf p_doPoll]; 
	} repeats:YES];
}
```