# 进程调度之stat/top统计信息分析

top命令的统计信息来源于/proc/stat文件导出

cat /proc/stat  \-》 show\_stat     

![6788dda68e08440e77c734b89d80f333.png](image/6788dda68e08440e77c734b89d80f333.png)

![b682645501d3ee68c87c228a0e7a2613.png](image/b682645501d3ee68c87c228a0e7a2613.png)

timer\_tick \-\>update\_process\_times \-》account\_process\_tick 

在系统滴答时钟中断中处理更新统计信息，user\_tick表明被滴答时钟中断的是内核态还是用户态。

如果定义了CONFIG\_IRQ\_TIME\_ACCOUNTING，则在函数irqtime\_account\_process\_tick中更新cpustat时间

如果没有定义CONFIG\_IRQ\_TIME\_ACCOUNTING，则在account\_process\_tick函数的末尾更新cpustat时间

![58fdcd6790adc5e3847b11fe40b4a801.png](image/58fdcd6790adc5e3847b11fe40b4a801.png)

【一】user/nice

account\_guest\_time    

nice: 静态优先级大于120，nice大于0的用户进程用户态所占用的时间（linux 4.9.94内核版本）。 在ubuntu 12.04上验证过（linux 3.10内核版本），网上有些文章说是优先级小于120的进程统计，是错误的。

![c8d2cfbac4a412a49c25de2d4d2dfa8a.png](image/c8d2cfbac4a412a49c25de2d4d2dfa8a.png)

![94e730267ff78fef3560a1f4519cb626.png](image/94e730267ff78fef3560a1f4519cb626.png)

【二】idle/iowait

1、当cpu处于idle状态时，有可能是有task处于等待io的状态，也有可能无任何task在等待io。此时可用rq\-\>nr\_iowait来记录处于io等待的task数量。

2、当cpu处于idle状态且，且存在rq\-\>nr\_iowait大于0则将这段时间作为CPU IO等待时间

3、cpu状态中的iowait高，是否表示系统压力较大？

![d82ed9df48674a0bcc43272c9504b66d.png](image/d82ed9df48674a0bcc43272c9504b66d.png)

![c9cffa006d954724035b87848ebf1142.png](image/c9cffa006d954724035b87848ebf1142.png)

【三】CONFIG\_IRQ\_TIME\_ACCOUNTING

在硬件中断和软件中断，入口调用account\_irq\_enter\_time函数，出口调用account\_irq\_exit\_time函数

![c93b324e77484e9a815c507697c24ee7.png](image/c93b324e77484e9a815c507697c24ee7.png)

**    i****rqtime\_account\_irq 函数统计**

![1431d41094b84430a29d810502513be0.png](image/1431d41094b84430a29d810502513be0.png)

只有定义了CONFIG\_IRQ\_TIME\_ACCOUNTING：

1、irqtime\_account\_irq才是非空。这样才能在中断和软中断入口出口记录准确的时间耗时。

2、ccount\_process\_tick函数中调用rqtime\_account\_process\_tick函数，用准确的中断时间耗时去更新各项统计计数

如果没有定义CONFIG\_IRQ\_TIME\_ACCOUNTING：

1、耗时用cputime = cputime\_one\_jiffy; 将一次周期调度固定的时间算入各项统计计数。

2、这个时间是较为粗略的，没有上面通过出入口记录时间值精确。还会如果周期调度频率低，则采样间隔大，中断处理一般较快，会出现采样不到的可能。表现出来top中看到的中断耗时为0或者较低。

update\_rq\_clock\_task

![3a3a8d46e6b83d85a360adc7afa72728.png](image/3a3a8d46e6b83d85a360adc7afa72728.png)

irqtime\_account\_process\_tick\-》account\_other\_time \-》irqtime\_account\_hi\_update \-》irqtime\_account\_update\-》cpustat\[idx\] \+= irq\_cputime;

![09e0ff884f42e67128845d015f309372.png](image/09e0ff884f42e67128845d015f309372.png)
