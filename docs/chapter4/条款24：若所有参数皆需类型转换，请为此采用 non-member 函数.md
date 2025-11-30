## 条款24：若所有参数皆需类型转换，请为此采用 non-member 函数

**Declare non-member functions when type conversions should apply to all parameters.**

这个条款的核心思想是：`member` 函数在处理参数时是不对称的。当一个函数是成员函数时（例如 `operator*`），它有一个隐式的 `this` 参数，以及显式的参数列表。编译器只会对显式参数进行隐式类型转换，而不会对 `this` 指针所指的那个隐式参数进行转换。如果希望一个函数的所有参数都能平等地参与隐式类型转换，那么这个函数必须是一个 non-member 函数。

书中举了一个 `Rational`（有理数）的例子，这个类提供了一个非 `explicit` 的构造函数，允许从 `int` 隐式转换为 `Rational`：
```cpp
class Rational {
public:
    // 允许从 int 隐式转换
    Rational(int numerator = 0, int denominator = 1) 
        : n(numerator), d(denominator) {}
    int numerator() const { return n; }
    int denominator() const { return d; }

private:
    int n, d;
};
```

我们希望支持混合运算，即下面两种乘法都应该能成功：
```cpp
Rational oneHalf(1, 2);
Rational result;

result = oneHalf * 2; // (1) Rational * int
result = 2 * oneHalf; // (2) int * Rational
```

如果我们把 `operator*` 作为 `Rational` 的成员函数来实现：
```cpp
class Rational {
public:
    // ... 构造函数 ...
    
    // 成员函数实现
    const Rational operator*(const Rational& rhs) const {
        return Rational(this->n * rhs.n, this->d * rhs.d);
    }
    // ...
};
```

- 对于 `result = oneHalf * 2;`
  - 编译器会把这个表达式看作 `oneHalf.operator*(2);`
  - `oneHalf` 是调用该函数的对象。`2` 是传入的显式参数。编译器发现 `operator*` 需要一个 `Rational` 类型的参数，但得到的是 `int (2)`。它会查找是否能将 `int` 转换为 `Rational`。编译器自动将 `2` 转换为 `Rational(2)`，然后调用 `oneHalf.operator*(Rational(2))`。代码编译通过。
- 对于 `result = 2 * oneHalf;` 
  - 编译器会把这个表达式看作 `2.operator*(oneHalf);`
  - 编译器试图在 `int (2)` 上调用一个名为 `operator*` 的成员函数。`int` 是一个内置类型，它没有任何成员函数。因此编译器不会尝试把 `2` 转换成 `Rational`，然后再调用它的 `operator*`。代码编译失败。


为了解决这种不对称性，正确的做法是将 `operator*` 移出类外，使其成为一个 `non-member` 函数：
```cpp
const Rational operator*(const Rational& lhs, const Rational& rhs) {
    // 因为 n 和 d 是 private, 我们必须使用 public 访问器
    // (或者将此函数声明为 friend)
    return Rational(lhs.numerator() * rhs.numerator(),
                    lhs.denominator() * rhs.denominator());
}
```

总结：
- 如果需要为某个函数的所有参数（包括那个本应是 `this` 的参数）进行类型转换，那么这个函数必须是一个 `non-member` 函数。