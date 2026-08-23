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