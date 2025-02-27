---
layout:     post
title:      "系统信号处理"
subtitle:   "signal vs sigaction"
date:       2022-03-08 16:37:00
author:     "Ethan"
header-img: "img/home-bg.jpg"
catalog: false
tags:
    - 编程基础
---


# 区别
signal函数和sigaction函数是用了处理系统信号（比如SIGINT,SIGTREM）的函数，signal函数比sigaction函数更为简单，需要对某一个信号比如SIGINT指定一个handler，如果程序运行中收到这个信号，则会由handler负责处理。
但需要注意的是，**signal注册的handler只能被使用一次**，调用handler处理相应信号后，之后再收到这个信号将会按照系统**`默认方式`**处理。

因此，如果需要一直让handler处理这个信号，需要每次在handler函数内部重新用signal函数注册一次handler（一般在handler开始处注册），当然，你可以注册另一个自定义的handler。
但这其实依然有可能带来问题，因为有可能在进入handler后，重新注册handler之前，程序又收到了这个信号，比如SIGINT，那么就按照默认方式处理，即退出程序。

但sigaction不会有这样的问题，不需要重新注册，同时sigaction提供了屏蔽指定信号的功能

# 函数声明
signal的函数声明可以说的上是顶尖难度，如下所示：
>void (*signal(int sig, void (*func)(int)))(int)

可以说，初看一头雾水，但利用David Anderson发明的Clockwise/Spiral Rule（见**References**）可以很好地去理解这个声明，规则如下：
- 从变量名开始，顺时针地螺旋前进，逐一分析语句含义，直到所有的token都被读取
- 括号内的token优先分析

So，let's get started.
从signal开始顺时针读取下一个token是(，说明signal是一个函数，进入了括号，那就先把括号内的token都分析完，也就是内部的这个
>(int sig,void (*func)(int))

说明函数第一个形参是int，第二个继续按照Clockwise/Spiral Rule分析，从func开始，顺时针读取下一个是)，是括号，那优先处理括号内的token，继续顺时针读取下一个是*，说明func是一个指针，继续顺时针读取下一个，得到(，说明func是一个指针，指向一个函数，函数的参数是int，继续顺时针读取，得到void，说明func是一个指针，指向的函数参数为int，返回值为void。

到此，我们回到signal本身，我们可以得出，signal是一个函数，有两个参数，第一个参数是int，第二个参数是函数指针，继续顺时针读取，得到*，说明signal的返回值是指针，继续读取，得到下一个是(，说明signal的返回值是一个指针，指向函数，函数的参数是int，继续读取，得到void，到此，所有token分析完毕，signal是一个函数，有两个参数，其中参数一是int，参数二是函数指针，是指向参数为int，返回值为void的函数，signal的返回值也是个函数指针，指向参数为int，返回值为void的函数


# 示例程序
下面是signal函数的简单应用
```c
#include<stdio.h>
#include<stdlib.h>
#include<signal.h>
#include<windows.h>
int num=1;

void sigintHandler(int);

int main(void){
    signal(SIGINT, sigintHandler);
    while(TRUE){
        printf("main thread running\n");
        Sleep(1000);
        raise(SIGINT);
    }
}

/* Signal Handler for SIGINT */
void sigintHandler(int sig_num){
    signal(SIGINT, sigintHandler);//重新注册，其实可以注册另一个handler函数
    if(num<10){
        printf("guess what, i am still alive\n");
        num++;
    }else{
        printf("oops, my life ended\n");
        getchar();
        exit(0);
    }
    fflush(stdout);
}
```
# 运行结果
![img](/img/blogs/signal.png)
如图所示，程序在收到SIGINT信号后，会转入sigintHandler函数进行处理，直到第10次接收SIGINT信号程序才退出。
# References
- <https://blog.csdn.net/wangzuxi/article/details/44814825>
- http://www.c-faq.com/decl/spiral.anderson.html