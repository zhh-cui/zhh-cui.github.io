---
layout: default
title: 增强的异常处理方法（enhancedExceptionHandlingMethod）
---
### 需求描述 ###
在利用OpenCV（2.x）进行图像处理算法开发过程中，总会遇到程序崩溃的情况。通常情况下，崩溃发生在程序运行时，OpenCV会给出一个“cv::Exception”，对错误进行简单描述。虽然异常产生于OpenCV库的内部，但是*通常是由用户错误调用OpenCV函数引起的*，该描述仅限于对异常本身的一点简单描述，没有对异常产生的过程进行清晰的梳理（这需要查看堆栈中函数调用过程），更不能给出异常发生时一些关键变量的值。当程序规模变得复杂一点，当程序部署到生产环境中，程序崩溃了，但用户无法获知有关于程序在崩溃前一时刻的状态的更多信息，无疑对于查找Bug带来困扰，尤其是一些偶发异常，想要重现Bug都很困难。

本文试图找出一个可行的解决方案。以下代码在OpenCV 2.4.4和g++ 4.6.3编译通过。

### 解决思路 ###
通过学习C/C++、OpenCV（2.x）中对于错误和异常的处理方法，参见[《程序中的错误处理方法》][1]，很容易想到这样一种解决方案：

像SIGFPE、SIGSEGV这样的信号，在算法开发过程中是比较常见的，就以此为例进行说明。

- 程序在系统中运行，系统很容易捕获SIGFPE、SIGSEGV、SIGINT信号

###### 这里需要强调的是，SIGFPE信号针对的是”整数除0“这一现象，”浮点数除0“不触发该信号，运算结果是inf。“浮点数除0”需要在\<fenv.h\>/\<cfenv\>中检测，属于C++ 11的范畴，在此不做讨论。 ######

- 在程序中注册信号处理函数，利用用户定义的函数处理异常，替代系统默认处理函数

- 用户定义的异常处理函数通过某种方式通知程序，有错误发生了

- 程序获知错误信息，决定下一步行为

- 程序退出时注销用户定义的异常处理函数

### 失败尝试一 ###

{% highlight c++ linenos %}
#include <iostream>
#include <csignal>
#include <cstdio>

sig_atomic_t signaled = 0;

void new_sigint_handler(int param) {
    std::cout << "In new_sigint_handler" << std::endl;
    signaled = 1;
}

int main(int argc, char **argv) {
    std::cout << "Set user handler for SIGINT signal" << std::endl;
    __sighandler_t old_sigint_handler;
    old_sigint_handler = signal(SIGINT, new_sigint_handler);
    if (SIG_ERR == old_sigint_handler) {
        perror("Error on set SIGINT handler");
        return -1;
    } else if (SIG_DFL == old_sigint_handler) {
        std::cout << "Found old SIGINT handler: default handling" << std::endl;
    } else if (SIG_IGN == old_sigint_handler) {
        std::cout << "Found old SIGINT handler: ignore signal" << std::endl;
    }
    std::cout << "Set successfully" << std::endl;

    while (!signaled) {
    }

    std::cout << "Set old handler for SIGINT signal" << std::endl;
    __sighandler_t reset_sigint_handler;
    reset_sigint_handler = signal(SIGINT, old_sigint_handler);
    if (SIG_ERR == reset_sigint_handler) {
        perror("Error on reset SIGINT handler");
        return -2;
    }
    std::cout << "Set successfully" << std::endl;
    return 0;
}
{% endhighlight %}

用户定义的“new_sigint_handler\(\)”函数通过修改全局变量“signaled”的值来通知主程序是否捕获到SIGINT信号。

首先通过“signal\(\)”函数注册用户定义的SIGINT信号处理函数，并且检查注册是否成功，同时提示出原有的SIGINT信号处理函数是什么。然后进入功能部分，即死循环等待SIGINT信号。用户按下ctrl-c组合键，触发SIGINT信号，被用户定义的处理函数捕获。用户定义的处理函数修改signaled值，被主程序检测到，于是跳出死循环。最后恢复原有的信号处理函数。

运行结果：

{% highlight text linenos %}
Set user handler for SIGINT signal
Found old SIGINT handler: default handling
Set successfully
^CIn new_sigint_handler
Set old handler for SIGINT signal
Set successfully
{% endhighlight %}

###### 这种处理方法貌似还不错，但是没有利用C++的异常处理机制，主程序流程不够灵活，显得很笨拙。*尤其是，对于SIGFPE、SIGSEGV信号，程序行为出现了异常*。 ######

