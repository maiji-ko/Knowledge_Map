`std::unique_otr` 是C++11引入的 **独占所有权** 智能指针。它在 **栈上** 包装了一个 **堆上分配的对象** 指针，当`unique_ptr` 本身被销毁时 (离开作用域)，它所管理的对象也被自动 `delete`
**核心认知**：`unique_ptr` 是 [[RAII | RAII(资源获取即初始化)]] 原则的完美体现，同时也是你之前学过的 "移动语义" 在实际工程中最经典的应用范例。

---
### 一、为什么需要它？(替代裸指针)
- **杜绝内存泄漏**：无论函数正常返回还是抛出异常，`unique_ptr` 的析构函数都会确保释放资源
- **明确所有权语义**：代码中清晰地标明 "这个内存归我管"，不像裸指针难以区分是 "观察" 还是 "拥有"
- **零开销抽象**：在默认情况下 (无自定义删除器)，`unique_ptr` 的大小与裸指针完全相同 (8字节)，不引入任何额外开销

---
### 二、核心特性：独占所有权与移动语义
`std::unique_ptr` **不能拷贝**，**只能移动**。这直接是C++左右值与移动语义的现实应用：
- **拷贝构造函数和拷贝赋值运算符被显式删除** (`=delete`)
- **移动构造函数和移动赋值运算符被定义**
``` c++
std::unique_ptr<int> ptr1 = std::make_unique<int>(42);
// std::unique_ptr<int> ptr2 = ptr1; // ❌编译错误！不能拷贝(因为会重复释放)

std::unique_ptr<int> ptr3 = std::move(ptr1); // ✅正确！转移所有权
// 此时 ptr1 变为空 (nullptr), ptr3 拥有 42
```
> **回忆左值**/**右值**：`ptr1` 是左值，所以必须用 `std::move` 强制转为右值，才能触发移动构造函数。一旦移动，指针变为 `nullptr`，析构时不会释放内存，保证了 "独占"的安全

---
### 三、基本API操作

| 操作              | 方法                             | 说明                                                |
| --------------- | ------------------------------ | ------------------------------------------------- |
| **创建**          | `std::make_unique<T>(args...)` | C++14起推荐，更安全、更高效(避免重复类型名)                         |
| **解引用**         | `*ptr` / `ptr->`               | 与裸指针语法一致                                          |
| **获取裸指针**       | `ptr.get()`                    | 仅用于观察，不要用它来 `delete` (否则 double free)             |
| **释放所有权**       | `ptr.release`                  | 返回裸指针，并 **放弃管理权** (内存不释放，需手动 delete)。⚠️易导致泄露，务必谨慎 |
| **重置** / **替换** | `ptr.reset(new_val)`           | 销毁当前管理对象，接管新对象(或传 `nullptr` 清空)                   |
| **判空**          | `if(ptr)` 或 `if(!ptr)`         | 支持 `explicit operator bool`                       |
| **交换**          | `ptr1.swap(ptr2)`              | 交换两个智能指针的所有权                                      |
``` c++
#include <iostream>
#include <memory>

struct MyClass {
    int value;
    ~MyClass() {
        std::cout << "释放" << std::endl;
    }
};

int main() {
    auto p = std::make_unique<MyClass>(MyClass{10});
    std::cout << p->value << std::endl; // 10
    std::cout << (p ? "非空" : "空") << std::endl; // 非空

    MyClass* raw = p.release(); // p 变为空，释放权交给 raw
    delete raw; // 必须手动杉树，否则泄露

    p.reset(new MyClass{20}); // 接管新对象，自动清理旧对象 (如果有)
    return 0;
}

```

---
### 四、自定义删除器(Deleter)
`unique_ptr` 允许指定自定义删除逻辑 (处理文件句柄、`malloc/free` 配对等)。注意：自定义删除器会影响类型和大小 (函数指针会增大体积，无状态 lambda 则无额外开销)
``` c++
// 使用 lambda 子当以删除 (推荐，零开销)
auto file_deleter = [](FILE* f) {
	if (f) fclose(f);
}
std::unique_ptr<FILE, decltype(file_deleter)> file_ptr(fopen("test.txt", "w"), file_deleter);
```

---
### 五、数组版本 (`std::unique_ptr<T[]>`)
如果需要管理动态数组，应使用特化版本 (它会调用 `delete[]`)
``` c++
std::unique_ptr<int[]> arr = std::make_unique<int[]>(10); // 分配 10 个 int
arr[0] = 42; // 支持下标访问
// 注意：普通 unique_ptr<T> 不支持下标，且对数组行为未定义，慎用!
```
> **注意**：在C++17之后，除非必须兼容C风格数组，否则更建议使用`std::vector`，它更安全且大小灵活

---
### 六、实战：在容器与函数间传递
#### 6.1 作为函数参数 (多种传参语义)
- **传递所有权** (**进函数**)：按值接收 (需 `std::move`)
- **仅读取**/**修改对象**：传递裸指针 `ptr.get()` 或引用 `*ptr`  (推荐，清晰表达 "非拥有观察者")
- **重新绑定或清空**：传 `std::unique_ptr&` (非典型)
``` c++
void consume(std::unique_ptr<MyClass> p) {
	// 函数拥有 p，离开时释放资源
}

void observe(const MyClass* p) {
	if (p) std::cout << p->value << std::enl;
}

auto p = std::make_unique<MyClass>(5);
observe(p.get()); // 只读，不转移所有权
consume(std::move(p)); // 转移所有权，p变为空
```
#### 6.3 作为函数返回值 (RVO/NRVO 与移动)
``` c++
std::unique_ptr<MyClass> factory(bool flag) {
	if (flag) return std::make::unique<MyClass>(1);
	return std::make_unique<MyClass>(2);
} // 返回值优化 (RVO) 或隐式移动，无需显式 std::move
```
#### 6.4 存入容器
``` c++
std::vector<std::unique_ptr<MyClass>> vec;
vec.push_back(std::make_unique<MyClass>(10));
// vec.push_back(std::make_unique<MyClass>(20)); // C++11 需 move，c++14 后 make_unique 可直接传
```

