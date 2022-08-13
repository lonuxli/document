

# 1、中断管理
---
___

## (A) arm

I really like using Markdown.

This is the first line.  
And this is the second line. **Lilong**

>Dorothy followed her through many of the beautiful rooms in her castle.
>this is my code hhh
>>- this is my code too
>> 


## (B) x86
1. 硬件
2. 软件
    > linux
    > app
    > shuju 
3. 正常
4. 不正常
     request_irq(FLOPPY_IRQ, floppy_interrupt, IRQF_DISABLED, "floppy", NULL) `

`code单词` 是显示的很好呢

    __irq_svc:
        svc_entry
        irq_handler

	#ifdef CONFIG_PREEMPT
        get_thread_info tsk
        ldr     r8, [tsk, #TI_PREEMPT]          @ get preempt count
        ldr     r0, [tsk, #TI_FLAGS]            @ get flags
        teq     r8, #0                          @ if preempt count != 0
        movne   r0, #0                          @ force flags to 0
        tst     r0, #_TIF_NEED_RESCHED
        blne    svc_preempt
	#endif

        svc_exit r5, irq = 1                    @ return from exception
	UNWIND(.fnend          )
	ENDPROC(__irq_svc)
- second
- three

# 2、内存管理
# 3、进程调度

* one 
* two
* three
- d

***
zhong文件
————————
---


[链接](https://markdown.com.cn%20great%20page)


|学号|姓名|序号|
|-|-|-|
|小明明|男|5|
|小红|女|79|
|小陆|男|192|

- ATT &lt; <<


| name | age | sex
| :-:|:-|-:
|tony|20|男
|lucy|18|女

![linuxcnc](.\images\linuxcnc001.png)

==Li Long==

2^2^

H~2~o

$\sum_{i=1}^{10}f(i)\,\,\text{thanks}$

> hello markdown!
>> hello markdown!

```python print('hello nick')```

`print('hello nick')`

<https://www.cnblogs.com/nickchen121/p/10718112.html>


[nickchen博客](https://www.cnblogs.com/nickchen121/p/10718112.html%20nickchen博客")

1. one
2. two 
3. three