{% highlight c++ linenos %}
#include <iostream>
#include <csignal>
#include <cstdio>

sig_atomic_t signaled = 0;

void new_sigfpe_handler(int param) {
    std::cout << "In new_sigfpe_handler" << std::endl;
    signaled = 1;
    sleep(1);
}

int main(int argc, char **argv) {
    std::cout << "Set user handler for SIGFPE signal" << std::endl;
    __sighandler_t old_sigfpe_handler;
    old_sigfpe_handler = signal(SIGFPE, new_sigfpe_handler);
    if (SIG_ERR == old_sigfpe_handler) {
        perror("Error on set SIGFPE handler");
        return -1;
    } else if (SIG_DFL == old_sigfpe_handler) {
        std::cout << "Found old SIGFPE handler: default handling" << std::endl;
    } else if (SIG_IGN == old_sigfpe_handler) {
        std::cout << "Found old SIGFPE handler: ignore signal" << std::endl;
    }
    std::cout << "Set successfully" << std::endl;

    float a = 0.0;
    float b = 10 / a;
    std::cout << "b = " << b << std::endl;
    int c = 0;
    int d = 10 / c;
    while (!signaled) {
    }

    std::cout << "Set old handler for SIGFPE signal" << std::endl;
    __sighandler_t reset_sigfpe_handler;
    reset_sigfpe_handler = signal(SIGFPE, old_sigfpe_handler);
    if (SIG_ERR == reset_sigfpe_handler) {
        perror("Error on reset SIGFPE handler");
        return -2;
    }
    std::cout << "Set successfully" << std::endl;
    return 0;
}
{% endhighlight %}

运行结果：

{% highlight text linenos %}
Set user handler for SIGFPE signal
Found old SIGFPE handler: default handling
Set successfully
b = inf
In new_sigfpe_handler
In new_sigfpe_handler
In new_sigfpe_handler
^C
{% endhighlight %}

在“new_sigfpe_handler\(\)”中，增加“sleep\(1\);”的目的是为了让程序慢下来，因为：

该函数被无限调用了！！直到用户按下ctrl-c组合键终止程序。

也就是说，系统不停的检测到SIGFPE信号，于是不停的调用用户定义的异常处理函数，主程序无法进行后面的判断。

SIGSEGV信号也存在类似的情况。

{% highlight c++ linenos %}
#include <iostream>
#include <csignal>
#include <cstdio>

sig_atomic_t signaled = 0;

void new_sigsegv_handler(int param) {
    std::cout << "In new_sigsegv_handler" << std::endl;
    signaled = 1;
    sleep(1);
}

int main(int argc, char **argv) {
    std::cout << "Set user handler for SIGSEGV signal" << std::endl;
    __sighandler_t old_sigsegv_handler;
    old_sigsegv_handler = signal(SIGSEGV, new_sigsegv_handler);
    if (SIG_ERR == old_sigsegv_handler) {
        perror("Error on set SIGSEGV handler");
        return -1;
    } else if (SIG_DFL == old_sigsegv_handler) {
        std::cout << "Found old SIGSEGV handler: default handling" << std::endl;
    } else if (SIG_IGN == old_sigsegv_handler) {
        std::cout << "Found old SIGSEGV handler: ignore signal" << std::endl;
    }
    std::cout << "Set successfully" << std::endl;

    int a[10];
    for (int i = 0; i < 65535; ++i) {
        a[i] = i;
    }
    std::cout << "Check" << std::endl;
    for (int i = 50000; i < 50005; ++i) {
        std::cout << "a[" << i << "] = " << i << std::endl;
    }
    while (!signaled) {
    }

    std::cout << "Set old handler for SIGSEGV signal" << std::endl;
    __sighandler_t reset_sigsegv_handler;
    reset_sigsegv_handler = signal(SIGSEGV, old_sigsegv_handler);
    if (SIG_ERR == reset_sigsegv_handler) {
        perror("Error on reset SIGSEGV handler");
        return -2;
    }
    std::cout << "Set successfully" << std::endl;
    return 0;
}
{% endhighlight %}

运行结果：

{% highlight text linenos %}
Set user handler for SIGSEGV signal
Found old SIGSEGV handler: default handling
Set successfully
In new_sigsegv_handler
In new_sigsegv_handler
In new_sigsegv_handler
^C
{% endhighlight %}

