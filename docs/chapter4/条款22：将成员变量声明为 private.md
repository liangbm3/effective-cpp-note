## 条款22：将成员变量声明为 private

**Declare data members private.**

这个条款是 C++ 面向对象编程中**封装**原则的基石。核心论点是：为了保持类的健壮性、灵活性和可维护性，成员变量**必须**声明为 `private`。

首先看为什么不能声明为 `public`，如果将成员变量声明为`public` 意味着我们放弃了对该变量的所有控制权。这会带来四大问题：

+ **封装与灵活性**：封装的意义在于隐藏实现细节，未来我们可以更改实现而无需破坏客户端代码。
+ **细粒度的访问控制**：通过成员函数，可以提供不对称的访问权限

```cpp
class AccessLevels {
public:
    ...
    int getReadOnly() const { return readOnly; }
    void setReadWrite(int value) { readWrite = value; }
    int getReadWrite( ) const { return readWrite; }
    void setWriteOnly(int value) { writeOnly = value; }
private:
    int noAccess;       //对此 int 无任何访问动作
    int readOnly;       //对此 int 做只读访问 (read-only access)
    int readWrite;      //对此 int 做读写访问 (read-write access)
    int writeOnly;      //对此 int 做惟写访问 (write-only access)
};
```

+ **约束与验证：**`setter` 函数可以在设置值之前进行检查。例如，一个 `setMonth(int m)` 函数可以保证 `m` 在 1 到 12 之间。`public` 成员变量可以被随意设置为无效值 (例如 `obj.month = 13;`)，导致对象进入无效状态。
+ **语法一致性**：如果类的所有功能都通过函数提供，用户就不必费神去记是用 `obj.value` 还是 `obj.getValue()`。

对于 `protected`，很多人认为 `protected` 提供了某种程度的封装，但作者指出`protected` 并不比 `public` 更具封装性。

封装的定义是**更改实现而无需破坏客户端代码：**

+ 对于 `public` 成员，客户端是**所有**使用该类的代码。
+ 对于 `protected` 成员，客户端是**所有**继承该类的派生类。

假设有一个 `protected: int x;`，有 100 个派生类直接访问了这个 `x`。如果基类作者修改了 `x`（比如改成 `double` 类型，或者将其移除，用两个 `int` 替代）。**所有 100 个派生类都会编译失败。因此**`public`和`protected` 成员变量都会**破坏封装**，只是破坏的范围不同。

总结：

+ 切记将成员变量声明为 `private`。这可以提供细粒度的访问控制、保证约束条件的能力，以及实现的灵活性。
+ `protected` 并不比 `public` 更具封装性。

