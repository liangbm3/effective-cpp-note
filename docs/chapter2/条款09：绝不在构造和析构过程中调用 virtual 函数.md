## 条款09：绝不在构造和析构过程中调用 virtual 函数

**Never call virtual functions during construction or destruction**

这个条款的核心是，在类的构造函数和析构函数中，绝对不要调用虚函数。原因在于，在构造和析构过程中，对象的动态类型并不是我们通常所期望的那个类型。即：在基类构造期间，虚函数不是虚函数。

考虑以下代码示例：

```cpp
class Transaction { // Base class
public:
    Transaction() {
        // ...
        logTransaction(); // <--- 试图调用 virtual 函数
    }
    virtual void logTransaction() const = 0; // 纯虚函数
};

class BuyTransaction : public Transaction { // Derived class
public:
    virtual void logTransaction() const override {
        // ... 记录 "Buy" 交易
    }
};

// ...
BuyTransaction b; // 创建一个派生类对象
```

在这个例子中，当你创建 `BuyTransaction b;` 时，C++ 保证 `Base class` 的构造函数必须先于 `Derived class` 的构造函数执行。因此会先进入 `Transaction` 的构造函数。在 `Transaction` 构造函数内部执行时，对象 `b` 的类型被 C++ 视为 `Transaction`，还不是一个 `BuyTransaction`。因为 `BuyTransaction` 特有的成员变量此时尚未被初始化。当调用虚函数时，由于当前对象的类型被视为 `Transaction`，编译器会去调用`Transaction::logTransaction()`。在这个例子中，`Transaction::logTransaction()` 是一个纯虚函数。在构造函数中调用一个纯虚函数会导致程序立即崩溃（或链接失败），这是严重的运行时错误。即使不是纯虚函数，那么被调用的也只会是这个基类版本，而不是程序员所期望的 `BuyTransaction::logTransaction()` 版本。

C++ 之所以这么设计是出于安全的考虑。如果 C++ 允许在 `Transaction` 构造函数中调用 `BuyTransaction::logTransaction()`，`BuyTransaction::logTransaction()` 几乎一定会访问 `BuyTransaction` 类自己声明的成员变量。但此时 `BuyTransaction` 的构造函数还未开始执行，它的成员变量都处于未定义状态。C++ 通过在构造期间禁用虚函数下降，从根本上阻止了这种灾难的发生。

析构函数的情况完全相同，只是顺序相反：
- 析构顺序： `Derived class` 的析构函数先执行，然后 `Base class` 的析构函数后执行。
- 进入 `Transaction` 析构函数： 当基类 `~Transaction()` 开始执行时，`BuyTransaction` 的析构函数已经执行完毕。`BuyTransaction` 的成员变量已被销毁，不再有效。
- 对象的类型： 此时，对象 `b` 的类型被 C++ 视为退化回了 `Transaction`。
- 调用虚函数： 如果 `~Transaction()` 调用了 `logTransaction()`，它只会调用 `Transaction::logTransaction()` 版本，绝不会调用 `BuyTransaction` 的版本（那个版本所依赖的成员数据已经不存在了）。

如果基类构造时确实需要派生类提供的信息，可以让派生类在成员初始化列表中，将这些信息作为参数传递给基类构造函数。
```cpp
// 解决方案：
class Transaction { // Base class
public:
    // 1. 接受派生类传来的信息
    explicit Transaction(const std::string& logInfo) {
        log(logInfo); // log 现在是一个普通的非虚函数
    }
    // ...
private:
    void log(const std::string& message); // 非虚函数
};

class BuyTransaction : public Transaction { // Derived class
public:
    BuyTransaction( /*...参数...*/ )
      : Transaction( createLogString(/*...参数...*/) ) // 2. 将信息向上传递
    {
        // ... BuyTransaction 自己的构造 ...
    }
    // ...
private:
    // 3. 使用一个 private static 辅助函数来创建这些信息
    //    它在对象构造前被调用，是安全的
    static std::string createLogString(/*...参数...*/) {
        return "Creating BuyTransaction...";
    }
};
```

之所以使用 `private static` 辅助函数，是因为在执行成员初始化列表时，`BuyTransaction` 对象本身还未构造完成，其非静态成员变量处于未定义状态。 `static` 成员函数没有 `this` 指针，因此它无法意外访问到那些尚未初始化的非静态成员，这使得它成为在构造阶段准备数据的完美、安全的选择。

总结：
- 绝不在构造函数或析构函数中调用 `virtual` 函数。
    - 因为在这两个阶段，对象的类型被视为当前正在执行构造/析构的那个基类，虚函数调用不会下降到派生类。
    - 如果基类构造时需要派生类的信息，应让派生类通过成员初始化列表将信息传递给基类构造函数。