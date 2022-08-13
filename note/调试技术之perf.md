# 调试技术之perf

[https://www.ibm.com/developerworks/cn/linux/l\-cn\-perf1/](https://www.ibm.com/developerworks/cn/linux/l-cn-perf1/)

[https://www.ibm.com/developerworks/cn/linux/l\-cn\-perf2/](https://www.ibm.com/developerworks/cn/linux/l-cn-perf2/)

1、Perf 可以计算每个时钟周期内的指令数，称为 IPC，IPC 偏低表明代码没有很好地利用 CPU。

2、Perf 还可以对程序进行函数级别的采样，从而了解程序的性能瓶颈究竟在哪里等等。

3、Perf 还可以替代 strace，可以添加动态内核 probe 点。

4、可以做 benchmark 衡量调度器的好坏
