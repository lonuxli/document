# 高速缓存之TLB管理

**一、TLB总览**

![TLB高速缓存(VIVT).png](image/TLB高速缓存(VIVT).png)

**二、歧义问题解决方法**

2.1 硬件刷TLB

2.2 软件刷TLB

flush TLb的时机：

1、进程切换时的歧义问题

2、虚拟地址重映射时的歧义问题

**三、Flush TLB接口**

TLB的flush是指将TLB Entry置无效。

|函数名               |功能                                    |使用场景                                                                           |
|---------------------|----------------------------------------|-----------------------------------------------------------------------------------|
|flush\_tlb\_all\(\)  |flush全部TLB                            |内核页表被改变时调用                                                               |
|flush\_tlb\_mm\(\)   |flush某个用户地址空间TLB                |被用来处理整个地址空间的页表操作，比如在fork和exec过程 中发生的事情，如fork、rmap中|
|flush\_tlb\_range\(\)|flush某个用户地址空间特定范围内的TLB    |需要处理某个地址范围内时使用                                                       |
|flush\_tlb\_page     |flush某个用户地址空间的一个page大小的TLB|故障处理时使用处理一个page时使用                                                   |

**四、参考资料**

1、[http://www.wowotech.net/process\_management/context\-switch\-tlb.html](http://www.wowotech.net/process_management/context-switch-tlb.html)

2、[https://www.kernel.org/doc/html/latest/translations/zh\_CN/core\-api/cachetlb.html](https://www.kernel.org/doc/html/latest/translations/zh_CN/core-api/cachetlb.html)