通过查询signal的man手册，看到如下文字：

> According to POSIX, the behavior of a process is undefined after it ignores a SIGFPE, SIGILL, or SIGSEGV signal that was not generated by kill(2) or raise(3). Integer division by zero has undefined result. On some architectures it will generate a SIGFPE signal. (Also dividing the most negative integer by -1 may generate SIGFPE.) Ignoring this signal might lead to an endless loop.

### 失败尝试二 ###
将全局变量修改成抛出异常，期待主程序能够捕获。

{% highlight c++ linenos %}
#include <iostream>
#include <csignal>
#include <cstdio>
#include <exception>

void new_sigint_handler(int param) {
    std::cout << "In new_sigint_handler" << std::endl;
    throw(std::exception());
}

int main(int argc, char **argv) {
    std::cout << "Set user handler for SIGINT signal" << std::endl;
    __sighandler_t old_sigint_handler;
    old_sigint_handler = signal(SIGINT, new_sigint_handler);
    if (SIG_ERR == old_sigint_handler) {
        perror("Error on set SIGINT handler");
        return -1;
    } else if (SIG_DFL == old_sigint_handler) {
        std::cout << "Found old SIGINT handler: default handling" << std::endl;
    } else if (SIG_IGN == old_sigint_handler) {
        std::cout << "Found old SIGINT handler: ignore signal" << std::endl;
    }
    std::cout << "Set successfully" << std::endl;

    try {
        while (true) {
        }
    } catch (std::exception ex) {
        std::cout << "Catch: " << ex.what() << std::endl;
    }

    std::cout << "Set old handler for SIGINT signal" << std::endl;
    __sighandler_t reset_sigint_handler;
    reset_sigint_handler = signal(SIGINT, old_sigint_handler);
    if (SIG_ERR == reset_sigint_handler) {
        perror("Error on reset SIGINT handler");
        return -2;
    }
    std::cout << "Set successfully" << std::endl;
    return 0;
}
{% endhighlight %}

运行结果：

{% highlight text linenos %}
Set user handler for SIGINT signal
Found old SIGINT handler: default handling
Set successfully
^CIn new_sigint_handler
terminate called after throwing an instance of 'std::exception'
  what():  std::exception
已放弃 (核心已转储)
{% endhighlight %}

看来还是不行，主程序不能catch到异常。

### 成功版本 ###

经过尝试，“signal\(\)”函数可以成功注册用户自定义的信号处理函数，但是信号处理函数结束后的程序行为需要更明显、更清晰、更强有力的控制，主程序才能正常执行。

C/C++标准库中有这样的头文件\<setjmp.h\>/\<csetjmp\>，可以赋予用户这样的控制能力。

下面代码同时注册了三个信号处理函数，主程序中先等待ctrl-c组合键触发SIGINT信号，然后依次触发SIGFPE信号和SIGSEGV信号。

{% highlight c++ linenos %}
#include <iostream>
#include <csignal>
#include <cstdio>
#include <csetjmp>
#include <exception>

jmp_buf int_env;
jmp_buf fpe_env;
jmp_buf segv_env;

void new_sigint_handler(int param) {
    std::cout << "In new_sigint_handler" << std::endl;
    longjmp(int_env, SIGINT);
}

void new_sigfpe_handler(int param) {
    std::cout << "In new_sigfpe_handler" << std::endl;
    longjmp(fpe_env, SIGFPE);
}

void new_sigsegv_handler(int param) {
    std::cout << "In new_sigsegv_handler" << std::endl;
    longjmp(segv_env, SIGSEGV);
}

