## 条款03：尽可能使用 const

**Use const whenever possible**

在 C++ 中，`const` 关键字用于告诉编译器和程序员某个值或对象不应该被修改，这能帮助编译器捕获错误，并且让代码更易于理解和维护。如果将某个东西声明为 `const` 后，如果在代码中无意中违反了这个承诺，编译器就会立即报错，可以将很多潜在的运行时错误提前在编译器发现。

文中介绍提到了 `const` 常见的一些使用场景，例如可以用来修饰指针：
- `const char* p` (或 `char const* p`): 指向**常量的指针 (Pointer to const data)** 。不能通过 `*p` 来修改 `p` 所指向的数据（例如 `*p = 'H';` 会报错）。但可以改变指针 `p` 本身的指向（例如 `p = another_buffer`;）。
- `char* const p`: **常量指针 (Const pointer)** 。可以通过 `*p` 修改所指向的数据（例如 `*p = 'H';` 是允许的）。但不能改变指针 `p` 的指向（例如 `p = another_buffer;` 会报错）。它必须在声明时初始化。
- `const char* const p`: **指向常量的常量指针 (Const pointer to const data)** 。指针的指向和所指的数据都不能被修改。

> 记忆：`const` 关键字出现在星号左边，表示被指物是常量，`const` 关键字出现在星号右边，表示指针自身是常量。

对于 STL 迭代器：
- `const_iterator`：指向常量内容的迭代器
  - 声明示例：`std::vector<int>::const_iterator it;`。
  - 模拟对象：相当于 `const T* ptr`。
  - 用途：遍历一个容器时，如果只想读取元素，而不希望修改元素，应该使用`const_iterator`。
- `const_iterator`：本身是常量的迭代器
  - 声明示例：`const std::vector<int>::iterator iter`。
  - 模拟对象：相当于 `T* const ptr`。
  - 用途：相对少见，迭代器作为标记，这个标记指向容器中的某个固定位置，且这个标记不能移动。

> 在 C++ 11 及以后的版本，更倾向于使用 `auto` 和 `cbegin()/cend()`。

对于引用: 引用本身就类似于一个 `T* const`（常量指针），它一旦绑定就不能更改。因此 `const` 只用于修饰它所引用的数据，如 `const char& ref`。

在声明函数时，参数和返回值也可以使用 `const` 修饰：
- 参数：
  - 示例：`void func(const std::string& s)`
  - 这是 C++ 中最高效、最安全的传递对象的方式之一（pass-by-reference-to-const）。`&` 避免了拷贝对象的开销。`const` 保证了函数内部不会修改传入的对象 `s`。
- 返回值：
  - 示例：`const Rational operator*(...)`
  - 将返回值声明为 `const` 可以防止一些无意义的赋值操作。例如，如果 `a` 和 `b` 都是 `Rational` 类型，`a * b` 的结果是 `const`，那么 `(a * b) = c;` 这样的代码（很可能是一个手误，本意是 `==`）在编译时就会失败，从而能够在编译时发现错误。

`const` 还有一个重要的用途便是 `const` 成员函数，通过在成员函数参数列表后加上 `const`，如 `char& operator[](...) const;`，这个 `const` 成员函数承诺不会修改器对象的逻辑状态，同时 `const` 对象只能调用 `const` 成员函数。`const` 成员函数允许我们对 `const` 对象和非 `const` 对象进行重载。文中的例子如下：
```cpp
class TextBlock {
public:
    // non-const 对象调用这个版本
    char& operator[](std::size_t pos) { return text[pos]; }
    
    // const 对象调用这个版本
    const char& operator[](std::size_t pos) const { return text[pos]; }
private:
    std::string text;
};

TextBlock tb("Hello");
const TextBlock ctb("World");

tb[0] = 'h';    // OK，调用 non-const 版本，返回 char&
ctb[0] = 'w';   // 错误！ctb 调用 const 版本，返回 const char&，不能赋值
```

在文章中，作者对编译器如何理解 `const` 成员函数做了进一步的探讨：
- **Bitwise Constness (位常量性)** ：这是编译器的定义。`const` 成员函数绝对不能修改对象内的任何一个 `bit` (数据成员)，`static` 成员除外。
- **Logical Constness (逻辑常量性)** ：这是程序员想要的定义。`const` 成员函数可以修改它处理的对象内部的某些 `bits`，但前提是这些修改对于调用者来说是不可察觉的。文中举了下面这个例子：
  ```cpp
  class CTextBlock {
  public:
      ...
      std::size_t length() const;
  private:
      char* pText;
      std::size_t textLength;    //最近一次计算的文本区块长度。
      bool lengthIsValid;        //目前的长度是否有效。
  };

  std::size_t CTextBlock::length() const
  {
      if (!lengthIsValid) {
          textLength = std::strlen(pText);  //错误！在 const 成员函数内
          lengthIsValid = true;             // 不能赋值给 textLength
      }                                     // 和lengthIsValid。
      return textLength;
  }
  ```
  在这个例子中，`CTextBlock` 类需要缓存文本长度。`length()` 函数在逻辑上是 `const` 的（它不改变文本内容），但它在第一次被调用时需要计算并修改 `textLength` 和 `lengthIsValid` 这两个成员变量。  
  解决方法是通过 C++ 提供的关键字 `mutable`。被 `mutable` 修饰的成员变量，永远不会被视为 `const`，即使在 `const` 成员函数内部，也可以合法地修改它们。这完美地解决了上述的需求。修改后的成员变量：
  ```cpp
  std::size_t textLength; 
  bool lengthIsValid; 
  ```

最后，文章还提到了一个问题，当同时提供了 `const` 和 `non-const` 版本的成员函数（如 `operator[]`）时，它们内部的实现逻辑（如边界检查、日志记录等）往往是完全相同的，为了避免代码的重复，正确的做法是让 `non-const` 版本调用 `const` 版本：
1. 在 `non-const` 函数内部，通过 `static_cast` 将 `*this` 转型为一个 `const` 引用。
2. 调用 `const` 版本的同名函数。
3. `const` 版本会返回一个 `const` 引用/指针。
4. 使用 `const_cast` 移除返回值的 `const` 属性，然后将其返回。

示例：
```cpp
// const 版本 (实现所有逻辑)
const char& operator[](std::size_t pos) const {
    ... // 所有检查
    return text[pos];
}

// non-const 版本 (调用 const 版本)
char& operator[](std::size_t pos) {
    // 1. 将 *this 转型为 const，以调用 const 版本
    // 2. 将 const 版本的 const 返回值，通过 const_cast 移除 const
    return const_cast<char&>(
        static_cast<const TextBlock&>(*this)[pos]
    );
}
```

文章强调，虽然转型（`const_cast`）通常很危险，但在这种特定模式下（`non-const` 调用 `const`）是安全且正确的。

总结：
- `const` 可施加于任何作用域内的对象、函数参数、函数返回值、成员函数本体。将某些东西声明为 `const` 可帮助编译器侦测出错误用法。
- 编译器强制实施 bitwise constness，但编写程序时应该使用 logical constness。（使用 mutable 实现）
当 `const` 和 `non-const` 成员函数有实质等价的实现时，令 `non-const` 版本调用 `const` 版本可避免代码重复。（使用 `static_cast` + `const_cast` 实现）