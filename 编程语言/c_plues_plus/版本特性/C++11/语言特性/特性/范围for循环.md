范围 for 循环时 C++11 引入的语法糖，用于 **遍历一个序列** (range) **中的所有元素**。它让代码更简洁、更安全，同时与 `auto` 、引用、移动语义等现代特性紧密结合
**核心认知**：范围 for 循环本质上是 **迭代器循环的语法糖** ，但编译器在幕后做了大量工作 (如 `begin` / `end` 查找、类型推导、生命周期延长)，理解这些机制才能避免踩坑

---
### 一、基本语法
``` cpp
for (declaration : expression) {
	// 循环体
}
```
- `expression`：任何人可迭代的对象 (数组、`std::vector`、`std::map`、初始化列表、自定义类型等)
- `declaration`：循环变量声明，通常配合 `auto`、`auto&`、`const auto&`、`auto&&` 使用
**最简单的例子**：
``` cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
for (int x : vec) {
	std::cout << x << ' ';
}
// 输出：1 2 3 4 5
```

---
### 二、背后的展开机制
``` cpp
{
    auto&& __range = expression; // 1. 只求值一次，绑定到万能引用
    auto ___begin = begin(__range); // 2. 获取其实迭代器 (ADL 查找)
    auto __end = end(__range); // 3. 获取结束迭代器 (ADL 查找)
    for (; __begin != __end; ++__begin) {
        declaration = *__begin; // 4. 用当前元素初始化循环变量
        // 5. 执行循环体
    }
}
```
- `expression` **只求值一次**，结果绑定到 `auto&&`(万能引用)，临时对象的生命周期被延长到整个循环结束
- `begin` / `end` 的查找：优先成员函数，否则通过 ADL (参数依赖查找) 找非成员函数，最后回退到 `std::begin` / `std::end` 
- `declaration` 的初始化相当于 **拷贝初始化**：`declaration = *__begin;`。如果 `declaratin` 是引用，则直接绑定，不拷贝

---
### 三、循环变量的四种典型写法 (与 auto 推到强相关)
|写法|推导行为|适用场景|性能|
|---|---|---|---|
|`for (auto x : range)`|拷贝每个元素（值语义）|元素小（`int`、`char`）、需要修改副本|元素大时开销大|
|`for (auto& x : range)`|绑定到元素的**左值引用**，可修改原元素|需要修改原容器元素|无拷贝|
|`for (const auto& x : range)`|绑定到**常量左值引用**，只读|只读遍历，避免拷贝（**最常用**）|无拷贝|
|`for (auto&& x : range)`|**万能引用**，根据 `*__begin` 的值类别推导|泛型代码、处理代理迭代器|
``` cpp
std::vector<std::string> words = {"hello", "world"};

for (auto w : words) {        // 拷贝每个 string，开销大
    w += "!";
}
for (auto& w : words) {       // 引用，可修改原元素
    w += "!";
}
for (const auto& w : words) { // 只读，无拷贝（推荐只读时使用）
    std::cout << w << '\n';
}
for (auto&& w : words) {      // 万能引用，此处推导为 std::string&
    // 可修改，也可用于泛型
}
```
> **回忆左值/右值**：`*__begin` 通常是左值（如 `vector::iterator` 返回 `T&`）。因此 `auto&&` 在范围 for 中通常推导为 `T&`。但对于返回纯右值的代理迭代器（如 `std::vector<bool>`），`auto&&` 会推导为 `T&&`，从而延长代理对象的生命周期。

---
### 四、