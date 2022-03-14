+++
title = "使用c++的新特性来替代宏"
date = 2022-02-21T20:38:05+08:00
draft = false
archives = 2022
categories = "c++"
tags = ["c++"]
+++

要说c++或者c中最容易滥用的特性，莫不过于宏。

事实上，宏有很多缺点，比如难以调试，容易造成污染。（比如Windows.h里就定义了max和min宏，毫无避讳。如果调用xxx.max()之类的函数，那么恭喜你的max被替换了）

所幸，C++为我们提供了一套机制来减少宏的使用。

<!--more-->

### 使用std::source_location来替代__LINE__和__FILE__等
不仅如此，std::source_location还能显示列号等宏并不实现的功能。

std::source_location的用法如:
```c++
#include <source_location>
#include <iostream>

void test(const std::source_location& location = std::source_location::current()){
    std::cout << "at " << local.file_name() << ":" <<local.line() << std::endl;
}

int main(){
    test()
}
```
至于std::source_location是怎么实现获取源代码位置的呢，可以看看实现。

现在看看这个`std::source_location::current()`的实现，其实并不高深:
```c++
_NODISCARD static consteval source_location current(const uint_least32_t _Line_ = __builtin_LINE(),
        const uint_least32_t _Column_ = __builtin_COLUMN(), const char* const _File_ = __builtin_FILE(),
        const char* const _Function_ = __builtin_FUNCTION()) noexcept {
        source_location _Result;
        _Result._Line     = _Line_;
        _Result._Column   = _Column_;
        _Result._File     = _File_;
        _Result._Function = _Function_;
        return _Result;
    }
```
小小的提示：`__builtin_XXXX`可以获取调用此函数的源代码处某些信息，内置于编译器中。

五秒钟的时间来思考他是怎么工作的。


---------------


时间到！

答案就是：C++会在调用函数之前构造参数，并在调用处构造，即使是默认参数也是如此。编译器会自动在函数的的调用处调用`std::source_location::current()`来获取源代码参数位置。

这样，我们就把代码中的`__FILE__`和`__LINE__`进行了替换。很轻松，不是么？


### 使用模板来替换一部分宏
要说把宏完全替换，这自然是不大可能的，但是用模板，就能替换绝大部分代码生成宏。

比如，笔者曾写过一段用于定义一个异常类的宏:
```c++
#define MAKE_EXCEPTION(Name)                            \
    class Name ## Excpetion : public Exception{    \
        /* more define ... */                           \
    }
```
仔细一看，这这种代码生成视乎是模板做不到的，但事实并非如此，而且可以做的更好:
```c++
/// @brief 字符串字面值
    template<size_t N>
    struct StringLiteral {
        constexpr StringLiteral(const char (&str)[N]) {
            std::copy_n(str, N, value);
        }

        char value[N];
    };

// 我们要生成异常的主要模板
template<utopia::core::StringLiteral exception_name>
    class UniversalException : public Exception {
      public:
        inline virtual std::string get_name() const noexcept override {
            return std::string{ exception_name.value };
        }
    };

// 用法如
using IOException = UniversalException<"IOException">;
// throw IOException{"I'm a exception."};

// 我们就定义出了一个新异常！
// std::is_same_v<IOException,UniversalException<"IOException">> == true
// std::is_same_v<IOException,UniversalException<"OtherException">> == false

// 并且我们支持比宏更多的用法!
// --------------------------------↓ 通过继承来使用
class FileSystemException : public UniversalException<"IOException">{
    // ...
}
```

### 使用inline等替换宏
宏最饱受诟病之处就在于，宏难以调试，难以知道复杂宏的展开结果。

但是使用inline就可以等价进行替代。并且inline的控制比宏更加灵活，可以做到：Debug时不展开函数，最大化地方便Debug。Release时完全展开，和宏的性能无异。

可见:
```c++
// 如MD5算法的实现
//F,G,H,I四个非线性变换函数
#define F(x, y, z) (((x) & (y)) | ((~x) & (z)))
#define G(x, y, z) (((x) & (z)) | ((y) & (~z)))
#define H(x, y, z) ((x) ^ (y) ^ (z))
#define I(x, y, z) ((y) ^ ((x) | (~z)))


// 使用inline和模板可以定义作:
namespace MD5{

    template<typename T>
    inline T F(T x,T y,T z){
        return (((x) & (y)) | ((~x) & (z)));
    }
    template<typename T>
    inline T G(T x,T y,T z){
        return (((x) & (z)) | ((y) & (~z)));
    }
    template<typename T>
    inline T H(T x,T y,T z){
        return ((x) ^ (y) ^ (z));
    }
    template<typename T>
    inline T I(T x,T y,T z){
        return ((y) ^ ((x) | (~z)));
    }

}
// 用法上几乎无异，但调试难度降低了一个级别
// 并且我们成功防止了命名空间污染
// 想一想全世界的程序员都把宏的名字定义成 F G H I 或者类似的名字会怎么样？
// Windows API提供的windows.h已经把max min定义成了宏，引起了许多问题
```

### 防止宏污染
c++20添加了module机制。

视乎并没有什么值得注意的，但有一个小小的highlight:通过import关键字导入一个模块不会引入这个模块定义的宏。

听起来不错，是吧？

但截至目前，各大构建系统对c++20 module的支持仍然不乐观。

只能依靠程序员多做一点，来让这个世界变得更美好。

DONE.
