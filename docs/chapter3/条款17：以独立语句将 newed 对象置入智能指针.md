## 条款17：以独立语句将 newed 对象置入智能指针

这是 Effective C++ 的第 17 个条款，Store newed objects in smart pointers in standalone statements，中文翻译为：**以独立语句将 newed 对象置入智能指针。**

文章举了一个经典的例子来说明这个规则的重要性：
```cpp
// 假设有这两个函数
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);
```
如果使用这种看起来很紧凑的调用方式：
```cpp
processWidget(std::shared_ptr<Widget>(new Widget()), priority()); // 危险！可能导致内存泄漏
```

这一行代码是非常危险的，因为 **C++ 编译器拥有重新排列函数参数求值顺序的自由**。在 `processWidget` 被真正调用之前，编译器必须完成三项工作：
- 执行 `new Widget()`，在堆上创建一个 `Widget` 对象。
- 调用 `priority()` 函数，获取一个 `int` 结果。
- 调用 `std::shared_ptr` 的构造函数，用 `new Widget()` 返回的原始指针来创建一个智能指针。

C++ 标准只保证 `new Widget()` 一定在 `std::shared_ptr` 的构造函数调用之前执行，但对于 `priority()` 的调用时机，标准并没有规定。这意味着，编译器可能会选择以下这个危险的执行顺序：
- `new Widget()`：在堆上成功创建了一个 `Widget` 对象，一个指向它的原始指针被生成。
- `priority()`：接着执行这个函数。
- `std::shared_ptr` 构造函数：最后才用第 1 步的原始指针去构造智能指针。

按照上面这个顺序，如果在第二步，即 `priority()` 函数内部抛出了一个异常。根据 C++ 的异常处理机制，函数调用栈会开始回滚，这时第一步中由 `new Widget()` 创建的那个原始指针将会丢失，同时第三步 `std::shared_ptr` 的构造函数永远也不会执行。最终造成的结果是丢失了唯一指向堆上 `Widget` 对象的指针。这块内存再也无法被 `delete`，造成了**内存泄漏**。

我们本想用智能指针来防止资源泄漏，但在这个执行顺序下，资源在被智能指针接管之前就因为异常而泄漏了。

对于这个问题的解决方案，也就是这个条款所强调的，**要用一条独立的语句完成智能指针的初始化**：
```cpp
// 安全的做法
std::shared_ptr<Widget> pw(new Widget()); // 第1步：创建对象并立刻用智能指针管理它

processWidget(pw, priority());            // 第2步：将已创建好的智能指针和另一个参数传入函数
```

C++ 保证语句之间是按顺序执行的。第一行代码必须完全执行完毕，才会开始执行第二行。对于这种写法：
- 在第一行 `std::shared_ptr<Widget> pw(new Widget());` 中。如果 `new Widget()` 失败（例如内存不足抛出 `std::bad_alloc`），`pw` 根本不会被创建，一切正常。如果 `new Widget()` 成功，那么返回的原始指针会立即被用来构造 `pw`。这个操作在一条语句内是不可分割的。此时，`Widget` 资源已经安全地被 `pw` 管理起来了。
- 程序执行第二行 `processWidget(pw, priority());`时。即使 `priority()` 抛出异常，栈回滚机制也会正确地销毁局部变量 `pw`。`pw` 的析构函数会负责 `delete` 它所管理的 `Widget` 对象，不会发生任何资源泄漏。

实际上，现代 C++ 提供了比直接使用 `new` 更好的工具来创建智能指针：`std::make_unique` 和 `std::make_shared`。使用 `std::make_shared` 可以完全避免上面讨论的问题，并且写法更简洁、更安全，甚至可能更高效。
```cpp
// 最佳实践：使用 std::make_shared
processWidget(std::make_shared<Widget>(), priority()); // 完全安全，并且推荐！
```

之所以 `std::make_shared` 更好，是因为 `std::make_shared` 是异常安全的，它像一个函数调用，它在内部完成了对象的创建和智能指针的构造。编译器无法将 `priority()` 的调用插入到这两个操作之间，从而从根本上杜绝了资源泄漏的风险。还有一个更重要的原因是效率更高：`std::shared_ptr` 需要一个额外的“控制块”来存储引用计数等信息。
- `std::shared_ptr<Widget>(new Widget())` 通常会导致两次堆内存分配：一次为 `Widget` 对象（在 `new` 中），一次为控制块（在 `shared_ptr` 构造函数中）。
- `std::make_shared<Widget>()` 则足够智能，它会进行一次大的堆内存分配，同时容纳 `Widget` 对象和控制块。这减少了内存分配的开销，性能更好。

<Aside type="note">
总结：
- **核心规则**：以独立语句将 `new` 创造的对象存储于智能指针中。
- **现代实践**：优先使用 `std::make_unique` 和 `std::make_shared`，而不是直接使用 `new`。这不仅能解决本条款讨论的异常安全问题，还能带来更好的性能。
</Aside>