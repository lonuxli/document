# 进程调度之rt调度类分析

`4.9.37`

（一）

scheduler\_tick\-》curr\-\>sched\_class\-\>task\_tick\(rq, curr, 0\)\-》task\_tick\_rt

![88ab6c0acde62da7319dd0cad94176b7.png](image/88ab6c0acde62da7319dd0cad94176b7.png)

1、SCHED\_FIFO进程直接返回，意味着该类型进程会继续占用CPU而不发生调度

2、SCHED\_RR进程类型，从代码 if \(rt\_se\-\>run\_list.prev \!= rt\_se\-\>run\_list.next\)，如果该进程优先级对应的运行队列中，只有该进程一个进程，那么直接返回，这意味着如果该进程会继续占用CPU而不发生调度。

     如果该进程优先级对应的运行队列中还有其他进程，那么就交替运行。如果该进程优先级对应的队列中有SCHED\_FIFO进程，那么CPU只能由SCHED\_FIFO进程运行。 

3、从以上两点可以看出，如果RT进程不是主动放弃CPU的话，其他低优先级的RT进程或普通进程无法得到运行。通过在ubuntu 12.04上验证可以得出，如果将几个测试进程绑定CPU核，高优先级的RT进程会霸占CPU。

（二）

sched\_yield \-》 current\-\>sched\_class\-\>yield\_task  \-》 yield\_task\_rt \-》 requeue\_task\_rt

如果是rt线程的sched\_yield系统调用，其先将自身放置到运行队列的尾部，然后再调用schedule让出CPU

![a03f041bf3542416f2888b0b4c172f63.png](image/a03f041bf3542416f2888b0b4c172f63.png)

（三）

\_\_schedule \-》 pick\_next\_task \-》pick\_next\_task\_rt,

1、pick\_next\_task函数中先判断是否进程中都是cfs调度类的进程，如果是的话则调用fair\_sched\_class.pick\_next\_task选择下一个进程；如果没有合适的cfs调度类进程则选择idle进程。

2、遍历各种调度类，从高优先级调度类往低优先级调度类遍历查找可运行进程，所以此处表明rt调度类类进程比cfs调度类进程先运行。

![4e33a0b4e5ffd67fee1db3ef05406787.png](image/4e33a0b4e5ffd67fee1db3ef05406787.png)

3、\_pick\_next\_task\_rt\-》 pick\_next\_rt\_entity  找到优先级最高的rt进程

4、put\_prev\_task和dequeue\_pushable\_task的作用是什么？

![ea83ae092d6feafe30b1c50a535a5849.png](image/ea83ae092d6feafe30b1c50a535a5849.png)

四（rt throttled ）

update\_curr\_rt \-》 sched\_rt\_runtime\_exceeded

![da989fdabb631b8c31f39726b1f35cc3.png](image/da989fdabb631b8c31f39726b1f35cc3.png)

[https://blog.csdn.net/wennuanddianbo/article/details/70037415       �](https://blog.csdn.net/wennuanddianbo/article/details/70037415)�

dequeue\_rt\_stack函数中临时变量back的作用，形成反向的指针链表：

![588a9395bf96ba3d464e12bebd0c5bc0.png](image/588a9395bf96ba3d464e12bebd0c5bc0.png)

%\!\(EXTRA markdown.ResourceType=, string=, string=\)

task\_group的组织形式：

![f7c81ece9700a71e13199e3d0d6a7c71.png](image/f7c81ece9700a71e13199e3d0d6a7c71.png)

rt组调度类似于cfs组调度策略：

![22e286f3ff7ef5613e9aa8b880827204.png](image/22e286f3ff7ef5613e9aa8b880827204.png)
