### 一、是什么
RTTI(Run-Time Type Information，运行时类型)是c++中一套 **在程序运行时** 识别对象实际类型的机制，它允许程序在只知道基类型指针或引用的情况下，安全地获取其派生类型。
RTTI的核心由以下三部分组成：
- `typeid` **运算符**：获取对象类型信息(`std::type_info`)
- `dybamic_cast`**运算符**：在继承体系中进行安全地多态向下转型(基类->派生类)或跨转型
- `std::type_info`**类**：存储类型的具体信息(如类型名称)
> **注意**：RTTI仅对 **多态类型** (即至少含有一个虚函数的类)生效，因为类型信息存储在虚函数表(vtable)中。对非多态类型，`typeid`和`dynamic_cast`退化为静态判定
### 二、核心机制讲解
#### 2.1 `typeid`运算符
返回`std::type_info`的常量引用，用于比较类型或获取类型名称

``` c++
#include <iostream>
#include <typeinfo>

class Base { public: virtual ~Base() = default; }; // 有虚函数，多态类型
class Derived : public Base {};

int main() {
    Base* ptr = new Derived();

    // typeid(对象)返回动态类型(实际类型)
    std::cout << typeid(*ptr).name() << std::endl; // 输出：Derived(具体名称以来编译器而定)

    //typrid(类型)返回静态类型
    std::cout << typeid(Base).name() << std::endl; // 输出：Base

    // 比较类型
    if (typeid(*ptr) == typeid(Derived)) {
        std::cout << "ptr is of type Derived" << std::endl;
    }

    delete ptr;

    return 0;
}
```
#### 2.2 `dynamic_cast`运算符
安全地将基类型指针/引用转为派生类指针/引用
- **指针版本**：转型失败返回`nullptr`
- **引用版本**：转型失败抛出`std::bad_cast`异常
``` c++
Base* b = new Derived();

// 指针转换(安全)
Drived* d = dynamic_cast<Dervied*>(b);
if (d) {
	std::cout << "转换成功" << std::endl;
} else {	std::cout << "转换失败" << std::endl;
}

// 引用类型(不安全时抛异常)
Base& ref = *b;
try {
	Dervied& dr = dynamic_cast<Dervied&>(ref);
	// 使用dr
} catch (const std::bad_cast& e) {
	std::cout << "引用转换失败" << std::endl;
}

```
### 三、典型使用场景
#### 3.1 安全向下转型(多态容器)
当从基类容器中取出对象，并需要执行派生类特有操作时：
``` c++
#include <iostream>
#include <memory>

class Animal { 
public: 
    virtual ~Animal() = default;
    virtual void speak() const {};
};

class Dog : public Animal {
public:
    void bark() const {
        std::cout << "Woof!" << std::endl;
    }
};

class Cat : public Animal {
public:
    void meow() const {
        std::cout << "Meow!" << std::endl;
    }
};

void handle_animal(const Animal& animal) {
    if (const Dog* dog = dynamic_cast<const Dog*>(&animal)) {
        dog->bark();
    } else if (const Cat* cat = dynamic_cast<const Cat*>(&animal)) {
        cat->meow();
    }
}

int main() {

    Dog dog;
    Cat cat;

    handle_animal(dog); // Output: Woof!
    handle_animal(cat); // Output: Meow!

    return 0;
}
```
#### 3.2 调试与日志(打印类型名称)
``` c++
template<typename T>
void log_object(const T& obj) {
	std::cout << "Process object of type: " << typeid(obj).name() << std::endl;
	// 在gcc/clang中，name()返回修饰名，可用 abi::__cxa_demangle 解码为可读名称
}
```
### 四、完成实例代码
``` c++
#include <iostream>
#include <typeinfo>
#include <memory>

class Shape {
public:
    virtual ~Shape() = default;
    virtual void draw() const = 0;
};

class Circle : public Shape {
public:
    void draw() const override { std:: cout << "Drawing Circle" << std::endl; }
    void radius() const { std::cout << "Radius: 5" << std::endl; }
};

class Square : public Shape {
public:
    void draw() const override { std:: cout << "Drawing Square" << std::endl; }
    void side() const { std::cout << "Side: 10" << std::endl; }
};

void process(Shape* shape) {
    // 1. 使用 typeid 检查类型
    if (typeid(*shape) == typeid(Circle)) {
        std::cout << "This is a Circle" << std::endl;
    }

    // 2. 使用 dynamic_cast 尝试转换
    if (Circle* circle = dynamic_cast<Circle*>(shape)) {
        circle->draw(); // 只有 Circle 才有 radius() 方法
    } else if (Square* s = dynamic_cast<Square*>(shape)) {
        s->side();// 只有 Squard 才有 side() 方法
    }

    shape->draw(); // 多态调用
}

int main() {
    std::unique_ptr<Shape> c = std::make_unique<Circle>();
    std::unique_ptr<Shape> s = std::make_unique<Square>();

    process(c.get());
    process(s.get());
}

```
### 五、性能与开销
- **内存开销**：启用 RTTI 后，每个多态的虚表会额外存储指向 `std::type_info` 的指针(通常增加几个字节)
- **时间开销**：`typeid` 几乎无额外开销(类似取 vtable 指针); `dynamic_cast` 设计字符串比较或指针偏移计算，开销比普通 `static_cast` 大得多(尤其在复杂的继承中)
- **禁用RTTI**：编译选项 `-fno-rtti` (GCC/Clang) 或 `/GR-`(MSVC)可完全关闭RTTI，但会导致 `dynamic_cast` 和 `typeid` 对多态类型不可用(`typeid` 退化为静态类型)
### 六、批判性思考与最佳实践
⚠️**常见问题与设计开销**
- **违反开闭原则**：每当新增派生类，所有使用 `dynamic_cast` 的地方都要修改
- **破坏封装**：业务代码需要知道所有派生类的具体类型，增加了耦合
- **性能陷阱**：在性能敏感的热路径中滥用 `dyname_cast` 可能导致性能下降
✔️**更优替代方案**
1. **虚函数**：绝大多数情况下，直接使用虚函数就能解决问题，无需知道具体类型
2. `std::variant` + `std::visit` (C++ 17)：用于有限集合的静态多态，在编译期确定访问逻辑，性能更改且类型安全
3. **双重派发(访问者模式)**：解决复杂交互问题，避免大量 `dynamic_cast`
4. **自定义类型枚举**：在基类中定义一个 `enum Type { ... }`成员，手动标记类型(常用于游戏引擎，比RTTI更快)
📌**何时可以用RTTI**
- **序列化**/**反序列化框架**：需要根据类型动态创建对象
- **调试工具与日志系统**：打印对象类型辅助排查
- **GUI框架事件处理**：需要区分不同控件的具体行为(但也要谨慎)
---
### 总结
RTTI 是 C++ 提供的一项"安全网"技术，在必要时可能解燃眉之急，但不应成为日常设计的依赖。**好的面向对象设计应尽量让系函数承担职责**，将RTTI限制在架构底层(如 工厂、反射机制) 或辅助调试中。如果发现自己频繁书写 `dynnamic_cast`, 往往是设计需要重构的信号。