## 条款21：必须返回对象时，别妄想返回其 reference

**Don't try to return a reference when you must return an object**

在条款 20 中，我们了解到 pass-by-value的低效，并倾向于使用 pass-by-reference-to-const。但这种思想很容易被过度推广。当一个函数（例如 `operator*`）的职责就是为了计算并创造一个全新的对象时，你必须返回这个新对象。试图通过返回引用或指针来“优化”这个过程，不仅不会带来好处，反而会导致严重的设计缺陷和运行时错误。

书中举了一个 `Rational`（有理数）类的 `operator*`（乘法操作符）的例子。我们的目标是实现 `c = a * b`。这个操作符必须创建一个新的 `Rational` 对象来存放 `a` 和 `b` 的乘积。

一种常见的错误是返回栈局部对象的引用：
```cpp
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d); // 1. result 是一个局部对象
    return result; // 2. 返回了对局部对象的引用
}
```

`result` 是一个局部变量，它存储在栈上。当函数 `operator*` 返回时，`result` 对象会立即被销毁。调用者（例如 `c = a * b;`）得到的是一个**“悬垂引用”**——它指向一块已经被释放、内容未定义的内存。任何对这个引用的后续使用（例如赋值给 c）都会导致未定义行为，通常表现为程序崩溃或数据错乱。

为了避免局部对象被销毁，有人可能会想到在堆上创建对象，这种做法也是错误的：
```cpp
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    // 1. 在堆上创建对象
    Rational* result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d); 
    return *result; // 2. 返回对堆对象的引用
}
```

错误的原因：
- 内存泄漏： 调用者只收到了一个 `Rational&`（引用），它根本不知道这个对象是在堆上分配的，因此也无法 `delete` 它。
- `Rational c = a * b;` 这样的代码会泄漏内存。
- 像 `Rational w = a * b * c * d;` 这样的链式调用，每调用一次 `operator*` 就会 `new` 一个对象，导致多次内存泄漏。这是一个严重的设计灾难。

还有一种常见的错误是返回对静态局部对象的引用，这种做法在单线程的简单测试中看似可行，但隐藏着巨大的逻辑漏洞：
```cpp
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
    static Rational result; // 1. 静态局部对象
    
    // 2. 修改这个唯一的静态对象
    result = ...; // (计算 lhs * rhs 并赋值给 result)
    
    return result; // 3. 返回对静态对象的引用
}
```

原因：
- 逻辑错误（核心问题）： `static` 意味着 `result` 对象全局只有一个实例。所有的 `operator*` 调用都共享这一个实例。
- 书中举了一个经典的例子：`if ((a * b) == (c * d))`
  - `(a * b)` 被计算。`static result` 被设置为 `a*b` 的值。
  - `(c * d)` 被计算。同一个 `static result` 被修改，覆盖了 `a*b` 的值，现在它等于 `c*d` 的值。
  - `operator==` 被调用，它比较的是 `(c*d)` 和 `(c*d)`（因为 `result` 对象现在的值是 `c*d`）。
  - 结果 `if` 语句永远为 `true`，无论 `a`, `b`, `c`, `d` 是什么。
- 线程安全问题： 在多线程环境中，如果两个线程同时调用 `operator*`，它们会同时读写这个 `static result`，导致数据竞争和彻底的混乱。

因此，当函数需要创建一个新对象时，正确的做法应该是按值返回这个新对象：
```cpp
inline const Rational operator*(const Rational& lhs, const Rational& rhs)
{
    // 1. 计算结果，并将其作为 "临时对象" (rvalue) 返回
    // 这通常会调用 Rational 的构造函数
    return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}

// 在 C++11 之后, 甚至可以省略 Rational 构造函数调用，使用列表初始化
// return {lhs.n * rhs.n, lhs.d * rhs.d};
```

这种写法并不是低效的，现代 C++ 编译器会使用返回值优化(Return Value Optimization, RVO)技术来跳过多余的拷贝，在调用方的内存空间中构造这个返回的对象。因此，
按值返回在语义上是正确的，并且在现代编译器中效率也是极高的（几乎没有拷贝开销）。

总结：
- 绝不要返回一个指向局部栈对象的指针或引用。
- 绝不要返回一个指向一个在函数内部 `new` 出来的堆对象的引用（这会导致内存泄漏）。
- 绝不要返回一个指向一个静态局部对象的引用，除非你非常确定你想要这种“全局共享”的怪异行为，并且已经处理了随之而来的线程安全和逻辑陷阱。
- 当一个函数必须返回一个新创建的对象时，就让它按值返回。相信编译器的返回值优化 (RVO) 会处理好效率问题。