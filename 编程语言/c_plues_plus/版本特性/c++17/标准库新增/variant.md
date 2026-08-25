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
#### 4.1 函数返回多种类型
#### 4.2 状态机中的不同状态数据
##### 4.3 多态容器的替代（不依赖继承）
