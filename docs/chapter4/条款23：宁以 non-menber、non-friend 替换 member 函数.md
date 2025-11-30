## 条款23：宁以 non-member、non-friend 替换 member 函数

**Prefer non-member, non-friend functions to member functions.**

这个条款的核心是：如果一个函数可以仅通过 `class` 的 `public` 接口完成它的任务，那么它就不应该成为该 `class` 的成员函数或友元函数，而是应该被实现为一个普通的非成员函数。

书中作者举了一个 `webBrowser` 的例子，`WebBrowser` 类已经提供了几个 `public` 成员函数：

```cpp
namespace WebBrowserStuff {
    class WebBrowser {
    public:
        void clearCache();
        void clearHistory();
        void removeCookies();
        // ... 其他核心功能 ...
    };
}
```

现在，我们想提供一个便利函数 `clearEverything`，它能一次性调用上述所有清理函数。我们有两种选择，第一种是作为成员函数：
```cpp
namespace WebBrowserStuff {
    class WebBrowser {
    public:
        void clearCache();
        void clearHistory();
        void removeCookies();

        // 选项 A: 作为成员函数
        void clearEverything() {
            clearCache();
            clearHistory();
            removeCookies();
        }
        // ...
    };
}
```

第二种是作为非成员、非友元函数：

```cpp
// 在 "webbrowser_utils.h" (或别的头文件) 中
#include "webbrowser.h"

namespace WebBrowserStuff {
    // 选项 B: 作为非成员函数
    inline void clearEverything(WebBrowser& wb) {
        wb.clearCache();
        wb.clearHistory();
        wb.removeCookies();
    }
}
```

这个条款明确指出，第二种选择是更好的设计，作者给出了如下原因：

+ 增加封装性：封装的定义是隐藏实现细节。我们隐藏的越多，未来修改实现时，需要更新的代码就越少。`clearEverything` 如果作为成员函数，它拥有访问 `WebBrowser` 所有 `private` 和 `protected` 成员的全部权限。`clearEverything` 如果作为一个非成员、非友元函数，它对 `WebBrowser` 的 `private` 成员一无所知。它只能（也只需要）调用 `public` 接口。因此非成员函数对 `WebBrowser` 的内部实现耦合度更低。如果未来我们重构 `WebBrowser` 的 `private` 成员（例如改变 `clearCache` 的实现，但保持其 `public` 签名不变）非成员、非友元函数完全不受影响。而成员函数由于可以看到 `private` 细节，可能会（被有意或无意地）依赖这些细节，从而降低了封装性。简单的说：一个 `non-member`函数无法访问 `private` 成员，因此它根本不会去依赖这些成员，这反而增强了类的封装。
+ 包裹弹性：成员函数必须被添加到 `class` 的定义中。类定义必须是连续的，不能在多个头文件中扩展一个类定义。非成员函数可以被放在不同的头文件中，只要它们都位于同一个 `namespace` 下。这种设计和 C++ 标准库非常类型，允许将功能模块化，例如：
- `webbrowser.h`：只包含 `WebBrowser` 的核心类定义。
- `webbrowser_bookmarks.h`：包含与书签相关的非成员便利函数。
- `webbrowser_cookies.h`：包含与 `cookie` 相关的非成员便利函数。
- `webbrowser_printing.h`：包含与打印相关的非成员便利函数。
  客户端可以只 `#include` 他们需要的功能，这减少了编译依赖。如果 `WebBrowser` 的所有功能都是成员函数，用户就必须 `include` 包含所有功能声明的那个庞大的头文件。
- 机能扩充性：`class` 的定义是封闭的，客户端无法向 `class` 中添加新的成员函数。但是，任何人都可以在 `namespace` 中添加新的非成员函数来扩充功能。

总结：
- 宁可拿 `non-member`、`non-friend` 函数替换 `member` 函数。这样做可以增加封装性、包裹弹性和机能扩充性。