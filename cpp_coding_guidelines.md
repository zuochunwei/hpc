# 动机
市面上有很多C++编码规范的文档，比如《Google C++编码规范》、《Cpp Core Guidelines》，但这些文档条款繁多，消化吸收+完整遵循难度较高；另一方面，在设计理念和权衡取舍等重要方面没有涉及或不够，我们需要既全面准确又简单可实施的C++开发向导。

为了帮助以Java经验为主的团队快速上手C++开发，让项目成员以一致的方式做设计和实现，让大家更高效的沟通协作，架构师编写了该文档。文档里包含需共同遵从的原则、规则、编码风格，我们既参考了市面上流行的C++编码文档，又补充了一些条目，任何增补内容都秉持一个原则：“摒弃个人好恶，只吸取在业界有广泛共识的最重要的那部分内容。”

C++是易学难精的专家型语言，它拥有丰富的语法特征和库，这既带来了强大的功能，也带来了复杂性，让团队内每个人都熟练掌握每一个语法特征和接口既不现实亦无必要，我们更注重的是如何用好C++做项目。

那么，如何用好C++这个强大的语言工具呢？

我的建议是“**通过人为限制去控制语言复杂度，把更多精力放到设计和领域**”，而这在各种流行的编码规范献少提及。只使用常用的语法特征和精心挑选的编程接口，只使用标准C++非常有限的子集，这样就能降低复杂度，同时获得C++零成本抽象和语言结构直接映射到硬件的双核心能力。

这样的做法是可行的，因为C++的很多高级特性是写库的需求倒逼产生的，如果不去写库，那么有些高级特性自然用不着；这样的做法也可能是有效的，特别是对于团队成员大多数C++经验不足的情况。

# 原则
- 遵从标准：使用ISO C++ Standard 17，不使用任何编译器的扩展语法
- 标准库优先：C++标准库和编程接口优先于Posix系统调用，优先于C标准库接口，优先于第三方库
- 简单原则：
	- 只使用常用的语法特征、容器、算法、编程接口，不做语言律师，非必要不使用高阶特征
	- 只要可能，就不引入额外的复杂性
	- 谨慎引入第三方库，可以用行业标准解（例如protobuf、gtest），但不用boost库，不为追求性能用标准库替代库（至少第一阶段不考虑）
- 实用原则：
	- 尽量用最直接的方式去解决问题
	- 让意图显而易见，不拐弯抹角，不让别人猜
- 一致性原则：有利于理解和维护代码
- 最小共识集: 团队在最小共识集下工作可以提升协作效率，同时降低门槛，代码要努力做到对新人友好
- 先让它跑，再让它跑的更好更快
- 把可读性放在更优先位置，只在非常必要的情况下，才考虑牺牲可读性换取性能

# 架构
- 架构的两个关注：
	- 系统间关注分层、协议和通信风格
	- 系统内关注模块、接口和规范约束
- 什么是好的架构？
	- 好的架构应该是通过把团队每个人“约束”在一个很小的共识集，从而让一群水平不咋样的人能尽快上手并高效协作
	- 好的架构在应对变化的时候应该展现出足够的韧性和适应性

# 设计
- 拆解
	- 通过拆解把复杂的大问题拆分成一个个简单的小问题，拆分问题的过程即是简化问题的过程；问题分解之后，还需要协作，治是分的逆操作
	- 先宏观再微观，先框架再细节，先把核心的框架写出来，不要一下子就陷入细节泥潭
	- 不追求一步到位，认知是螺旋上升的，好框架好代码都是改出来的
- 避免过度设计，但要适当考虑扩展性
- 遵从common sense，遵从常规
- 优先common case，corner case后置
- 直截了当表达设计

# 模块
- 模块内高内聚、模块间低耦合
- 检查在模块间进行，模块内尽量不做重复检查

# 文件和目录
- 一个文件包含一个主要类，文件名跟主要类名保持一致
- 文件名用下划线拼接的单词组成，例如：`file_name_abc.h`
- 头文件以`.h`结尾，源文件以`.cpp`结尾，不使用其他后缀，比如`.hh .hpp .inl .cxx .cc`等
- 每一个源文件尽量对应一个头文件（`abc.h vs abc.cpp`），即文件名除后缀外相同
- 头文件用`#pragma once`预防被重复包含
- 头文件包含要自给自足：即包含所有必须的头文件，不包含任何多余的头文件
- 文件路径不要过深，不要搞像Java那样很深的目录

# 范围（namespace）
- 只使用一级`namespace`，不搞嵌套`namespace`，利用`namespace`的防冲突作用，同时避免多级`namespace`的副作用（影响开发和调试）
- 不要图省事在头文件中用`using namespace std`或其他`namespace`（会扩散），源文件可以使用`using namespace`，但不推荐使用
- 只在某源文件使用的辅助函数、源文件域变量，置于匿名`namespace`下

# 命名
- 好名字的标准：大声读出来，舌头不打结，感到舒服和清晰就好
- 简明扼要、见名知义
- 避免否定之否定：例如`IsNotInvalid`

# 类
- 一个类做一个事情，单一职责，不要巨类、大类
- 基类名不需要加Base后缀：不要ListBase，要List
- 类名：单词首字母大写，类名尽量用名词，比如：`BloomFilter`
- 区分公开接口和私有实现
- 控制类的接口数量
- 当试图定义拷贝构造和赋值操作符，牢记`Rule of three/five/zero`

