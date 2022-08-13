# 中断管理之ARM64中断栈变化

CPU平台：ARM64

linux版本：linux\-5.8.9

ARM64的linux内核在2015年之前中断没有自己独立的栈，都是借用线程的内核栈。公用一个栈的情况下，为了防止内核栈的溢出，内核栈只能强制使用16kB的，不能使用8kB的，这样就存在一个问题，当系统有大量线程的时候，16kB的内核栈对系统内存会是一个很大的浪费。 于是后来社区人员为ARM64添加了中断栈的feature，用来保存中断的上下文。

1. 中断栈的创建：内核启动时中会去为每个cpu创建一个per cpu的中断栈：start\_kernel\-\>init\_IRQ\-\>init\_irq\_stacks

2. 中断栈的使用：中断发生和退出的时候调用irq\_stack\_entry和irq\_stack\_exit来进入和退出中断栈。

```
        .macro  irq_stack_entry
        mov     x19, sp                 // preserve the original sp
#ifdef CONFIG_SHADOW_CALL_STACK
        mov     x24, scs_sp             // preserve the original shadow stack
#endif
        /*
         * Compare sp with the base of the task stack.
         * If the top ~(THREAD_SIZE - 1) bits match, we are on a task stack,
         * and should switch to the irq stack.
         */
        ldr     x25, [tsk, TSK_STACK]
        eor     x25, x25, x19
        and     x25, x25, #~(THREAD_SIZE - 1)
        cbnz    x25, 9998f

        ldr_this_cpu x25, irq_stack_ptr, x26
        mov     x26, #IRQ_STACK_SIZE
        add     x26, x25, x26

        /* switch to the irq stack */
        mov     sp, x26

#ifdef CONFIG_SHADOW_CALL_STACK
        /* also switch to the irq shadow stack */
        adr_this_cpu scs_sp, irq_shadow_call_stack, x26
#endif
9998:
        .endm

        /*
         * The callee-saved regs (x19-x29) should be preserved between
         * irq_stack_entry and irq_stack_exit, but note that kernel_entry
         * uses x20-x23 to store data for later use.
         */
        .macro  irq_stack_exit
        mov     sp, x19
#ifdef CONFIG_SHADOW_CALL_STACK
        mov     scs_sp, x24
#endif
        .endm
```

```
        .macro  irq_handler
        ldr_l   x1, handle_arch_irq
        mov     x0, sp
        irq_stack_entry
        blr     x1
        irq_stack_exit
        .endm
```

[https://lwn.net/Articles/666991/](https://lwn.net/Articles/666991/)

[https://zhuanlan.zhihu.com/p/185851980](https://zhuanlan.zhihu.com/p/185851980)
