### 一、是什么
模板是 C++ 中实现 **泛型编程** 的核心工具。它允许你编写 **与类型无关的** 的代码(如函数或类)，在编译时根据传入的具体类型自动生成对应的版本。你可以把模板看作一种"代码模具" -- 你定义好 "形状"，编译器在编译期帮你“浇筑”出不同的具体实现。
> **关键认知**：模板与 [[RTTI | RTTI(运行时类型识别)]] 恰好相反。RTTI是在 **运行时** 识别类型，而模板是在 **编译时** 生成类型。因此，模板是 **零开销抽象** 的典范 -- 它没有运行时额外负担(除非你主动使用 `typeid` 等)

---

### 二、为什么需要它
- **代码复用**：不必为 `int`、`double`、`string` 分别写重复的排序或容器代码(如 `std::vector`、`std::sort` 都是模板)
- **类型安全**：比`void`或宏替换更安全，编译器即可检查类型匹配
- **性能卓越**：编译期生成具体代码，内敛友好，没有虚函数调用开销(除非显式使用)
---
### 三、核心语法与分类
#### 3.1 函数模板(Founction Template)
定义与类型无关的函数
``` c++
#include <iostream>

// 定义一个通用的最大值函数
template<typename T>
T max(T a, T b) {
	return (a > b) ? a : b;
}

int main() {
	std::cout << max(3, 5) << std::endl; // T = int, 返回5
	std::cout << max(3.14, 2.71) << std::endl; // T = double, 返回 3.14
	std::cout << max('a', 'b') << std::endl; // T = char, 返回 'z'
	
	return 0;
}
```
> **注意**：`typename` 和 `class` 在声明模板参数时完全等价(除了模板模板参数，很少遇到)
#### 3.2 类模板(Class Template)
定义与类型无关的类(如容器)：
``` c++
template <typenam T>
calss Box {
private:
	T content;
public:
	Box(const T& val) : content(val) {}
	T get() const {
		return content;
	}
	void set(const T& val);
};

// 类模板成员函数在类外定义时，必须重复模板声明
template <typename T>
void Box<T>::set(const T& val) {
	content = val;
}

int main() {
	Box<int> intBox(42); // 实例化为 int 版本
	Box<std::string> strBox("hi"); // 实例化为 string 版本
	std::cout << intBox.get() << std::endl;

	return 0;
}
```
#### 3.3 模板参数类型
模板参数不仅可以是类型，还可以是 **非类型参数** (编译器常量) 或 **模板模板参数**：
``` c++
// 非类型模板参数 (常在 std::array 中)
template <typename T, int Size>
class StaticArray {
	T arr[Size]; // Size 在编译器必须已知
public:
	int size() const {
		return Size;
	}
};

StaticArray<int, 10> arr; // Size = 10

// 默认模板参数
template<typename T = int, int N = 100>
class Buffer { /* ... */ };
```
#### 3.4 变参模板(Variadic Template，C++11)
接受任意数量的模板参数(核心用于实现 `std::tuple`, `std::variant` 等)：
``` c++
// 递归展开 (C++11/14 风格)
template <typename T>
void print(T t) {
	std::cout << t << std::endl; // 最后一个元素
}

template <typename T, typename... Args>
void print(T first, Args... args) {
	std::cout << first << ", ";
	print(args...); // 递归调用
}

// C++17 更优雅：折叠表达式(Fold Expression)
template <typename... Args>
void print_all(Args... args) {
	(std::cout << ... << args) << std::endl; // 二元左折叠
}

int main() {
	print(1, 2.5, "hello"); // 输出：1, 2.5, hello
	print_all(1, 2.5, "worlf"); // 输出：12.5world (无分隔符)

	return 0;
}
```
### 四、模板特化与偏化(Specialization / Partial Specialization)
当模板对于某些特定类型有特殊实现需求时，可以进行特化
#### 4.1 全特化(Full Specialization)
为特定类型提供专属实现：
``` c++
template <typename T>
class Printer {
public:
	static void print(const T& val) {
		std::cout << "Generic: " << val << std::endl;
	}
};

// 对 bool 类型进行全特化
template <>
class Printer<bool> {
public:
	static void print(conts bool& val) {
		std::cout << "Boolean: " << (val ? "true" : "false") << std::endl;
	}
}

Printer<int>::orint(10); // Generic: 10
Printer<bool>::print(true); // Boolean: true
```
#### 4.2 偏特化(Partial Specialization)
仅针对部分模板参数或指针/引用等修饰进行特化 (金类模板支持，函数模板不支持)：
``` c++
template <typename T, typename U>
class Pair {
	// 通用实现
};

// 偏特化：两个类型相同时
template <typename T>
class Pair<T, T> {
	// 特殊实现
};

// 偏特化：第二个类型为指针时
template <typename T, typename U>
class Pair<T, U*> {
	// 特殊实现
};
```
### 五、现代C++的关键增强
#### 5.1 CTAD (类模板参数推导，C++17)
C++17起，构造函数可自动推导模板参数，无需显式指定：
``` c++
std::pair p(1, 2.5); // 推导出 pair<int, double> 以前必须写 std::pair<int, double>)
std::vector v = { 1, 2, 3 }; // 推导出 vector<int>
Box b(42); // 如果定义了合适的构造函数，推导出 Box<int> (需 C++17 推导指引)
```
#### 5.2 概念 (Concepts, C++20)
用 `requires` 或 `concept` 约束模板参数，极大的改善编译错误信息并明确接口定义：
``` c++
#include <iostream>

// 要求 T 必须支持 < 比较运算符
template <typename T>
requires std::totally_ordered<T>
T max(T a, T b) {
	return (a > b) ? a : b;
}

// 更简洁的写法 (C++20)
template <std::totally_ordered T>
T min(T a, T b) {
	return (a < b) ? a : b;
}
```
---
### 六、完整示例：实现简易 [[optional | std::optional]] 风格容器
结合多个模板特性
``` c++
#include <iostream>
#include <string>

template <typename T>
class MyOptional {
private:
    alignas(T) unsigned char storage[sizeof(T)]; // 原始内存
    bool m_hasValue = false;

public:
    MyOptional() = default;

    MyOptional(const T& val) {
        new (storage) T(val);
        m_hasValue = true;
    }

    ~MyOptional() {
        if (m_hasValue) {
            reinterpret_cast<T*>(storage)->~T(); // 调用析构函数
        }
    }

    T& value() {
        if (!m_hasValue) {
            throw std::runtime_error("No value");
        }
        return *reinterpret_cast<T*>(storage);
    }

    bool hasValue() const {
        return m_hasValue;
    }

    // 禁止拷贝
};

int main() {
    MyOptional<int> opt1(42);
    MyOptional<std::string> opt2("hello");

    if (opt1.hasValue()) {
        std::cout << opt1.value() << std::endl; // 42
    }
    if (opt2.hasValue()) {
        std::cout << opt2.value() << std::endl; // hello
    }

    return 0;
}
```

