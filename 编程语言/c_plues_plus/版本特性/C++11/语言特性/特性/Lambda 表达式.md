Lambda 表达式是C++引入的 **匿名函数对象** (闭包)。它允许你在需要函数的地方 (如算法问题、排序规则) **就地定义函数体**，而无需额外编写一个具名函数或函数类
**本质揭秘**：编译器每遇到一个Lambda，都会在幕后生成一个 **独一无二的**、**重载了** `operator()` **的类** (即仿函数)。这个类的实例就是 "闭包对象"

---
### 一、为什么需要它
- **简化 STL 算法**：用几行代码写出自定义排序、查找规则，无需单独写函数或结构体
- **捕获上下文变量**：可以直接使用当前作用域内的局部变量 (这是普通函数做不到的)
- **配合** `std::variant` **的** `std::visit`：这是处理 `variant` 当期活跃类型最优雅的方式
- **移动语义落地**：C++14起支持捕获 `std::unique_ptr` (移动捕获)，完美体现"独占所有权转移"

---
### 二、核心语法结构
完整的 Lambda 表达形式如下 (其中大部分可以省略)
``` c++
[capture](paramters) -> return_type { body }
```

| 部件               | 说明                        | 是否必须                        |
| ---------------- | ------------------------- | --------------------------- |
| `[capture]`      | **捕获子句**：指定如何捕获外部变量(值、引用) | **必须**                      |
| `(parameters)`   | **参数列表**：类似普通函数参数         | 可选 (无参可省略括号，除非指定 `mutable`) |
| `-> return_typr` | **返回类型**：尾置返回类型           | 可选(若编译器能推导则省略)              |
| `{body}`         | **函数体**：执行的具体逻辑           | **必须**                      |
**最简单的例子**：
``` c++
auto greet = []() { std::cout << "Hello" << std::endl; }

greet(); // 调用
```

---
### 三、捕获子句详解 (重要)
捕获决定了Lambda 如何访问外部变量，这和 "左值/右值" 及 "所有权" 紧密相关
#### 3.1 按值捕获 `[=]` 与 按引用捕获 `[&]`
- `[=]`  **(默认值捕获)**：可瓯北外部变量的值到闭包内。**注意**：拷贝发生在定义时，且默认是 `const` 只读 (不可修改)
- `[&]` **(默认引用捕获)**：以引用方式捕获外部变量。**注意**：必须保证 Lambda 执行时，被引用的对象仍存活 (否则悬垂引用)
``` c++
int x = 10;
auto f1 = [=]() { return x; }; // 拷贝 x, x 变不影响内部
auto f2 = [&]() { return ++x; }; // 引用 x, 修改影响外部
f2();
std::cout << x << std::endl; // 输出 11

```
#### 3.2 显式指定单个变量 `[a, &b]`
精细控制，推荐优先使用 (更清晰、更安全)：
``` c++
int a = 1, b = 2;
auto f = [a, &b]() {
	// a 只读，b可修改外部
	b += a;
};
f(); // b 变为 3
```
#### 3.3 修改值捕获的副本 (`mutable`)
默认按值捕获的是常量副本。若想在 Lambda 体内副本而不影响外部，加 `mutable` 关键字
``` c++
#include <iostream>

int main() {
    int count = 0;
    auto increment = [count]() mutable {
        return ++count; // 修改的是副本
    };

    std::cout << increment() << std::endl; // 返回 1
    std::cout << increment() << std::endl; // 返回 2 (内部副本持续累积)

    std::cout << count << std::endl; // 外部仍为0
}
```
#### 3.4 C++14 初始化捕获 (移动捕获) -- 核心
允许在捕获子句中 **定义并初始化** 一个成员变量，支持移动语义。这对于 `std::unique_ptr` 至关重要
``` c++
std::unique_ptr<int> ptr = std::make_unique<int>(42);
// C++11 无法将 unique_ptr 直接捕获 (只能引用或拷贝)，C++14 可以移动
auto take_ownership = [ p = std::move(ptr)]() {
	std::cout << *p << std::endl;
}
// 此时 ptr 已变为空，所有权转交给 Lambda 内部成员
take_ownership();
```

---
### 四、参数列表与的泛型 Lambda (C++14)
#### 4.1 通常的写法
``` c++
auto add = [](int a, int, b) -> int {
	return a + b;
}
// 返回值也可省略 (编译器推导为 int)
auto add2 = [](int a, int b) {
	return a + b;
}
```
#### 4.2 泛型 Lambda (C++14)
使用 `auto` 作为参数类型，编译器会自动生成模板化的 `operator`。这直接依赖 [[函数模板的参数推导规则 | 函数模板参数推导规则]]：
``` c++
auto generic = [](auto x, auto y) {
	return x + y;
}
std::cout << generic(3, 5); // int
std::cout << generic(3.1, 2.2); // double
// 甚至 generic("a", "b"); 编译错误，因为指针不能相加，完美体现了模板推导的即时性
```

