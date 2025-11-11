## 条款14：在资源管理类中小心 copying 行为

这是 Effective C++ 的第 14 个条款，Think carefully about copying behavior in resource-managing classes，中文翻译为：**在资源管理类中小心 copying 行为。**

当我们自己写一个 RAII 类时，比如书中提到的用于管理互斥锁的 Lock 类：
```cpp
class Lock {
public:
    explicit Lock(Mutex* pm) : mutexPtr(pm) {
        lock(mutexPtr); // 在构造函数中获取资源（加锁）
    }
    ~Lock() {
        unlock(mutexPtr); // 在析构函数中释放资源（解锁）
    }
private:
    Mutex* mutexPtr;
};
```

这个基础的 Lock 类遵循了 RAII 原则，但是当我们复制一个 Lock 对象时，问题就来了：
```cpp
Mutex m; // 一个互斥锁
{
    Lock m1(&m);    // m1 对 m 加锁
    Lock m2(m1);    // m2 是 m1 的一个副本。这里会发生什么？
} // m1 和 m2 在这里离开作用域，它们的析构函数被调用
```

如果我们没有手动定义拷贝构造函数的话，编译器会自动生成一个默认的拷贝构造函数，这个默认的拷贝构造函数只会进行浅拷贝操作，即简单地复制成员变量的值。在这个例子中，`m2` 会得到和 `m1` 相同的 `mutexPtr` 指针，这意味着当 `m1` 和 `m2` 离开作用域时，它们都会尝试解锁同一个互斥锁。这会导致未定义行为，因为同一个互斥锁被解锁了两次。

因此，作者提到，当设计一个 RAII 类时，对于“复制”操作，通常有以下四种选择：
- **禁止复制**：这是最简单、也常常是最安全的选择。如果复制一个资源句柄没有意义（比如锁），或者会带来问题，那就直接禁止它。
  - 适用场景：资源的逻辑本身就不支持被复制或共享，例如 `std::unique_ptr`。
  - 实现方式：通过将拷贝构造函数和拷贝赋值运算符声明为私有（C++11 之前）或删除（C++11 及以后）来禁止复制。
  ```cpp
  class Lock {
  public:
      // ...
      Lock(const Lock&) = delete;
      Lock& operator=(const Lock&) = delete;
  private:
      // ...
  };
  ```
- **引用计数**：允许多个对象共享同一个底层资源，并记录“使用者”的数量。只有当最后一个使用者被销毁时，资源才被真正释放。
  - 适用场景：需要共享资源所有权的场景，例如 `std::shared_ptr`。
  - 实现方式：最简单的方法是在 RAII 类内部使用一个 `std::shared_ptr` 来持有资源。`shared_ptr` 可以接受一个自定义删除器，这让它不仅能管理内存，还能管理任何资源。
  ```cpp
  class Lock {
  public:
      explicit Lock(Mutex* pm)
          : pMutex(pm, unlock) // 使用 shared_ptr，并提供 unlock 作为删除器
      {
          lock(pMutex.get());
      }
      // 不需要手动写析构函数，shared_ptr 会自动管理！
  private:
      std::shared_ptr<Mutex> pMutex;
  };
  ```
  当最后一个 Lock 对象被销毁时，它内部的 `shared_ptr` 的引用计数会变为 0，此时它会自动调用我们提供的 `unlock` 函数来释放锁。
- **深拷贝**：在复制对象时，也完整地复制一份底层资源。这样每个对象都拥有一份独立的、内容相同的资源。
  - 适用场景：当资源本身可以被安全、完整地复制时。例如，一个管理动态字符数组的字符串类。
  - 实现方式：手动实现拷贝构造函数和拷贝赋值运算符，在其中创建新资源并复制内容。
  ```cpp
  // 伪代码示例
  String(const String& other) {
      // 分配新内存
      // 复制 other 中的字符串内容到新内存
  }
  ```
- **转移所有权**：在复制时，将底层资源的所有权从源对象“转移”到目标对象，源对象则不再拥有该资源（通常变为空）。
  - 适用场景：需要严格的独占所有权，但又希望能够转移它。例如 `std::unique_ptr`。
  - 实现方式：在 C++11 之后，通过实现移动构造函数和移动赋值运算符来完成。
  ```cpp
  class UniqueResource {
  public:
      UniqueResource(Resource* res) : resourcePtr(res) {}
      // 移动构造函数
      UniqueResource(UniqueResource&& other) noexcept
          : resourcePtr(other.resourcePtr) {
          other.resourcePtr = nullptr; // 转移所有权
      }
      // 移动赋值运算符
      UniqueResource& operator=(UniqueResource&& other) noexcept {
          if (this != &other) {
              release(); // 释放当前资源
              resourcePtr = other.resourcePtr;
              other.resourcePtr = nullptr; // 转移所有权
          }
          return *this;
      }
  private:
      Resource* resourcePtr;
      void release() {
          if (resourcePtr) {
              // 释放资源
          }
      }
  };
  ```

<Aside type="note">
总结：
- 复制一个 RAII 对象必须一并复制它所管理的资源。因此，底层资源的逻辑特性决定了你的 RAII 类应该采用哪种复制行为。
- 普遍而常见的 RAII 类复制行为是：禁止复制和施行引用计数法。
- 永远不要想当然地依赖编译器生成的默认复制函数，它几乎肯定不是你想要的。
</Aside>