# 继承
- 尽量避免使用多继承
- 抽象基类的析构函数要定义成虚函数
- 非终端类不要设计成可实例化类，换言之，不要从可实例化的类派生子类
- 子类`override`的成员函数，要加`override`，从而让编译器帮助检查
- 终端类建议加`final`
- 确保公有继承模塑的是`is-a`关系
- 确保私有继承模塑的是`implements-with`关系

# 函数
- 一个函数做一个事情，一个函数专注于一个逻辑
	- 要做函数名涵盖的事情
	- 不要做超过名字含义的事情
- 函数要尽量短小，尽量不超过50行，短小函数实现建议直接写在类定义里
- 函数要保持简单，把一段逻辑封装成一个函数，然后调用该函数，比就地铺开逻辑好，性能敏感的关键代码除外
- 给函数起一个好名字，不要把参数编码进函数名，比如要`get(int id)`，不要`getById(int id)`
- 参数：
	- 个数尽量不超过6个
	- 类型原则：基本数据类型传值，非基本上类型传引用，输入参数尽量`const T&`，输入输出参数`T&`
	- 默认参数要合情合理
- 通过返回值报告结果
- 全局函数
	- 尽量无状态
	- 优先定义为模块内函数
- 成员函数
	- 只读函数要加`const`
	- 不要把成员函数无故设计成虚函数

# 变量
- 就近定义原则，不要集中定义在函数的开头部分
- 为变量精心挑选一个简短而富有意义的名字，循环控制变量可以用`i、j、k`，符合常规即可
- 不要任意扩大变量的作用域

# 宏
- 不完全拒用宏，但要非常谨慎和小心
- 宏影响可读性
- 警惕宏的副作用
- 考虑用`do {} while (0)`去定义宏
- 考虑宏的替代：`inline`函数、`const`变量

# 异常
- 不用任何异常（因为使用异常的收益低于引入的成本，C++的异常规范一直在变化）

# 注释
- 适量注释，不是越多越好，好的代码是自注释的，只在必要的时候添加注释

# 资源管理
- 资源包括：内存、文件句柄、套接字、锁等
- 了解`RAII`和工作原理，使用`RAII`预防资源泄漏
- 基本原则：
	- 使用资源句柄或者RAII自动管理资源
	- 接口中，使用原始指针表示单独的对象
	- 一个原始指针（a T*）意味着非拥有（non-owning）语义
	- 一个原始引用（a T&）意味着非拥有（non-owning）语义
	- 偏好范围（scoped）对象，非必要不堆分配

# 指针
- 源码里不拒绝原始指针，不矫枉过正
- 分配和回收
	- 尽量避免`malloc/free`
	- 尽量避免显式调用`new/delete`
	- 立即把一个资源分配的结果给到资源管理对象
	- 单个表达式语句最多只做一次显式资源分配
	- 分配回收的重载要配对
- 参数传递时，`T*`代表单个对象，`T* + num`代表数组

# 智能指针
- unique_ptr代表独占语义
- shared_ptr代表共享语义
- 使用make_unique去创建unique_ptr对象
- 使用make_shared去创建shared_ptr对象
- 使用weak_ptr去打破shared_ptr的循环引用
- 只有为了去表达清晰的生命周期语义，才应该将智能指针作为参数传递
- 用unique_ptr<widget>做参数，去表达函数取得widget的所有权
- 用unique_ptr<widget>&做参数，表示reseat widget
- 用shared_ptr<widget>做参数，表达共享所有权
- 用const shared_ptr<widget>&做参数，表示它可能保留对象的引用计数
- 不要传递从智能指针别名里获取的对象的指针或引用


# 内存管理
- 优化内存管理要放到功能需求后

# 多线程和多线程同步
- 线程

- 锁
- 条件变量

# 代码格式和风格
主要遵从：Google C++ Style Guide

```c++
class MyClass {
 public:
  int CountFooErrors(const std::vector<Foo>& foos) {
    int n = 0;  // Clear meaning given limited scope and context
    for (const auto& foo : foos) {
      ...
      ++n;
    }
    return n;
  }

  void DoSomethingImportant() {
    std::string fqdn = ...;  // Well-known abbreviation for Fully Qualified Domain Name
  }

 private:
  const int kMaxAllowedConnections = ...;  // Clear meaning within context

	std::string table_name_;  // OK - underscore at end.
  static Pool<TableInfo>* pool_;  // OK.
};

// if / else if
if (typeid(*data) == typeid(D1)) {
  ...
} else if (typeid(*data) == typeid(D2)) {
  ...
} else if (typeid(*data) == typeid(D3)) {
	...
}

// Struct Data Members
struct UrlTableProperties {
  std::string name;
  int num_entries;
  static Pool<UrlTableProperties>* pool;
};

// Constant Names
const int kDaysInAWeek = 7;
const int kAndroid8_0_0 = 24;  // Android 8.0.0

// Enumerator Names
enum class UrlTableError {
  kOk = 0,
  kOutOfMemory,
  kMalformedInput,
};

// Macro Names
#define MYPROJECT_ROUND(x) ...

```

# 参考
- [CppCoreGuidelines](https://github.com/fluz/CppCoreGuidelines)
- [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
- [Thriving in a Crowded and Changing World:C++ 2006–2020](https://dl.acm.org/doi/pdf/10.1145/3386320)