---
### 七、常见陷阱与最佳实践
| 陷阱                      | 说明与解决方案                                                                               |
| ----------------------- | ------------------------------------------------------------------------------------- |
| 编译器错误信息的"天书"            | 模板错误常长达数百行。建议使用C++20概念(Concept) 约束类型，或使用 `static_assert` 提前检查                         |
| 代码膨胀(Code Bloat)        | 每种类型实例化都会生成独立二进制代码。如果类型过多，可提取非模板公共逻辑到基类或函数中                                           |
| 以来名称(Dependent Name) 歧义 | 在模板中，若依赖于模板参数的成员是类型，必须加 `typename` ; 若是模板，必须加 `template` 关键字。如：`typename T::iterator` |
| 声明与定义分离问题               | **模板定义必须放在头文件中** (或源文件显式实例化), 否则链接器找不到实例化代码                                           |
| **函数模板偏特化不存在**          | 函数模板不支持偏特化，但可以用重载或 `if constexpr` 替代(C++17)                                           |
| **整型非类型参数局限**           | 非类型模板参数必须是编译器常量(如字面量、`constexpr` 变量), 不能是运行时变量                                        |

---
### 八、与之前问题的关联
| 对比项    | 模板(Template)     | RTTI           | [[variant \| std::variant]] / [[optional \| std::optional]] |
| ------ | ---------------- | -------------- | ----------------------------------------------------------- |
| **时间** | 编译时多态            | 运行时多态          | 运行时 (但内部常依赖模板)                                              |
| **开销** | 零运行时开销 (代码膨胀换速度) | 有内存和时间开销       | 极小开销 (类型索引 + 内存对齐)                                          |
| **用途** | 容器、算法、泛型库        | 调试、复杂对象识别、动态转型 | 表达 "或" 类型、"可选"类型                                            |
实际上, `std::variant<int, double>` 的实现深度以来 **变参模板** (Variadic Template) 和 `union` + **模板特化**, 可见模板是这些现代特性的"地基"

---
### 总结
模板是 C++ 中最强大且最复杂的特性之一。它让 C++ 成为兼具高性能和高抽象的 "双栖语言"。**掌握模板基础** (**函数**/**类模板**、**特化**、**变参**) **足以应对** 90% **日常开发**，而深入 (如 SFINAE、元编程、Concepts) 则是成为 C++ 专家级程序员的必经之路。在实战中，建议优先使用标准库提供的模板 (如 `vector`、`algorithm`)，自己编写模板时则务必注意 **可读性** 与 **编译速度**。