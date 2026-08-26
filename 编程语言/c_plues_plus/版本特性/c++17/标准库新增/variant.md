### 一、是什么
`std::variant<T1, T2, ..., Tn>` 是C++17引入的 **类型安全联合体** (union)。它可以在任意时刻存储 **一组类型列表中的一个值** (就像 `union` 那样)，但它会记住当前存储的是哪种类型，并且不允许访问错误类型，从而避免了传统 `union` 的未定义行为。
### 二、为什么需要它
- **类型安全**：`std::variant` 保证你只能访问当前有效类型的值，访问错误类型会抛出异常或返回空指针(取决于方法)
- **代替危险的传统**(union)：传统`union`无法存储非平凡类型(如 `std::string`)或自定义类(除非手动管理析构/构造)，而`std::variant` 可以存储任意类型
- **代替混乱的RTTI** + **指针**：在某些场景下(如函数返回多种类型)，以前可能用基类型指针 + `dynamic_cast` 或 `void*` + 枚举类型，而现在可以用 `variant` 优雅的表达"一种类型或另一种"
- 与现代C++**无缝协作**：与`std::visit`、结构化绑定等配合，实现模式匹配风格
### 三、基本用法
#### 3.1 包含头文件
``` c++
#include <variant>
```
#### 3.2 声明与初始化
``` c++
std::variant<int, double, std::string> v; // 默认初始化为第一个类型(int)的初值化(0)
std::variant<int, double, std::string> v2 = 3.14; // 存储 double
std::variant<int, double, std::string> v3 = "hello"; // 存储 std::string(const char* 可转换)
std::variant<int, double> v4{ std::in_place_typr<double>, 2.178 }; // 显式指定类型构造
```
> **注意**：如果你想让 `variant` 默认处于 "无值" 状态(类似于 `optional` 的空)，可以使用 `std::monostate`