---
### 五、作为函数返回值与存储
- **用** `auto` **存储**：每个 Lambda 类型唯一，用 `auto` 完美保留类型 (零开销)
- **用** `std::function` **存储**：会进行类型擦除，有轻微性能开销 (堆分配可能发生)
- **返回** Lambda：直接返回 `auto` 即可 (得益于移动语义或 RVO)
``` c++
// 返回一个加法 Lambda
auto make_adder(int n) {
	return [n](int x) {
		return x + n; // 捕获 n
	};
	
	auto add5 = make_adder(5);
	std::cout << add5(10); << std::endl; // 15
}
```

---
### 六、实战：结合之前的 [[variant | std::variant]] 与 `std::visit`
这是现代 C++ 中 Lambda 最酷的应用之一。配合 `ovverloaded` 技巧 (继承重载)，可以实现模式匹配：
``` c++
#include <iostream>
#include <variant>

using Var = std::variant<int, double, std::string>;

template<class... Ts>
struct overloaded : Ts... {
    using Ts::operator()...;
};

template<class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

int main() {
    Var v = 3.14;
    std::visit(overloaded{
        [](int i) {
            std::cout << "int: " << std::endl;
        },
        [](double d) {
            std::cout << "double: " << d << std::endl;
        },
        [](const std::string& s) {
            std::cout << "str: " << s << std::endl;
        }
    }, v); // 输出 double: 3.14
}

```

---
### 七、性能考量 (对比 `std::function`)
- **无捕获或值捕获** Lambda：编译器完全内联，生成的机器码等同于手动写的函数调用，**零开销**
- **按引用捕获**：内敛后等同于直接操作原变量，也几乎无开销
- **存储在** `std::function` **中**：会触发类似擦除和可能的动态内存分配，调用时也可能无法内联。**性能优先场景**，尽量用 `auto` 或模板 (如 `tempalte<typename Func>)` 接收 Lambda

---
### 八、常见陷阱与最佳实践

| 陷阱                          | 说明与对策                                                                                                         |
| --------------------------- | ------------------------------------------------------------------------------------------------------------- |
| **悬垂引用**                    | `[&]` 捕获了栈变量，但 Lambda 被延迟执行 (如存入线程回调) 时，原变量已销毁。**对策**：优先用 `[=]` (值捕获) 或显式移动捕获                                 |
| **误用** `[=]` 修改值            | `[=]` 默认是 `const`，无法修改内部拷贝。**对策**：加 `mutable`，或改用显式传参                                                         |
| **在循环中捕获引用**                | `for (auto& x : vec) { auto f = [&x]() { ... }; }` 如果 `f` 在循环外使用，`x` 可能失效。**对策**：按值捕获 `[x]`                   |
| **过度捕获**                    | 使用 `[=]` 会拷贝所有可见变量，低效且可能意外拷贝大对象。**对策**：显式列出所需变量 `[a, &b]`                                                     |
| **默认捕获** `[=]` **与** `this` | C++20 前，`[=]` 会隐式捕获 `this` 指针 (即引用方式)，可能引发悬垂。**对策**：C++20 推荐使用 `[=, this]` 显式声明，或 `[*this]` (C++17) 值捕获当前对象副本 |

---
### 九、与之前概念串联

| 知识点                                                    | 关联点                                                    |
| ------------------------------------------------------ | ------------------------------------------------------ |
| [[函数模板的参数推导规则 \| 函数模板推导]]                              | 泛型 Lambda 的 `auto` 参数，本质就是 `template<typename T>` 的语法糖 |
| [[左值与右值 \| 左值/右值]] & [[unique_ptr \| unique_ptr]]      | C++14 的移动捕获 `[p = std::move(p)]` 是右值转发的直接应用            |
| [[optional \| std::optional]] / [[variant \| variant]] | `std::visit` 必须配合 Lambda 访问器使用                         |
| [[RTTI \| RTTI(运行时类型识别)]]                              | Lambda 彻底抛弃了运行时类型识别，在编译器确定类型和调用，性能远超虚函数                |

---
### 总结
Lambda 表达式是 C++ 迈向 "现代化、高效、表现力" 的关键一步。**黄金法则**：默认优先使用 **显式捕获列表**  (`[a, &b]`) 而非默认捕获 ( `[=]` 或 `[&]`)；若需移动大对象或 `unique_ptr`，坚决使用 **C++14** **的初始化捕获**；当需要将 Lambda 作为参数传递时，优先使用模板函数 ( `template<typename F>` ) 以零开销接收，而非 `std::function`。