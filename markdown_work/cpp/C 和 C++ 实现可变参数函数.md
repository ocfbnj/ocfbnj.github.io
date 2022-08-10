# C和C++实现可变参数函数

C语言中的`printf`相关的函数，C++中的`emplace`相关的函数都使用了变参函数。这篇文章将简单介绍C和C++的不同实现方式。

## C语言实现可变参数函数

在C语言中，通过在函数的最后一个参数加省略号（`...`），例如`int printf(const char* format, ...);`，来声明变参函数：

~~~c
int printx(const char* fmt, ...); // 此方法声明的函数
printx("hello world"); // 可能会以一个
printx("a=%d b=%d", a, b); // 或更多参数调用
 
// int printy(..., const char* fmt); // 错误： ... 必须在最后
// int printz(...); // 错误： ... 必须跟随至少一个具名参数
~~~

在函数体内可以且必须通过`<stdarg.h>`中的工具访问这些参数的值。

### 一个简单的例子

下面的`add_nums`函数将参数中的`count`个数相加然后返回它们的和：

~~~c
#include <stdarg.h>
#include <stdio.h>

int add_nums(int count, ...) {
    int result = 0;
    va_list args;
    va_start(args, count);

    for (int i = 0; i < count; ++i) {
        result += va_arg(args, int);
    }

    va_end(args);

    return result;
}

int main(void) {
    printf("%d\n", add_nums(4, 25, 25, 50, 50));
    
    return 0;
}
~~~

输出：

~~~text
150
~~~

### va_list类型

`va_list`维护了宏 `va_start`、 `va_copy`、 `va_arg`及 `va_end`所需的信息。

### va_stat宏

~~~c
#include <stdarg.h>

void va_start(va_list ap, parmN);
~~~

`va_start` 宏使函数能访问`parmN`后的可变参数。

其中`ap`为`va_list`类型实例，`parmN`为首个可变参数前的参数名。

应该在任何对`va_arg`的调用前，以合法的`va_list`对象`ap`调用`va_start` 。

在上面的例子中，参数`count`在可变参数`...`的前面，因此调用`va_start(args, count);`。

### va_copy宏

~~~c
#include <stdarg.h>

void va_copy(va_list dest, va_list src); // C99 起
~~~

`va_copy`宏把`src`复制到`dest`。

在函数返回，或`dest`的任何再初始化（通过调用`va_start`或`va_copy`）前，应该对`dest`调用`va_end`。

下面是一个简单的例子：

~~~c
#include <math.h>
#include <stdarg.h>
#include <stdio.h>

double sample_stddev(int count, ...) {
    double sum = 0;
    va_list args1;
    va_start(args1, count);
    va_list args2;
    va_copy(args2, args1); /* 复制 va_list 对象 */

    /* 以args1计算平均数。 */
    for (int i = 0; i < count; ++i) {
        double num = va_arg(args1, double);
        sum += num;
    }
    va_end(args1);
    double mean = sum / count;

    /* 以args2和平均数计算标准差。 */
    double sum_sq_diff = 0;
    for (int i = 0; i < count; ++i) {
        double num = va_arg(args2, double);
        sum_sq_diff += (num - mean) * (num - mean);
    }
    va_end(args2);

    return sqrt(sum_sq_diff / count);
}

int main(void) {
    printf("%f\n", sample_stddev(4, 25.0, 27.3, 26.9, 25.7));
}
~~~

输出：

~~~text
0.920258
~~~

### va_end宏

~~~c
#include <stdarg.h>

void va_end(va_list ap);
~~~

`va_end` 可以修改`ap`的值，使得它不再能使用。

若无对应的对`va_start`或`va_copy`调用，或在调用`va_start`或`va_copy`的函数返回前没有调用 `va_end` ，则行为未定义。

### va_arg宏

~~~c
#include <stdarg.h>

T va_arg(va_list ap, T);
~~~

`va_arg`宏展开成`T`类型的表达式，表达式对应来自`va_list ap`的下个参数。

调用 `va_arg` 前，必须调用`va_start`或`va_copy`初始化 `ap` ，中间不能有`va_end`调用。每次调用`va_arg`宏都会修改`ap`，令它指向下一个可变参数。

---

## C++实现可变参数函数

在C++11之前，可以通过C语言的方式实现可变参数函数。

