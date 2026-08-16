### 一、含义
- Resource Acquistion Is Initialization, 资源获取即初始化
### 二、解释
- 将资源的生命周期与对象的生命周期严格绑定
### 三、核心机制
- 获取：在**构造函数**中申请资源(分配内存、打开文件、加锁)
- 释放：在**析构函数**中释放资源(释放内存、关闭文件、解锁)
- 例子
**不使用 RAII（危险写法）：**
```c++
void bad_func() {
    int* p = new int[100]; // 手动申请内存
    if (some_error) {
        return; // 提前返回！内存泄漏发生了，没人 delete[]
    }
    delete[] p; // 正常流程才能释放
}
```
**使用 RAII（标准写法）：**
``` c++
#include <memory>
#include <vector>

void good_func() {
    std::unique_ptr<int[]> p = std::make_unique<int[]>(100); 
    // 构造函数申请好了内存
    
    if (some_error) {
        return; // 提前返回？没问题！
    }
    // 无论函数怎么结束，只要 p 出了这个作用域，它的析构函数自动调用 delete[]
}
```