#### 3.3 访问值
有三种主要访问方式
(1) `std::get`(**按类型或索引**，**抛出异常**)
``` c++
try {
	int val = std::get<int>(v); // 如果 v 当前不是 int，抛出 std::bad_variant_access
	std::cout << val;
} catch (const std::bad_variant_access& e) {
	std::cout << "类型不匹配" << std::endl;
}

// 按索引(0 开始)
double d = std::get<1>(v); 若当前不是第2个类型，同样抛出异常
```
(2) `std::get_if` (**返回指针**，**不抛异常**)
``` c++
if (int* p = std::get_if<int>(&v)) {
	std::cout << "当前是 int: " << *p << std::endl;
} else if (std::string* p = std::get_if<std::string>(&v)) {
	std::cout << "当前是 string: " << *p << std::endl;
}
```
(3) `std::visit` (**访问者模式**，**最推荐**)
``` c++
std::visit([](auto&& arg) {
	// 泛型 lambda 能接受所有类型
	using T = std::decay_t<decltype(arg)>;
	if constexpr (std::is_same_v<T, int>) {
		std::cout << "int: " << arg;
	} else if constexpr (std::is_same_v<T, double>) {
		std::cout << "double: " << arg;
	} else if constexpr (std::is_same_v<T, std::string>) {
		std::cout << "string: " << arg;
	}
}, v);

// 或者直接对每种类型写重载函数对象(典型的使用 ovverloaded 技巧)
```
#### 3.4 查询当前类型
``` c++
// 获取当前类型的索引 (0-based)
size_t idx = v.index(); // 返回当前活跃类型的位置
```
#### 3.5 修改值
- 直接赋值：`v = 42` 会切换到 `int` 类型
- 使用 `emplace`：`v.emplace<std::string>("world");` 原地构造
- 使用 `std::in_place_index` 或 `std::in_place_type` 构造
#### 3.6 使用 `std::monostate` 表示"空状态"
``` c++
std::variant<std::monostate, int, std::string> v; // 默认是 monostate(空)
if (v.index() == 0) {
	std::cout << "空状态" << std::endl;
}
v = "hi"; // 现在存储 string
```
### 四、典型应用场景
#### 4.1 函数返回多种类型(如解析器/读取器)
``` c++
std::variant<int, double, std::string> read_value(const std::string& input)
{
	// 尝试解析为 int，失败则 double, 再失败则返回原始字符串
	// ...
}
```
#### 4.2 状态机中的不同状态数据
``` c++
struct Idle {};
struct Running { int elapsed };
struct Paused { int elasped };
using state = std::variant<Idle, Running, Paused>;

void handle(State& s) {
	std::visit([](auto& state) {
		using T = std::decay_t<decltype(state)>;
		if constexpr (std::is_same_v<T, Idle>) { /* ... */ }
		else if constexpr (std::is_same_v<T, Running>) { /* ... */ }
		// ...
	}, s);
}
```
##### 4.3 多态容器的替代（不依赖继承）
``` c++
std::vector<std::variant<int, double, std::string>> vec;
vec.push_back(10);
vec.push_bak(3.14);
vec.push_back("hello");
```
### 五、完整示例代码
``` c++
#include <iostream>
#include <variant>
#include <string>

template<class... Ts>
struct overloaded : Ts... {
    using Ts::operator()...;
};
template<class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

int main() {
    using Var = std::variant<int, double, std::string>;
    Var v = 42;

    std::cout << "当前索引：" << v.index() << std::endl; // 0

    // 使用 visit + overloaded
    std::visit(overloaded {
        [](int i) {
            std::cout << "int: " << i << std::endl;
        },
        [](double d) {
            std::cout << "double: " << d << std::endl;
        },
        [](const std::string& s) {
            std::cout << "string: " << s << std::endl;
        }
    }, v);

    // 修改为 string
    v = "hello variant";
    std::visit([](auto&& arg) {
        std::cout << "通过泛型 lanmbda 打印: " << arg << std::endl;
    }, v);

    // 安全访问
    if (auto p = std::get_if<std::string>(&v)) {
        std::cout << "确实是 string: " << *p << std::endl;
    }

    // 使用 monostate 表示空
    std::variant<std::monostate, int> empty_or_int;
    if (empty_or_int.index() == 0) {
        std::cout << "当前为空" << std::endl;
    }

    empty_or_int = 100;
    std::cout << "现在有 int: " << std::get<int>(empty_or_int) << std::endl;

    return 0;
}

```
### 六、性能与注意事项
- **内存占用**：`variant` 占用的内存等于 **所有可选类型中最大者** 的大小加上一些额外开销(用于存储当前类型索引)。例如，如果类型是`int`(4字节) 和 `std::string` (32字节)，则`variant` 大小至少为 32 字节 + 对齐 + 索引(通常 1 ~ 4 字节)
- **构造**/**析构开销**：切换类型时，会调用旧类型的析构和新类型的构造，开销与类型本身相关
- **不允许引用**、**数组或不完整类型**(但可以包含 `std::reference_warpper<T>`实现引用语义)
- **重复类型**：不能有重复类型(如 `varinat<int, int>` 是错误)，因为无法区分
- **与** [[optional | std::optional]] **关系**：`optional<T>`是`variant<monistate, T>`的特化，但专门优化了空状态访问
### 七、与`std::union`/`std::any`/`std::optional`对比
| 特性     | `std::variant`          | 传统`union` | `std::any`             | [[optional \| std::optional]] |
| ------ | ----------------------- | --------- | ---------------------- | ----------------------------- |
| 类型安全   | ✅编译器检查                  | ❌需手动管理    | ❌运行时类型擦除，访问需`any_cast` | ✅只有 有/无 两种状态                  |
| 支持平凡类型 | ✅                       | ❌(手动析构构造) | ✅                      | ✅                             |
| 固定类型合集 | ✅显式列出                   | ✅显示列出     | ❌任意类型                  | ❌单一类型                         |
| 性能开销   | 较小(类型索引 + 最大空间)         | 最小        | 较大(动态分配、类型擦除)          | 较小(额外 bool 标记)                |
| 访问方式   | `std::visit`、`std::get` | 手动判断      | `std""any_cast`抛异常     | `value()`/`operator*`         |
| 适用场景   | 存储有限几种互斥类型              | 低级内存操作    | 需要存储任意类型(如反射、属性系统)     | 可能缺失的单一值                      |
### 八、C++20/23 补充
- C++20：`std::visit` 支持 `constexpr`
- C++23：`std::variant`新增`std::variant::transform`和`std::variant::and_then`等原子操作(类似 `optional`)，使链式调用更便捷
### 九、常见陷阱与最佳实践
- **避免用** `std::get` **时硬编码索引**：尽量使用类型，如果类型重复则使用索引(但更好是避免重复)
- **当所有类型都相同操作时**，**使用泛型** lambda + `if constexpr` 分支，比重载多个 lambda  更简洁
- **若需要“空状态”**，**优先使用**`std::monostate` **作为第一个类型**，`index()` 为 0 表示空
- **不要滥用**`std::variant`代替继承：如果类型数量不确定或会频繁增加，继承 + 虚函数可能更合适
- **访问异常问题**：`std::get` 抛出异常在某些场景下可能是不期望的，优先使用 `std::get_if` 或 `std::visit` 保证访问安全
---
### 总结
`std::variant` 是一个类型安全、现代的代替传统 `union` 的工具，适合表示 "有限几种互斥类型" 的场景。它比 `std::any` 更高效且类型安全，比  [[optional | std::optional]]  更灵活(支持多种类型)。结合 `std::visit` 和 `overloaded` 模式，可以实现清晰、可维护的多态行为，是现代 C++ 中重要的词汇类型之一。