## 条款32：确定你的 public 继承塑模出 is-a 关系

**Make sure public inheritance models "is-a"**

这是 C++ 面向对象编程中非常重要的一条规则，它强调 C++ 中的 `public` 继承，其唯一的含义就是 "is-a" 关系。当我们写下 `class Derived : public Base { ... };` 时，即是在向编译器和所有阅读代码的人做出一个承诺：每一个 `Derived` 类型的对象同时也是一个 `Base` 类型的对象。

这个承诺意味着，所有适用于 `Base` 对象的行为和属性，也必须同样适用于 `Derived` 对象。这通常被称为Liskov 替换原则 (Liskov Substitution Principle, LSP)。即如果程序中任何地方需要一个 `Base` 对象，都可以用一个 `Derived` 对象去替换它，而程序的行为不应该有任何“意外”的改变。

示例：
```cpp
class Person { ... };
class Student : public Person { ... }; // 学生 "is-a" 人

void eat(const Person& p); // 任何人都会吃饭
void study(const Student& s); // 只有学生才到校学习

Person p;
Student s;

eat(p); // 正确
eat(s); // 正确，因为学生也是人
study(s); // 正确
// study(p); // 错误，并非所有人都是学生
```

文中作者还举了两个经典违法 LSP 的例子。

第一个例子如下：
```cpp
class Bird {
public:
    virtual void fly();
};

class Penguin : public Bird { ... }; // 糟糕的设计！
```

从分类学的角度，企鹅是一种鸟，所以 `class Penguin : public Bird { ... }` 看起来很正常。但是如果为 `Bird` 定义一个 `fly()`方法，这个继承关系承诺了企鹅会飞，但我们都知道这是错误的，违反了 "is-a" 原则。对于这种错误的设计，文中提到了两种修复方法：
- 在 `Penguin` 中重新定义 `fly()` 并让它报错。
  ```cpp
  virtual void fly() {
      throw "企鹅不会飞！"; // 这是一种糟糕的运行时错误
  }
  ```
  但这是一个坏主意，因为一个设计层面的逻辑错误，推迟到了运行时才暴露出来。好的设计应该尽可能在编译期就防止错误。
- 更合理的继承体系，因为鸟这个概念本身不应该和会飞绑定。
  ```cpp
  class Bird {
      // ... 关于鸟的通用属性，比如有羽毛、是卵生等
      // 没有 fly() 函数
  };

  class FlyingBird : public Bird {
  public:
      virtual void fly();
  };

  class Penguin : public Bird {
      // ... 企鹅特有的属性
  };
  ```
  这个设计更加精确。`Penguin` is-a `Bird`，`FlyingBird` is-a `Bird`。这个模型是正确的，因为它没有做出任何错误的承诺。

第二个例子是正方形与矩形的陷阱。从数学直觉上，正方形是一种特殊的矩形。所以 `class Square : public Rectangle` 似乎完全没有问题。但如果从行为上去分析，假设有一个 `Rectangle` 类：
```cpp
class Rectangle {
public:
    virtual void setWidth(int newWidth);
    virtual void setHeight(int newHeight);
    int width() const;
    int height() const;
    ...
};
```

`Rectangle` 的一个核心行为是：它的长和宽可以被独立修改。假设有一个函数依赖于这个行为：
```cpp
void makeBigger(Rectangle& r) {
    int oldHeight = r.height();
    r.setWidth(r.width() + 10);
    // 我们断言：修改宽度不应该影响高度
    assert(r.height() == oldHeight);
}
```

如果我们让 `Square` 公开继承 `Rectangle`：
```cpp
class Square : public Rectangle { ... };
```
为了维持“正方形”的特性（长宽必须相等），`Square` 类的 `setWidth` 和 `setHeight` 必须被特殊实现，比如：
```cpp
void Square::setWidth(int newWidth) {
    // 为了保持正方形特性，必须同时修改长和宽
    Rectangle::setWidth(newWidth);
    Rectangle::setHeight(newWidth);
}
```

当我们把一个 `Square` 对象传给 `makeBigger` 函数时：
```cpp
Square s;
s.setWidth(10); // 此时 s 的长和宽都是10
makeBigger(s);  // s "is-a" Rectangle，所以编译通过
```
在 `makeBigger` 内部，`oldHeight` 被记录为 10。`s.setWidth(20)` 被调用。根据 `Square` 的实现，这会把 `s` 的宽和高都设置成 20。`assert(s.height() == oldHeight)` 会被触发，因为 `s.height()` 现在是 20，而 `oldHeight` 是 10。因此这种继承关系是错误的。

<Aside type="note">
总结：`public` 继承的意义就是 "is-a"。"is-a" 关系的核心是行为上的一致性，也就是 Liskov 替换原则。所以在使用 `public` 继承前，需要思考派生类的对象，是否能真正地、毫无意外地履行基类的所有行为契约。
</Aside>