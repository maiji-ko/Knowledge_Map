### 一、是什么
`std::optional<T>` 是一个模板类，它表示一个 **可能包含** 类型`T`的值，也可能 **不包含任何值** (即"空"状态)。它常用于那些"可能返回无效结果"的函数，用来代替错误码、空指针或哨兵值(如 `-1`, `nullptr`)
### 二、为什么需要它
- **明确表达意图**：函数签名直接返回表明返回值可能缺失，提高可读性
- **类型安全**：避免使用`nullptr`或特殊数值带来的歧义和运行时错误
- **避免额外输出参数**：不用再通过引用参数返回结果，使接口更简洁
- **与标准库无缝配合**：支持迭代器、范围 for 循环等
### 三、基本用法
#### 3.1 包含头文件
```c++
#include <optional>
```
#### 3.2 创建 `optional`对象
``` c++
std::optional<int> o1; // 空(未初始化)
std::optional<int> o2 = 42; // 包含42
std::optional<int> o3 = std::nullopt; // 显示置空
std::optional<int> 04 { std::in_place, 10 }; // 原地构造(避免拷贝)
```
#### 3.3 检查是否有值
- `has_value()` 或 `operator bool`:
``` c++
if (o2.has_value()) { ... }
if (o2) { ... } // 更简洁
```
#### 3.4 获取值
- `value`: 返回引用，若为空则抛出`std::bad_optional_access`异常
- `operator*`和`operator->`：直接访问，但 **不检查空值** (行为未定义，需自己确保有值)
- `value_or(default)`：如果有值则返回该值，否则返回默认值
``` c++
std::cout << o2.value() // 42
std::cout << o1.value_or(0) // 0 (因为 o1 为空)
```
#### 3.5 修改值
- 赋值操作: `opt = 5;` 或 'opt = std::nullopt;' 置空
- `emplace(arg...)`：原地构造新值
- `reset()`：置空
#### 3.6 比较
`optional`支持`==`、`!=`、`<`等比较运算符，空值之间相等，有值按`T`的比较规则
### 四、典型应用场景
#### 4.1 可能失败的数据查找
``` c++
std::optional<int> find_value(const sdt::vector<int>& vec, int target) {
	for (int v : vec) {
		if (v == target) return v; // 隐式构造 optional
	}
	return std::nullopt; // 未找到
}

// 使用
if (auto res = find_value(data, 42)) {
	std::cout << "Found" << *res << '\n';
} else {
	std::cout << "Not found\n";
}
```
#### 4.2 可选的配置参数
``` c++
struct Config {
	std::optional<int> timeout; // 未设置则使用默认
	std::optional<std::string> log_file;
}
```
#### 4.3 避免使用输出参数
``` c++
// 旧风格：通过 bool 返回成功，结果通过引用传出
bool parse_int(const std::string& s, int& out);

// 新风格：直接返回 optional
std::optional<int> parse_int(const std::string& s) {
	try { return std::stoi(s); }
	catch (...) { return std::nullopt; }
}
```
### 五、完成示例代码
``` c++
#include <iostream>

#include <optional>

#include <vector>

#include <string>

  

// 查找第一个大于阈值的值

std::optional<int> first_greater(const std::vector<int>& nums, int threshold) {

    for (int n : nums) {

        if (n > threshold) return n;

    }

    return std::nullopt;

}

  

int main() {

    std::vector<int> data = {3, 1, 4, 1, 5, 9};

  

    auto result = first_greater(data, 5);

    if (result) {

        std::cout << "Found: " << *result << std::endl; // 9

    } else {

        std::cout << "No value" << std::endl;

    }

  

    // 使用 value_or

    auto result2 = first_greater(data, 10);

    std::cout << "Result: " << result2.value_or(-1) << std::endl; // -1

  

    // 修改 optional

    std::optional<std::string> name;

    name = "Alice";

    if (name) std::cout << "Name: " << *name << std::endl; // Alice

  

    name.reset(); // 置空

    std::cout << "Empty: " << (name ? "有值" : "空") << std::endl; // 空

    return 0;

}
```### 一、是什么
`std::optional<T>` 是一个模板类，它表示一个 **可能包含** 类型`T`的值，也可能 **不包含任何值** (即"空"状态)。它常用于那些"可能返回无效结果"的函数，用来代替错误码、空指针或哨兵值(如 `-1`, `nullptr`)
### 二、为什么需要它
- **明确表达意图**：函数签名直接返回表明返回值可能缺失，提高可读性
- **类型安全**：避免使用`nullptr`或特殊数值带来的歧义和运行时错误
- **避免额外输出参数**：不用再通过引用参数返回结果，使接口更简洁
- **与标准库无缝配合**：支持迭代器、范围 for 循环等
### 三、基本用法
#### 3.1 包含头文件
```c++
#include <optional>
```
#### 3.2 创建 `optional`对象
``` c++
std::optional<int> o1; // 空(未初始化)
std::optional<int> o2 = 42; // 包含42
std::optional<int> o3 = std::nullopt; // 显示置空
std::optional<int> 04 { std::in_place, 10 }; // 原地构造(避免拷贝)
```
#### 3.3 检查是否有值
- `has_value()` 或 `operator bool`:
``` c++
if (o2.has_value()) { ... }
if (o2) { ... } // 更简洁
```
#### 3.4 获取值
- `value`: 返回引用，若为空则抛出`std::bad_optional_access`异常
- `operator*`和`operator->`：直接访问，但 **不检查空值** (行为未定义，需自己确保有值)
- `value_or(default)`：如果有值则返回该值，否则返回默认值
``` c++
std::cout << o2.value() // 42
std::cout << o1.value_or(0) // 0 (因为 o1 为空)
```
#### 3.5 修改值
- 赋值操作: `opt = 5;` 或 'opt = std::nullopt;' 置空
- `emplace(arg...)`：原地构造新值
- `reset()`：置空
#### 3.6 比较
`optional`支持`==`、`!=`、`<`等比较运算符，空值之间相等，有值按`T`的比较规则
### 四、典型应用场景
#### 4.1 可能失败的数据查找
``` c++
std::optional<int> find_value(const sdt::vector<int>& vec, int target) {
	for (int v : vec) {
		if (v == target) return v; // 隐式构造 optional
	}
	return std::nullopt; // 未找到
}

// 使用
if (auto res = find_value(data, 42)) {
	std::cout << "Found" << *res << '\n';
} else {
	std::cout << "Not found\n";
}
```
#### 4.2 可选的配置参数
``` c++
struct Config {
	std::optional<int> timeout; // 未设置则使用默认
	std::optional<std::string> log_file;
}
```
#### 4.3 避免使用输出参数
``` c++
// 旧风格：通过 bool 返回成功，结果通过引用传出
bool parse_int(const std::string& s, int& out);

// 新风格：直接返回 optional
std::optional<int> parse_int(const std::string& s) {
	try { return std::stoi(s); }
	catch (...) { return std::nullopt; }
}
```
### 五、完成示例代码
``` c++
#include <iostream>
#include <optional>
#include <vector>
#include <string>

// 查找第一个大于阈值的值
std::optional<int> first_greater(const std::vector<int>& nums, int threshold) {
    for (int n : nums) {
        if (n > threshold) return n;
    }
    return std::nullopt;
}

int main() {
    std::vector<int> data = {3, 1, 4, 1, 5, 9};

    auto result = first_greater(data, 5);
    if (result) {
        std::cout << "Found: " << *result << std::endl; // 9
    } else {
        std::cout << "No value" << std::endl;
    }

    // 使用 value_or
    auto result2 = first_greater(data, 10);
    std::cout << "Result: " << result2.value_or(-1) << std::endl; // -1

    // 修改 optional
    std::optional<std::string> name;
    name = "Alice";
    if (name) std::cout << "Name: " << *name << std::endl; // Alice

    name.reset(); // 置空
    std::cout << "Empty: " << (name ? "有值" : "空") << std::endl; // 空
    return 0;
}
```
### 六、笔记关系
- 属于[[c++17 | c++ 17标准库新增]]