int main(int argc, char **argv) {
    std::cout << "Set user handler for SIGINT signal" << std::endl;
    __sighandler_t old_sigint_handler;
    old_sigint_handler = signal(SIGINT, new_sigint_handler);
    if (SIG_ERR == old_sigint_handler) {
        perror("Error on set SIGINT handler");
        return -1;
    } else if (SIG_DFL == old_sigint_handler) {
        std::cout << "Found old SIGINT handler: default handling" << std::endl;
    } else if (SIG_IGN == old_sigint_handler) {
        std::cout << "Found old SIGINT handler: ignore signal" << std::endl;
    }
    std::cout << "Set successfully" << std::endl;

    std::cout << "Set user handler for SIGFPE signal" << std::endl;
    __sighandler_t old_sigfpe_handler;
    old_sigfpe_handler = signal(SIGFPE, new_sigfpe_handler);
    if (SIG_ERR == old_sigfpe_handler) {
        perror("Error on set SIGFPE handler");
        return -2;
    } else if (SIG_DFL == old_sigfpe_handler) {
        std::cout << "Found old SIGFPE handler: default handling" << std::endl;
    } else if (SIG_IGN == old_sigfpe_handler) {
        std::cout << "Found old SIGFPE handler: ignore signal" << std::endl;
    }
    std::cout << "Set successfully" << std::endl;

    std::cout << "Set user handler for SIGSEGV signal" << std::endl;
    __sighandler_t old_sigsegv_handler;
    old_sigsegv_handler = signal(SIGSEGV, new_sigsegv_handler);
    if (SIG_ERR == old_sigsegv_handler) {
        perror("Error on set SIGSEGV handler");
        return -3;
    } else if (SIG_DFL == old_sigsegv_handler) {
        std::cout << "Found old SIGSEGV handler: default handling" << std::endl;
    } else if (SIG_IGN == old_sigsegv_handler) {
        std::cout << "Found old SIGSEGV handler: ignore signal" << std::endl;
    }
    std::cout << "Set successfully" << std::endl;

    try {
        int status = setjmp(int_env);
        if (status) {
            throw std::exception();
        }
        while (true) {
        }
    } catch (std::exception ex) {
        std::cout << "Catch SIGINT: " << ex.what() << std::endl;
    }

    try {
        int status = setjmp(fpe_env);
        if (status) {
            throw std::exception();
        }
        int a = 0;
        int b = 10 / a;
        while (true) {
        }
    } catch (std::exception ex) {
        std::cout << "Catch SIGFPE: " << ex.what() << std::endl;
    }

    try {
        int status = setjmp(segv_env);
        if (status) {
            throw std::exception();
        }
        int *a = NULL;
        int b = 10 * (*a);
        while (true) {
        }
    } catch (std::exception ex) {
        std::cout << "Catch SIGSEGV: " << ex.what() << std::endl;
    }

    std::cout << "Set old handler for SIGINT signal" << std::endl;
    __sighandler_t reset_sigint_handler;
    reset_sigint_handler = signal(SIGINT, old_sigint_handler);
    if (SIG_ERR == reset_sigint_handler) {
        perror("Error on reset SIGINT handler");
        return -4;
    }
    std::cout << "Set successfully" << std::endl;

    std::cout << "Set old handler for SIGFPE signal" << std::endl;
    __sighandler_t reset_sigfpe_handler;
    reset_sigfpe_handler = signal(SIGFPE, old_sigfpe_handler);
    if (SIG_ERR == reset_sigfpe_handler) {
        perror("Error on reset SIGFPE handler");
        return -5;
    }
    std::cout << "Set successfully" << std::endl;

    std::cout << "Set old handler for SIGSEGV signal" << std::endl;
    __sighandler_t reset_sigsegv_handler;
    reset_sigsegv_handler = signal(SIGSEGV, old_sigsegv_handler);
    if (SIG_ERR == reset_sigsegv_handler) {
        perror("Error on reset SIGSEGV handler");
        return -6;
    }
    std::cout << "Set successfully" << std::endl;

    return 0;
}
{% endhighlight %}

运行结果：

{% highlight text linenos %}
Set user handler for SIGINT signal
Found old SIGINT handler: default handling
Set successfully
Set user handler for SIGFPE signal
Found old SIGFPE handler: default handling
Set successfully
Set user handler for SIGSEGV signal
Found old SIGSEGV handler: default handling
Set successfully
^CIn new_sigint_handler
Catch SIGINT: std::exception
In new_sigfpe_handler
Catch SIGFPE: std::exception
In new_sigsegv_handler
Catch SIGSEGV: std::exception
Set old handler for SIGINT signal
Set successfully
Set old handler for SIGFPE signal
Set successfully
Set old handler for SIGSEGV signal
Set successfully
{% endhighlight %}

查询程序退出状态，可以看到程序是正常退出的：

{% highlight bash linenos %}
$ echo $?
0
{% endhighlight %}

我已经将该功能写成一个库“enhanced exception handling method”，缩写为“eEHM”，还增加了堆栈的解析功能，可以清晰的看到函数的调用过程。下载地址。

示例代码：



运行结果：





[1]: ../05/程序中的错误处理方法.html "程序中的错误处理方法"
[2]: https://github.com/zhh-cui/eEHM "eEHM"


