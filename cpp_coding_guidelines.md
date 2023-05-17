
# 原则
- 遵从标准：使用标准C++17，不使用任何编译器的扩展语法
- C++标准库优先：标准库和编程接口优先于Posix系统调用，优先于C标准库接口，优先于第三方库
- 简单原则：
	- 只使用常用的语法特征、容器、算法、编程接口，非特殊必要不使用高阶特征
	- 谨慎引入第三方库，可以用行业标准解（例如protobuf、gtest），但不用boost库，不为追求性能用更快速的库（至少第一阶段不考虑）
	- 不要让别人猜
- 最小共识集: 团队在最小共识集下工作可以提升协作效率，同时降低门槛，要努力做到对新人友好
- 一致性原则：保持一致性有利于理解和维护代码
- 把可读性放在更优先位置，只在非常必要的情况下，才考虑牺牲可读性换取性能

# 设计
- 避免过度设计，但要适当考虑扩展性
- 遵从common sense，遵从常规
- 优先考虑common case，corner case后置
- 直截了当表达设计

# 模块
- 模块内高内聚、模块间低耦合
- 检查在模块间进行

# 文件
- 头文件以.h结尾，源文件以.cpp结尾，不使用其他后缀，比如.hh .hpp .inl .cxx .cc等
- 每一个源文件尽量对应一个头文件（abc.h vs abc.cpp）,即除文件名除后缀外相同
- 头文件用`#pragma once`预防被重复包含
- 头文件包含要自给自足：即包含所有必须的头文件，不包含任何多余的头文件

# 范围（namespace）
- 只使用一级namespace，不搞嵌套namespace，利用namespace的防冲突作用，同时避免多级namespace的副作用（影响开发和调试）
- 不要图省事在头文件中用using namespace std或其他namespace（会扩散），源文件可以使用using namespace，但不推荐使用
- 只在某源文件使用的辅助函数、源文件域变量，置于匿名namespace下

# 命名
- 好名字的标准：大声读出来，舌头不打结，感到舒服和清晰就好
- 简明扼要、见名知义
- 避免否定之否定

# 类
- 一个类做一个事情，不要巨类
- 控制类的接口数量
- 区分类的成员函数：公开接口 + 私有实现
- 公有继承代表is-a，私有继承代表implements-with
- 终端类建议加final

# 函数
- 一个函数做一个事情，一个函数专注于一个逻辑
	- 要做函数名涵盖的事情
	- 不要做超过名字含义的事情
- 函数要尽量短小，尽量不超过50行
- 把一段逻辑封装成一个函数，然后调用该函数，比就地铺开逻辑好，性能敏感的关键代码除外
- 参数类型定义原则：基本数据类型传值，非基本上类型传引用，输入参数尽量const T&，输入输出参数T&
- 通过返回值报告结果
- 全局函数要尽量无状态
- 成员函数积极使用const、override
- 不要把参数编码进函数名，比如要`get(int id)`，不要`getById(int id)`

# 变量
- 就近定义原则
- 为变量精心挑选一个简短而富有意义的名字，循环控制变量可以用i、j、k，符合常规即可
- 不要任意扩大变量的作用域

# 宏
- 不完全拒用宏，但要非常谨慎和小心
- 宏影响可读性
- 警惕宏的副作用
- 考虑用`do {} while (0)`去定义宏
- 考虑宏的替代：inline函数、const变量

# 异常
- 不用任何异常（因为使用异常的收益低于引入的成本，C++的异常规范一直在变化）

# 注释
- 适量注释，不是越多越好，好的代码是自注释的，只在必要的时候添加注释

# 资源管理
- 源码里不拒绝raw pointer，不矫枉过正
- 了解RAII和工作原理，使用RAII预防资源泄漏
- 偏好scoped对象和stack对象
- 参数传递时，`T*`代表单个对象，`T* + num`代表数组
- 区分`unique_ptr shared_ptr weak_ptr`，恰当的使用它们

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

