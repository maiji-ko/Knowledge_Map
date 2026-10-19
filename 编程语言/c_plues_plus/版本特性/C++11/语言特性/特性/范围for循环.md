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
关键点