---
### 七、与之前的知识串联
| 知识点                                                         | 与 `unique_ptr` 的关系                                      |
| ----------------------------------------------------------- | ------------------------------------------------------- |
| [[RAII]] & [[异常处理 \| 异常安全]]                                 | `unique_ptr` 是 [[RAII \| RAII]] 的标杆实践，析构时自动释放，异常场景下不会泄露 |
| **移动语义** ([[左值与右值 \| 左值/右值]])                               | 不可拷贝、只能移动，完美诠释了"将亡值"的资源转移机制                             |
| [[Templates \| 模板 (Templates)]]                             | `unique_ptr` 是一个 **类模板**，依赖编译时类型生成具体代码                  |
| [[auto 和 decltype \| auto/decltype]]                        | 通常用 `auto ptr = make_unique<T>()` 简化书写，推导类型             |
| [[函数模板的参数推导规则 \| 函数模板推导]]                                   | 传参时若形参是 `T&&`，可配合`std::forward` 完美转发 `unique_ptr`       |
| [[variant \| std::variant]] / [[optional \| std::optional]] | 在实际开发中，`variant` 内部常使用 `unique_ptr` 包裹大对象以减少拷贝代价        |

---
### 八、常见陷阱与最佳实践

| 陷阱                                   | 说明与解决                                                                                             |
| ------------------------------------ | ------------------------------------------------------------------------------------------------- |
| **使用** `release()` **后忘记** `delete`  | `release()` 只放弃管理不释放，除非你立即给另一个 `unique_ptr` 管理，否则裸指针需 `delete`，极易泄露。<br>**建议**：非必要不使用 `release()` |
| **用** `get()` **返回的指针删除对象**          | `get()` 只是观察，严禁 `delete ptr.get()`，否则触发 double free (析构时再删除一次)                                    |
| **循环引用** (**与** `shared_ptr` **对比**) | `unique_ptr` 不存在循环引用问题 (独占所有权，不被共享)，这是它与 `shared_ptr` 相比的一大优势                                     |
| **数组特化误用**                           | 不要用 `unique_ptr<T>` 管理数组 (如 `new T[10]`)，应使用 `unique_ptr<T[]>`，或者最好直接用 `vector`                   |
| **自定义删除器与类型兼容性**                     | 带有不同删除器的 `unique_ptr` 是不同的类型，不能互相赋值 (除非删除器类型相同)                                                   |
| **在C接口中传递所有权**                       | 如果 C API 需要 `free`，请使用自定义删除器 `[](void* p){ free(p); }` 搭配 `malloc` 分配                             |

---
### 九、终极示例：工厂模式与多态
``` c++
#include <iostream>
#include <memory>

class Animal {
public:
    virtual void speak() = 0;
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void speak() override {
        std::cout << "Woof" << std::endl;
    }
};

class Cat : public Animal {
public:
    void speak() override {
        std::cout << "Meow" << std::endl;
    }
};

std::unique_ptr<Animal> createAnimal(const std::string& type) {
    if (type == "dog") {
        return std::make_unique<Dog>();
    }
    if (type == "cat") {
        return std::make_unique<Cat>();
    }

    return nullptr;
}

int main() {
    auto pet = createAnimal("dog");
    pet->speak(); // Woof!
    // pet 离开作用域，自动销毁 Dog 对象

    return 0;
}

```

---
### 总结
`std::unique_ptr` 是现代 C++ 内存管理的首选默认工具。它让你从繁琐的 `new` / `delete` 中解放出来，用 **零代价的抽象** 实现了内存安全。牢记它的 **排他性** (不能复制，只能移动)，你将自然而然地写出清晰、健壮且高效的程序。**凡是需要动态分配内存且所有权独占的场景**，**请第一时间考虑**`std::unique_ptr`，**而非裸指针**