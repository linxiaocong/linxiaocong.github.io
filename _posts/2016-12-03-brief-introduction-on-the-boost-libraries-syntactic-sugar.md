---
layout: post
title: Boost简介之语法糖篇
categories: [code, C++]
---

之前在工作中写的一个C++ API因为要交给手机QQ后台非常重要的模块使用，所以把代码交给相关同事Review，过程中同事说你的代码里面用了那么多Boost的东西，有的看起来确实挺不错，可以极大提升编码效率和减少出bug的概率，可以考虑做个分享，同时也方便别人Review代码或者以后维护。今天周末有空在家，于是就有了这篇水文。

StackOverflow上面曾经看到有人说过这样的话：

>If you are coding C++ without using Boost, you are properly wasting your time.

在C++98/03的年代确实是这样子的，然而随着C++11标准的诞生大量新特性被加入到语言本身，使得本文描述的这些Boost特性变得过时，所以本文仅限于还和我一样迫于环境原因用着低版本编译器（比如GCC < 4.8）的C++开发者，或者使用着最新版本编译器而不知道C++11新特性（身在福中不知福）的人。

历史上C++的设计哲学里面有这么一条：

>Prefer libraries to language extensions.（更倾向于使用库而不是扩展语言来实现特性）

而Boost就是诞生在这样的指导方针之下的一个准C++标准库。由于Boost本身非常庞大，本文先介绍一些比较实用的通过库实现的语法糖特性，后续的再慢慢通过其他连载介绍。

### Boost.Assign

其实boost.assign并不是很实用而且有点过于丧心病狂，个人不建议在生产代码里面使用boost.assign以免使得代码过于晦涩，只不过因为它的名字以A开头，所以就先第一个介绍吧。

boost.assign用于方便地对容器赋值，比如说现在有一个 `std::vector<int>`，你需要将它填充1到10总共10个数字，那么可以：

{% highlight c++ %}
#include <vector>
#include <boost/assign.hpp>
#include <boost/assign/std/vector.hpp>

using namespace boost::assign; // 为了使用boost重载的operator+=() 

int main() {
    std::vector<int> v;
    v += 1, 2, 3, 4, 5, 6, 7, 8, 9, 10；
}

{% endhighlight %}

其他容器请参考：[Boost.Assign](http://www.boost.org/doc/libs/1_62_0/libs/assign/doc/index.html)

### Boost.Foreach

相信大多写C++的人都写过类似于这样的代码：

{% highlight c++ %}
std::vector<std::string> v;
std::map< std::string, std::vector<std::string> > m;

for (std::vector<std::string>::const_iterator it = v.begin();
     it != v.end(); ++it) {
    const std::string& s = *it;
    // deal something with s
}

for (std::map< std::string, std::vector<std::string> >::iterator it = m.begin();
     it != m.end(); ++it) {
    const std::string& s = it->first;
    for (std::vector<std::string>::iterator it2 = it->second.begin();
         it2 != it->second.end(); ++it2) {
        std::string &item = *it2;
        // deal something with s, item
    }
}
{% endhighlight %}

其实标准库定义了 `std::for_each`，但在C++11之前你得先定义个 `function object`，使得这个函数其实非常的鸡肋。因此Boost提供了 `BOOST_FOREACH`宏，作为方便遍历容器的解决方案，上面的vector遍历变成了：

{% highlight c++ %}
#include <boost/foreach.hpp>

BOOST_FOREACH(const std::string& s, v) {
    // deal something with s
}
{% endhighlight %}

而对于map的遍历则相对来说麻烦一点，因为map声明中的逗号会使得宏编译失败，因此需要先 `typedef`：

{% highlight c++ %}
typedef std::map< std::string, std::vector<std::string> >::value_type m_value_type;

BOOST_FOREACH(m_value_type& it, m) {
    BOOST_FOREACH(std::string& item, it.second) {
        // deal something with it.first, item
    }
}
{% endhighlight %}

对比一下C++11中最新的`foreach`语法：

{% highlight c++ %}
for (auto &it, m) {
    for (auto &item, it.second) {
        // deal something with it.first, item
    }
}
{% endhighlight %}

差别貌似不是很大 :)

### Boost.Format

格式化字符串这件事情，在C++里面也是一件挺麻烦的事情，一直以来大概有两种方案：
* snprintf
* ostringstream

`snprintf` 的优势是性能好，缺点在于首先你得知道格式化之后大字符串最长有多长，其次，需要记住各种类型有不同的`specifier`，光是整型就有`%d`、`%u`、`%lu`、`%llu`等等好几种类型，不小心用错就会导致各种编译Warning。而`ostringstream`的问题则是把一整条字符串拆分开来，显得不直观。而boost.format可以提供类似于Python那张直观方便的格式化字符串方式，以格式化今天的日期`2016-12-03`为例：

{% highlight c++ %}
#include <stdio.h>
#include <sstream>
#include <iostream>
#include <iomanip>
#include <string>
#include <boost/format.hpp>

using namespace std;

int main() {
    int y = 2016, m = 12, d = 3;

    char str1[256];
    snprintf(str1, 256, "%04d-%02d-%02d", y, m, d);
    cout << str1 << endl;

    ostringstream oss;
    oss << y << '-'
        << std::setfill('0') << std::setw(2) << m << '-'
        << std::setfill('0') << std::setw(2) << d;
    cout << oss.str() << endl;

    // 如果月份和日期不需要指定位数，则可以简化为
    // boost::format("%1%-%2%-%3%") % y % m % d
    cout << boost::format("%1%-%2$02d-%3$02d") % y % m % d << endl;
}
{% endhighlight %}