在C++11之后，可以通过`std::initializer_list`或参数包来实现。

`std::initializer_list`只能实现参数数目可变，但这些参数必须具有相同的类型（或者它们的类型可以转换为同一个公共类型）。而通过参数包，可以实现参数数目和类型都可变。

### 通过std::initializer_list实现变参函数

`std::initializer_list`是一个类模板，可以通过初始化列表来构造它。

![initializer_list](https://img-blog.csdnimg.cn/2020112212283964.png#pic_center)

#### 一个简单的例子

下面的`add_nums`函数实现和上面C语言的`add_nums`类似的功能：

~~~cpp
#include <initializer_list>
#include <iostream>

template <typename Ty>
Ty add_nums(std::initializer_list<Ty> il) {
    std::cout << "count: " << il.size() << "\n";

    int sum = 0;

    for (auto it = il.begin(); it != il.end(); ++it) {
        sum += *it;
    }

    return sum;
}

int main() {
    std::cout << add_nums({25, 25, 50, 50}) << "\n";
}
~~~

输出：

~~~text
count: 4
150
~~~

#### 类模板std::initializer_list

`std::initializer_list`的实现非常简单，下面是MSVC的实现源码：

~~~cpp
// CLASS TEMPLATE initializer_list
template <class _Elem>
class initializer_list {
public:
    using value_type      = _Elem;
    using reference       = const _Elem&;
    using const_reference = const _Elem&;
    using size_type       = size_t;

    using iterator       = const _Elem*;
    using const_iterator = const _Elem*;

    constexpr initializer_list() noexcept : _First(nullptr), _Last(nullptr) {}

    constexpr initializer_list(const _Elem* _First_arg, const _Elem* _Last_arg) noexcept
        : _First(_First_arg), _Last(_Last_arg) {}

    _NODISCARD constexpr const _Elem* begin() const noexcept {
        return _First;
    }

    _NODISCARD constexpr const _Elem* end() const noexcept {
        return _Last;
    }

    _NODISCARD constexpr size_t size() const noexcept {
        return static_cast<size_t>(_Last - _First);
    }

private:
    const _Elem* _First;
    const _Elem* _Last;
};
~~~

---

### 通过参数包实现变参函数

#### 基础知识

我们可以通过如下语法声明参数包：

~~~cpp
// Args是一个模板参数包; rest是一个函数参数包
// Args表示零个或多个模板类型参数
// rest表示零个或多个函数参数

template <typename Ty, typename... Args>
void func(const Ty& t, const Args&... rest);
~~~

通过如下语法展开参数包：

~~~cpp
func(t, rest...); // 即在参数包后面放一个省略号（...）
~~~

#### 一个简单的例子

下面的`print`函数实现简单的打印功能：

~~~cpp
#include <iostream>

template <typename Ty>
void print(Ty arg) {
    // 处理最后一个参数
    std::cout << arg << "\n";
}

template <typename Ty, typename... Args>
void print(Ty arg, Args... args) {
    std::cout << arg << ", ";
    print(args...);
}

int main() {
    print(0, 3.14, 'C', "C++");
}
~~~

输出：

~~~text
0, 3.14, C, C++
~~~

上面的例子是一种常见的编写方式，将多个参数分解为一个和一包，第一个`print`处理最后一个参数，第二个`print`处理一个和一包参数。

#### 使用折叠表达式简化操作

在C++17之后引入了折叠表达式语法，通过它可以简化参数包的操作：

~~~cpp
#include <iostream>

template <typename... Args>
void print(Args... args) {
    (std::cout << ... << args) << "\n";
}

template <typename... Args>
auto sum(Args... args) {
    return (... + args);
}

int main() {
    print(0, 3.14, 'C', "C++");
    std::cout << sum(1, 2, 3, 4, 5) << "\n";
}
~~~

输出：

~~~text
03.14CC++
15
~~~

### 关于emplace相关的函数

在C++11之后，标准库容器增加了一些`emplace`相关的函数，它们可以在未初始化的内存上原位构造对象，从而避免拷贝对象。其中有一个完美转发的问题，这不是这篇文章的重点，因此这篇文章不会对它进行说明。

## 参考

- 《C++ Primer》
- [cppreference.com](https://cppreference.com)
