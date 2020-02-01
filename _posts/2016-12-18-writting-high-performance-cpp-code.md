---
layout: post
title: 编写高性能的C++代码
categories: [code, C++]
---
曾经STL受人诟病的一点是，STL容器基于拷贝的方式来工作，任何需要放入容器的元素，都必须满足Copy-Constructible，也就是说把元素存放到总是不可避免要被拷贝一次，造成性能上的浪费；然而事实上C++提供一些减少不必要拷贝的技巧与技术，只要熟练掌握可以写出效率远高于其他语言的代码，本文抛砖引玉介绍几种常用的技巧，如果大家有什么其他补充欢迎在评论指出。

### 善用const引用

虽然这是最基本的常识，但还是经常看到有人（特别是其他语言出身的C++程序员）写出类似于这样的代码：

{% highlight c++ %}
void foo(vector<string> v) {
    // deal something with v
}
{% endhighlight %}

或者类似于这样的：

{% highlight c++ %}
string s = bar(); 
{% endhighlight %}

第一种情况会把整个vector以及里面所有的string元素统统拷贝一遍，而第二种虽然有时候可以RVO和NRVO ([Copy elision](http://en.cppreference.com/w/cpp/language/copy_elision)) ，但在函数体中有分支判断的情况下NRVO经常会失效，所以这两种情况请最好使用`const&`避免不要的拷贝。

### 右值引用与move语义

在C语言中常常会提起lvalue（左值）和rvalue（右值）的概念，通常来说，可以作为赋值运算符=的左边的，就是lvalue，只能出现在=的右边的，就是rvalue。在C++98中，定义一个引用只能引用左值，唯一可以引用右值的情况是const引用函数的返回值（如上面的例子），其余情况引用右值将会导致编译失败。

在C++11中引入了rvalue reference的概念，主要的作用一个是用于move semantics（move语义），另一个是perfect forwarding（完美转发），本文仅介绍和move语言相关的内容。

右值引用在C++11里面的标示是`T&&`，所谓move语义，就是把一个类的资源转移到另一个类；在使用move语义之前，需要先定义move语义的拷贝构造函数，例如：

{% highlight c++ %}
constexpr size_t SZ = 1024;

class HugeElement {
public:
    // constructor
    HugeElement(): _buf(NULL) {
        cout << "calling HugeElement()" << endl;
        _buf = new char[SZ];
    }

    // copy constructor
    HugeElement(const HugeElement& h): _buf(NULL) {
        cout << "calling HugeElement(const HugeElement&)" << endl;
        _buf = new char[SZ];
        memcpy(_buf, h._buf, SZ);
    }

    // move constructor
    HugeElement(HugeElement&& h): _buf(NULL) {
        cout << "calling HugeElement(HugeElement&&)" << endl;
        if (_buf != h._buf) { // in case of "a = std::move(a)"
            delete []_buf;
            _buf = h._buf;
            h._buf = NULL;
        }
    }

    ~HugeElement() {
        cout << "calling ~HugeElement()" << endl;
        delete []_buf;
    }

private:
    char *_buf;
};
{% endhighlight %}

我们定义了一个拥有一段巨大buffer的类，众所周知在高并发的情况下，不断的分配拷贝内存是相当耗CPU的，如果业务逻辑里面需要把这个类存放到容器里面，写一段测试代码，看看`HugeElement`与`vector`的行为是怎么样的：

{% highlight c++ %}
#include <iostream>
#include <vector>
#include <boost/container/vector.hpp>
#include <utility>

using namespace std;

int main() {
#ifndef BOOST
    vector<HugeElement> vec;
#else
    boost::container::vector<HugeElement> vec;
#endif
    vec.reserve(3);
    for (size_t i = 0; i < 4; ++i) {
        vec.push_back(HugeElement());
    }
}
{% endhighlight %}

在MacOS的环境下，使用clang编译上面的程序`clang++ -std=c++11 -g test.cc`输出如下：

```
calling HugeElement()
calling HugeElement(HugeElement&&)    // 三次push_back
calling ~HugeElement()
calling HugeElement()
calling HugeElement(HugeElement&&)
calling ~HugeElement()
calling HugeElement()
calling HugeElement(HugeElement&&)
calling ~HugeElement()
calling HugeElement()                 // 第四次push_back的时候，vector满了
calling HugeElement(HugeElement&&)
calling HugeElement(const HugeElement&)
calling HugeElement(const HugeElement&)
calling HugeElement(const HugeElement&)
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
```

实验结果非常有趣，调用push_back的时候，clang的vector识别HugeElement()返回一个右值引用，因此自动调用了move constructor；然而，当push_back第四个元素的时候vector已经满了，此时clang的实现是先分配新的空间，把第四个元素move到4的位置，再对前三个元素调用copy constructor，导致了三次拷贝，可想而知随着vector不断增大不必要的拷贝仍然不可避免（并且消耗将会越来越大）。

实际上clang的vector并不是一个很好的实现，让我们看看boost的vector是怎么做的，使用`clang++ -std=c++11 -I/usr/local/include -DBOOST test.cc`重新编译并运行，输出如下：

```
calling HugeElement()
calling HugeElement(HugeElement&&)
calling ~HugeElement()
calling HugeElement()
calling HugeElement(HugeElement&&)
calling ~HugeElement()
calling HugeElement()
calling HugeElement(HugeElement&&)
calling ~HugeElement()
calling HugeElement()
calling HugeElement(HugeElement&&)
calling HugeElement(HugeElement&&)
calling HugeElement(HugeElement&&)
calling HugeElement(HugeElement&&)
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
calling ~HugeElement()
```

注意看此时已经完全没有`memcpy`调用了，简直完美！

说到这里，C++11及以后的vector还新加了个`emplace_back`成员函数，通过`完美转发`使得可以直接在vector分配的内存里面直接构造出新的对象，甚至连move constructor都不用调用！详情可以了解：[emplace_back](http://en.cppreference.com/w/cpp/container/vector/emplace_back)。

至于还在苦逼使用C++98的孩子，可以使用[Boost.Move](http://www.boost.org/doc/libs/1_62_0/doc/html/move.html)获得同样的效果；事实上，我在手机QQ调度系统的性能优化项目中采用了boost.move稍微改了几个类的构造函数就使得总体性能提高了大约10%左右！

### “奇技淫巧” Expression Templates

`Expression Templates`是一种通过构造延迟计算使得可以避免不必要的拷贝的模版元编程技术。举个例子，比方说我要把两个同样大小的vector<double>按照元素一个个相加，并返回一个新的vector<double>，那么实现可以如下：

{% highlight c++ %}
#include <vector>
#include <iostream>

class Vec {
    std::vector<double> elems;
public:
    Vec(size_t n) : elems(n) { std::cout << "constructor" << std::endl; }
    double &operator[](size_t i)      { return elems[i]; }
    double operator[](size_t i) const { return elems[i]; }
    size_t size()               const { return elems.size(); }
};

Vec operator+(Vec const &u, Vec const &v) {
    Vec sum(u.size());
    std::cout << "for loop" << std::endl;
    for (size_t i = 0; i < u.size(); i++) {
         sum[i] = u[i] + v[i];
    }
    return sum;
}
{% endhighlight %}

对于`Vector x = a + b`这样的语句来说这样的代码是没有问题的，但是，如果是`Vector x = a + b + c`这样的语句呢？首先`a+b`会产生一个临时变量`t`，接着计算`t+b`赋值给`x`，即使有NRVO，此时也至少会有2次内存分配（分配临时变量和x）和两次for循环。

{% highlight c++ %}
int main()
{
    Vec a(10), b(10), c(10);
    Vec x = a + b + c;
};
{% endhighlight %}

代码输出如下：

```
constructor
constructor
constructor
constructor
for loop
constructor
for loop
```

接下来展示如何使用`Expression Templates`提高性能。首先定义一个`VecExpression`表示任何可以转化为`Vec`的表达式：

{% highlight c++ %}
template <typename E>
class VecExpression {
public:
    double operator[](size_t i) const { return static_cast<E const&>(*this)[i];     }
    size_t size() const { return static_cast<E const&>(*this).size(); }
    E& operator()() { return static_cast<      E&>(*this); }
    const E& operator()() const { return static_cast<const E&>(*this); }
};
{% endhighlight %}

然后，定义我们的`Vec`作为`VecExpression`的子类：

{% highlight c++ %}
class Vec : public VecExpression<Vec> {
    std::vector<double> elems;
public:
    double operator[](size_t i) const { return elems[i]; }
    double &operator[](size_t i)      { return elems[i]; }
    size_t size() const               { return elems.size(); }
    Vec(size_t n) : elems(n) { std::cout << "constructor" << std::endl; }

    template <typename E>
    Vec(VecExpression<E> const& vec) : elems(vec.size()) {
        std::cout << "constructor" << std::endl;
        std::cout << "for loop" << std::endl;
        for (size_t i = 0; i != vec.size(); ++i) {
            elems[i] = vec[i];
        }
    }
};
{% endhighlight %}

最后，我们再定义一个`VecSum`表示两个`Vec`的相加结果：

{% highlight c++ %}
template <typename E1, typename E2>
class VecSum : public VecExpression<VecSum<E1, E2> > {
    E1 const& _u;
    E2 const& _v;
public:
    VecSum(E1 const& u, E2 const& v) : _u(u), _v(v) {}
    double operator[](size_t i) const { return _u[i] + _v[i]; }
    size_t size()               const { return _v.size(); }
};

template <typename E1, typename E2>
VecSum<E1,E2> const
operator+(E1 const& u, E2 const& v) {
   return VecSum<E1, E2>(u, v);
}
{% endhighlight %}

如此这来，`a + b +c`就会被展开为`VecSum<VecSum<Vec, Vec>, Vec>`，从而`elems[i] = v[i];`会被展开为`elems[i] = a.elems[i] + b.elems[i] + c.elems[i];`，于是代码运行如下：

```
constructor
constructor
constructor
constructor
for loop
```

少了一次constructor和一次for loop！关于更多的Expression Templates，这边还有一篇论文使用该技巧优化std::string的连接，众所周知string的operator+会产生大量的临时对象，也是性能杀手之一，感兴趣可以了解一下 [C++03 Expression Templates](http://craighenderson.co.uk/papers/exptempl/)