### Boost.Lambda

在C++98的年代，函数在C++里面并不是一等公民，无法随时随地定义函数。然而类是一等公民，可以通过重载`operator()`使得类的对象看起来像函数，从而使得lambda表达式成为可能，这也是boost.lambda的实现原理。（事实上，C++11的lambda表达式最后也是生成一个匿名类的函数对象）

由于大量使用模版元编程，引入boost.lambda会使代码编译时间显著加长，并且由于语法晦涩，个人并不是很推荐在代码里面大量使用。下面的示例代码，先对v1中的每个字符串元素调用`lexical_cast`转化成整型，插入到v2中，再遍历v2中所有的元素求和（Map => Reduce!）

{% highlight c++ %}
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <boost/assign.hpp>
#include <boost/assign/std/vector.hpp>
#include <boost/lambda/lambda.hpp>
#include <boost/lambda/bind.hpp>
#include <boost/lexical_cast.hpp>

using namespace std;
using namespace boost::assign;
using namespace boost::lambda;

int main() {
    vector<string> v1;
    vector<int>    v2;
    v1 += "1", "2", "3", "4", "5", "6";

    for_each(v1.begin(), v1.end(),
            bind(&vector<int>::push_back, &v2,
                bind(&boost::lexical_cast<int,string>, _1)));

    int sum = 0;
    for_each(v2.begin(), v2.end(), sum += _1);
    cout << sum << endl;
}
{% endhighlight %}

### Boost.LexicalCast

你知道怎么把一个`const char*`的字符串数字转换成整型吗？`atoi`！转换成长整型呢？`atol`！转换成非负整型呢？把数字转换成字符串呢？`snprintf`？...其实我也忘了。

因为所有的字符串到数字、数字到字符串的转换，统统可以用`boost::lexical_cast`解决！

{% highlight c++ %}
#include <iostream>
#include <string>
#include <boost/lexical_cast.hpp>

using namespace std;

int main() {
    cout << (boost::lexical_cast<int>("123") == 123) << endl;
    cout << (boost::lexical_cast<string>(456) == "456") << endl;
}
{% endhighlight %}

### Boost.ScopeExit

ScopeExit应该是今天介绍的最最实用的特性了，即使你使用的是C++11，也强烈推荐了解一下ScopeExit。

很多高级语言比如(Java)经常诟病C++的一点就是需要手动管理资源一个不小心很容易泄露等等。其实个人觉得，只要熟练运用C++的`RAII (Resource Acquisition Is Initialization)`特性，基本上很难发生内存泄漏、文件未关闭等问题。`scoped_ptr`、`scoped_array`、`scoped_lock`这些都是运用`RAII`特性的例子，当boost或者C++标准提供的这些智能指针、锁无法满足你的需求的时候，就可以考虑用ScopeExit了。

ScopeExit用于声明一段代码，当当前的scope退出的时候，自动执行声明的语句，如果你熟悉Golang的话，ScopeExit非常类似于Golang的defer语句。

考虑一下下面的情景：一个函数在多线程的环境中需要访问一个全局的队列，同时打开一个文件进行读写等等操作，对于初学者来说有可能在异常处理的逻辑里面要么忘了关闭文件，要么忘了释放锁，造成严重的后果：

{% highlight c++ %}
pthread_mutex_t mutex;
std::queue<std::string> q;

void foo(const char* filename) {
    pthread_mutex_lock(&mutex);
    FILE *fp = fopen(filename, "a+");

    if (fp == NULL) {
        pthread_mutex_unlock(&mutex);
        return;
    }

    // 访问全局的队列q，写文件等等..

    // oops! 忘了关闭文件了
    if (something_wrong) {
        pthread_mutex_unlock(&mutex);
        return;
    }
}
{% endhighlight %}

传统的解决方案一般有两种：
* 对于每一个return，都仔细检查并加上资源释放语句
* 在函数末端声明一个goto的tag，在此加上资源释放语句，把函数中间所有的return换成goto

相对而言，第二种是比较好的做法，然而通过ScopeExit，我们有更加优雅的解决方案：

{% highlight c++ %}
#include <boost/scope_exit.hpp>

pthread_mutex_t mutex;
std::queue<std::string> q;

void foo(const char* filename) {
    pthread_mutex_lock(&mutex);
    FILE *fp = fopen(filename, "a+");

    BOOST_SCOPE_EXIT(&fp) {
        if (fp)
            fclose(fp);
        pthread_mutex_unlock(&mutex);
    } BOOST_SCOPE_EXIT_END

    if (!fp)
        return;

    // ... 想干嘛干嘛！
}
{% endhighlight %}

关于ScopeExit更多用法，请参考文档：[Boost.ScopeExit](http://www.boost.org/doc/libs/1_62_0/libs/scope_exit/doc/html/scope_exit/tutorial.html)

### Summary

所谓语法糖就是对语言功能没有影响，但是可以让程序更加简洁，有更高的可读性。篇幅有限关于Boost库中的语法糖就先介绍到这里，希望这篇水文可以让你的代码更简洁更快地完成需求 :)
