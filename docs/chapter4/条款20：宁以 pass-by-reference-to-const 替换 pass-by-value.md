## 条款20：宁以 pass-by-reference-to-const 替换 pass-by-value

**Prefer pass-by-reference-to-const to pass-by-value**

这个条款的核心是，在 C++ 中编写函数时，对于用户自定义类型的参数，应该优先使用常量引用传递 (`const T&`)，而不是默认的值传递 (`T`)。

如果使用默认的值传递会在函数调用时，创建一份参数的副本，这个过程是通过调用该对象的拷贝构造函数来完成的。当函数返回时，这个副本会被销毁，又会调用其析构函数。如果这个对象很“昂贵”（内部包含 `std::string`、`std::vector` 或其他复杂数据，或者有复杂的继承体系），拷贝和销毁的开销会非常大。如文中的例子：
```cpp
class Person { ... private: std::string name, address; };
class Student : public Person { ... private: std::string schoolName, schoolAddress; };

// 值传递 (昂贵)
bool validateStudent(Student s);
```

当调用 `validateStudent(plato)` 时，会发生：
- 调用 `Student` 的拷贝构造函数来创建 `s`。
- `Student` 的拷贝构造函数会调用其基类 `Person` 的拷贝构造函数。
- `Student` 的拷贝构造函数还会调用 `schoolName` 和 `schoolAddress` (两个 `string`) 的拷贝构造函数。
- `Person` 的拷贝构造函数还会调用 `name` 和 `address` (另外两个 `string`) 的拷贝构造函数。
- 函数返回时，`s` 被销毁，上述所有对象的析构函数被反向调用。

> 这总共导致了6次拷贝构造和6次析构，仅仅是为了传递一个参数！

解决方法是使用常量引用高效传递：
```cpp
// 常量引用传递 (高效)
bool validateStudent(const Student& s);
```
使用 `const` 的目的是防止函数内部意外修改原始对象。

在面向对象编程中，值传递还会存在一个更严重、更隐蔽的问题，那便是对象切片。当使用值传递来传递一个基类对象时，如果调用者传入了一个派生类对象，那么这个派生类对象的所有派生类特性都会被切割掉，只剩下基类的部分，这会彻底破坏多态，如文中的例子：
```cpp
class Window {
public:
    virtual void display() const; // 基类版本
    std::string name() const;
    ...
};
class WindowWithScrollBars : public Window {
public:
    virtual void display() const override; // 派生类版本 (不同的行为)
    ...
};

// 错误的设计：使用“值传递”
void printNameAndDisplay(Window w) { // w 是一个 Window 对象
    std::cout << w.name();
    w.display(); // ！！！问题在这里
}

WindowWithScrollBars wwsb;
printNameAndDisplay(wwsb); // 传入派生类对象
```

当 `printNameAndDisplay(wwsb)` 被调用时：
- 函数参数 `w` 作为一个新的 `Window` 对象被创建。
- 它通过 `Window` 的拷贝构造函数，从 `wwsb` 中拷贝属于 `Window` 的那部分数据来初始化。
- `wwsb` 中所有属于 `WindowWithScrollBars` 的独特数据（例如滚动条的位置）被“切割”抛弃了。
- 在函数内部，`w` 就是一个 `Window` 类型的对象，它不再是 `WindowWithScrollBars`。
- 因此，`w.display()` 总是调用 `Window::display()`，即使 `display` 是 `virtual` 的，多态失效了。

解决方案是使用引用传递：
```cpp
// 正确的设计：使用“引用传递”
void printNameAndDisplay(const Window& w) { // w 是对原始对象的引用
    std::cout << w.name();
    w.display(); // ！！！现在可以正常工作了
}

WindowWithScrollBars wwsb;
printNameAndDisplay(wwsb); // 传入派生类对象
```

当然，这个规则不适用于所有类型，按值传递适用于以下类型：
- 内置类型: 例如 `int`, `double`, `char`, `bool` 等。
- 指针: 例如 `int*`。
- STL 迭代器。
- STL 函数对象。

总结：
- 尽量以 pass-by-reference-to-const (常量引用传递) 替换 pass-by-value (值传递)。
  - 效率更高：避免了不必要的拷贝构造和析构。
  - 行为正确：避免了在继承体系中的“对象切割”问题。
- 此规则不适用于内置类型、STL迭代器和函数对象。 对于它们，按值传递是适当的。