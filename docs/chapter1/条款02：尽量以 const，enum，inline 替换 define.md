## 条款02：尽量以 const，enum，inline 替换 #define

**Prefer consts, enums, and inlines to #defines**

这个条款强调使用 C++ 语言特性来增强代码的健壮性、可维护性和可调试性，而不是依赖 C 语言时代的预处理器。`#define` 并不是 C++ 语言的一部分，而是一个由预处理器处理的简单文本替换工具，这种在编译器介入之前的粗暴替换，会带来很多问题。

以如下代码为例：
```cpp
#define ASPECT_RATIO 1.653
```

在编译器开始编译之前，预处理器便会把代码中所有的 `ASPECT_RATIO` 替换为 `1.653`。记号名称 `ASPECT_RATIO` 根本不会进入编译器的的符号表。如果使用这个变量出错，编译器只会提到 `1.653`，导致调试困难。

对于简单变量，最好的办法是使用 C++ 的 `const` 常量：
```cpp
const double AspectRatio = 1.653; // 替换 #define ASPECT_RATIO 1.653
```
替换后的好处如下：
- `AspectRatio` 是一个受语言支持的常量，它会被编译器看到并放入符号表。
- 调试器可以访问它，错误信息也会显示 `AspectRatio`。
- 代码更小： 使用 `#define`，预处理器会无脑地将 `1.653` 插入到每一个使用点，可能导致目标代码中出现多份 `1.653`；而 `const` 变量通常只会产生一份拷贝。

如果想创建一个只能在类内部使用的常量，`#define` 是无法做到的，因为 `#define` 没有作用域的概念，一旦被 `#define` ，它在后续的编译中就全局可见，除非被 `#undef`。在类里面声明声明静态常量成员可以实现这个目的：
```cpp
class GamePlayer {
private:
    static const int NumTurns = 5; // 常量声明
    int scores[NumTurns];          // ...并使用它
    ...
};
```

如果需要取这个常量的地址，还需要在 `.cpp` 文件中提供一个单独的定义：`const int GamePlayer::NumTurns;`。

一些老旧的编译器不支持在类的内部初始化静态常量成员，这时候可以用 "enum hack" 来代替：
```cpp
class GamePlayer {
private:
    enum { NumTurns = 5 }; // "the enum hack"
    int scores[NumTurns];   // 这就没问题了
    ...
};
```

`NumTurns` 此时是一个枚举值，它在编译期就固定为 5。它的行为很像 const，但有几个特性：
- 更像 `#define`： 你不能取一个 `enum` 值的地址，这保证了它绝对不会有不必要的内存分配。
- 编译期保证： 它是纯粹的编译期常量。

`#define` 除了可以定义常量之外，还可以用来创建看起来像函数的东西，即函数式宏，例如：
```cpp
// 一个调用函数 f，并传入 a 和 b 中较大值的宏
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```
调用示例：
```cpp
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);   // a 被递增了两次！
CALL_WITH_MAX(++a, b+10); // a 甚至可能只被递增一次！
```

在上面这个示例中，宏被展开为`f((++a) > (b) ? (++a) : (b))`。如果 `++a` (值为6) 大于 `b` (值为0)：
- `++a` 被求值（`a` 变为 6）。
- `6 > 0` 为真。
- 执行 `?` 后的部分：`f(++a)`。
- `++a` 再次被求值（`a` 变为 7）。
- 最终调用 `f(7)`。
变量 `a` 被意外地递增了两次。**这是 C++ 中最难排查的错误之一。**

使用 C++ 的模板和 `inline` 可以完美解决这个问题：
```cpp
template<typename T>
inline void callWithMax(const T& a, const T& b) {
    f(a > b ? a : b);
}
```
好处：
- 行为可预测： 这是一个真正的函数。当你调用 `callWithMax(++a, b)` 时，参数 `a` 只会在函数被调用前递增一次。函数体内部使用的是 `a` 和 `b` 的值（这里是引用），不会再次触发递增。
- 类型安全 (Type Safety)： 这是一个模板函数，编译器会进行类型检查。
- 遵守作用域： 它可以是一个普通函数，也可以是类的一个 `private` 成员，完全受 C++ 访问规则的控制。`#define` 无法做到这一点。
- 高效： 它被声明为 `inline`，编译器会尝试对其进行优化，实现与宏同等（甚至更高）的效率，而没有任何宏的副作用。

总结：
- 对于单纯常量，最好以 `const` 对象或 `enums` 替换 `#defines`。
- 对于形似函数的宏，最好改用 `inline` 函数替换 `#defines`。

**虽然我们仍然需要预处理器（如 `#include` 和 `#ifdef` / `#ifndef` 用于头文件保护），但应该尽量避免使用 `#define` 来定义常量和函数式宏。**