# 并发同步之data race

**一、data race（数据竞争） 介绍**

**1.1 什么是data race？�**�

满足以下3个条件即为data race

\* two or more threads in a single process access the same memory location concurrently 两个或两个以上线程同时访问同一变量

\* at least one of the accesses is for writing 至少有一个线程是写访问

\* the threads are not using any exclusive locks to control their accesses to that memory. 对共享变量的访问没有额外的锁保护

**1.2 data race的分类**

Write \-\> Write Data Race

Read \-\> Write Data Race

**1.3 data race的后果**

当以上三个条件data race成立时，线程\(线程\)对内存访问的顺序是不确定的，并且根据不同的访问顺序在运行之后会得出不同的结果。 

一些data race可能是良性的，例如，当内存访问用于忙等待时，通过一个线程修改数据来通知另一个线程退出忙等。

大部分data race是程序的错误，例如，读取该变量时得到的值将变得不可知，使得该多线程程序的运行结果将完全不可预测，可能直接崩溃。

**二、data race程序错误示例**

示例1：线程1可能在线程2初始化完成nlmsvc\_timeout前就读取该变量

![a2a947688571dd1b18280f1ddfbfb11a.png](image/a2a947688571dd1b18280f1ddfbfb11a.png)

示例2：线程1可能在线程2将battery\-\>bat.dev置空后访问，导致空指针

![b0eef52f41a37371d5a167b3bca42a44.png](image/b0eef52f41a37371d5a167b3bca42a44.png)

**三、如何防止data race引起的问题**

对于有可能被多个线程同时访问的变量使用排他访问控制，具体方法包括使用mutex（互斥量）和monitor（监视器），或者使用atomic变量。

**四、race condition概念**

race condition与data race概念是不一样的，解释如下：

Race Around Condition in an operating system is a situation where the result produced by two processes\(or threads\) operated on shared resources depends in an unexpected way on the relative order in which process gains access to the CPU\(s\). 竞态条件的核心是多个线程，没有对共享资源顺序访问，而可能产生破坏或其他预料之外的结果。因此需要正确的使用锁，来保证对共享资源的顺序访问，以避免竞态条件。

**五、参考资料**

1、[https://www.intel.com/content/www/us/en/develop/documentation/inspector\-user\-guide\-linux/top/problem\-type\-reference/data\-race.html](https://www.intel.com/content/www/us/en/develop/documentation/inspector-user-guide-linux/top/problem-type-reference/data-race.html)

2、Chinese J of Electronics \- 2018 \- Shi \- Linux Kernel Data Races in Recent 5 Years.pdf
