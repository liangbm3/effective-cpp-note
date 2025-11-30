## 条款25：考虑写一个不抛异常的 swap 函数

**Consider support for a non-throwing swap.**

C++ 标准库提供的默认 std::swap 实现是非常通用的，它的逻辑大致如下：
```cpp
// 默认实现：基于拷贝
template<typename T>
void swap(T& a, T& b) {
    T temp(a); // 1. 调用拷贝构造
    a = b;     // 2. 调用拷贝赋值
    b = temp;  // 3. 调用拷贝赋值
}
```
对于包含大量资源的类，这种深拷贝是极其浪费的。我们明明只需要交换两个指针的值，却不得不复制整个庞大的数据结构。这时候我们需要手动去实现 `swap` 函数，来提高性能。而这个条款要讲的就是在实现自定义的 `swap` 函数时，务必确保它不会抛出异常。

为了让自定义 `swap` 既高效又能被标准库（如 `std::vector`, `std::sort`）正确识别，我们需要遵循特定的模式。
- 提供一个 `public swap` 成员函数
  因为我们要交换的是 `private` 指针，所以必须由成员函数来完成。此函数绝不可抛出异常（通常加上 `noexcept`）。
  ```cpp
  class Widget {
  public:
      void swap(Widget& other) noexcept { // 高效交换内部数据
          using std::swap;
          swap(pImpl, other.pImpl); // 只是交换指针
      }
  private:
      WidgetImpl* pImpl;
  };
  ```
- 提供一个非成员 `swap` 函数
  为了让 `Widget` 支持 `swap(a, b)` 这种自然的调用方式，我们需要定义一个全局（或同命名空间）的 `swap`。
  - 如果 `Widget` 是普通类（非模板）：我们可以特化 `std::swap`。
    ```cpp
    namespace std {
      template<> // 全特化
      void swap<Widget>(Widget& a, Widget& b) {
          a.swap(b); // 调用成员函数
      }
    }
    ```
  - 如果 `Widget` 是类模板，C++ 是不允许偏特化函数模板的。而且，C++ 标准禁止向 std 命名空间添加新的重载函数（虽然全特化是可以的）。解决方案是在 `Widget` 所在的同一个命名空间内，定义一个非成员 `swap` 函数。
    ```cpp
    namespace WidgetStuff {
    template<typename T>
    class Widget { ... };

    // 在同一个命名空间内定义 non-member swap
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b) {
        a.swap(b);
    }
  }
  ```
  这利用了 ADL (Argument-Dependent Lookup，实参依赖查找) 规则：编译器在看到 `swap(w1, w2)` 时，会去 `w1` 所在的命名空间查找是否有 `swap`。
- 第三步：正确地调用 `swap`
  客户端应该正确编写代码确保调用到最高效的版本。
  - 错误的调用：
    ```cpp
    std::swap(obj1, obj2); // 强制使用 std::swap，可能会错过我们在 WidgetStuff 中定义的优化版本
    ```
  - 正确的调用：
    ```cpp
    using std::swap; // 让 std::swap 可见 (作为备选)
    swap(obj1, obj2); // 让编译器去挑最好的：
                      // 1. 如果有专属的 non-member swap (通过 ADL 找到)，用它。
                      // 2. 如果有 std::swap 的全特化版本，用它。
                      // 3. 都没有，才退而求其次使用 std::swap 的默认版本。
    ```

这个条款特意强调了不抛异常，原因是 `swap` 是实现 “强异常安全保证” (Strong Exception Guarantee) 的核心技术。在“先修改副本，确信没问题了再交换回去”的策略中（即 Copy-and-Swap 手法），`swap` 是最后一步提交动作。如果 `swap` 这一步抛出了异常，程序的状态就会不一致，导致异常安全崩塌。

总结：
- 如果 `std::swap` 的默认行为对自定义的类效率低下，请提供一个 `public swap` 成员函数，并在其中高效地置换内部数据。这个函数绝不能抛出异常。
- 如果提供了一个 `member swap`，请同时提供一个 `non-member swap` 来调用前者。
  - 对于普通类，可以特化 `std::swap`。
  - 对于类模板，请在类所在的同一个命名空间内定义非成员 `swap` 函数（推荐对普通类也这么做，因为更统一且无需触碰 `std`）。
- 调用 `swap` 时，请永远使用 `using std::swap; swap(obj1, obj2);` 的模式，以确保编译器能利用 ADL 找到优化的版本。