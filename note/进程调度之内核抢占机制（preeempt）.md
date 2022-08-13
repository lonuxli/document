# 进程调度之内核抢占机制（preeempt）

早期的Linux核心是不可抢占的。它的调度方法是：**一个进程可以通过schedule\(\)函数自愿地启动一次调度。****非自愿的强制性调度只能发生在每次从系统调用返回的前夕以及每次从中断或异常处理返回到用户空间的前夕**。但是，如果在系统空间发生中断或异常是不会引起调度的。这种方式使内核实现得以简化。但常存在下面两个问题：

     1、如果这样的中断发生在内核中,本次中断返回是不会引起调度的,而要到最初使CPU从用户空间进入内核空间的那次系统调用或中断\(异常\)返回时才会发生调度。

     2、另外一个问题是优先级反转。在Linux中，在核心态运行的任何操作都要优先于用户态进程，这就有可能导致优先级反转问题的出现。例如，一个低优先级的用户进程由于执行软/硬中断等原因而导致一个高优先级的任务得不到及时响应。

当前的Linux内核加入了内核抢占\(preempt\)机制。内核抢占指用户程序在执行系统调用期间可以被抢占，该进程暂时挂起，使新唤醒的高优先级进程能够运行。这种抢占并非可以在内核中任意位置都能安全进行，比如在临界区中的代码就不能发生抢占。临界区是指同一时间内不可以有超过一个进程在其中执行的指令序列。在Linux内核中这些部分需要用自旋锁保护。

内核抢占要求内核中所有可能为一个以上进程共享的变量和数据结构就都要通过互斥机制加以保护，或者说都要放在临界区中。在抢占式内核中，认为如果内核不是在一个中断处理程序中，并且不在被 spinlock等互斥机制保护的临界代码中，就认为可以"安全"地进行进程切换。

Linux内核将临界代码都加了互斥机制进行保护，同时，还在运行时间过长的代码路径上插入调度检查点，打断过长的执行路径，这样，任务可快速切换进程状态，也为内核抢占做好了准备。

Linux内核抢占只有在内核正在执行例外处理程序（通常指系统调用）并且允许内核抢占时，才能进行抢占内核。禁止内核抢占的情况列出如下：

（1）内核执行中断处理例程时不允许内核抢占，中断返回时再执行内核抢占。

（2）当内核执行软中断或tasklet时，禁止内核抢占，软中断返回时再执行内核抢占。

（3）在临界区禁止内核抢占，临界区保护函数通过抢占计数宏控制抢占，计数大于0，表示禁止内核抢占。

**抢占式内核实现的原理是在释放自旋锁时或从中断返回时，如果当前执行进程的 need\_resched 被标记，则进行抢占式调度（抢占式内在自旋锁及中断返回时会发生调度，发生调度的点比非抢占式内核多几个地方）**。

Linux内核在线程信息结构上增加了成员preempt\_count作为内核抢占锁，为0表示可以进行内核高度，它随spinlock和rwlock等一起加锁和解锁。线程信息结构thread\_info列出如下（在include/asm\-x86/thread\_info.h中）：

struct thread\_info {

struct task\_struct \*task;     /\*主任务结构 \*/

struct exec\_domain \*exec\_domain;     /\* 执行的域\*/

\_\_u32 flags;     /\* low level flags \*/

\_\_u32 status;     /\* 线程同步标识\*/

\_\_u32 cpu;     /\* 当前的CPU \*/

int preempt\_count;     /\* 0 =\> 可以抢占（preemptable）,\<0 =\> BUG \*/

mm\_segment\_t addr\_limit;

struct restart\_block restart\_block;

\#ifdef CONFIG\_IA32\_EMULATION

void \_\_user \*sysenter\_return;

\#endif

};

\#endif

 

内核调度器的入口为preempt\_schedule\(\)，他将当前进程标记为TASK\_PREEMPTED状态再调用schedule\(\)，在TASK\_PREEMPTED状态，schedule\(\)不会将进程从运行队列中删除。

内核抢占API函数

在中断或临界区代码中，线程需要关闭内核抢占，因此，互斥机制（如：自旋锁（spinlock）、RCU等）、中断代码、链表数据遍历等需要关闭内核抢占，临界代码运行完时，需要开启内核抢占。关闭/开启内核抢占需要使用内核抢占API函数preempt\_disable和preempt\_enable。

内核抢占API函数说明如下（在include/linux/preempt.h中）：

preempt\_enable\(\) //内核抢占计数preempt\_count减1

preempt\_disable\(\) //内核抢占计数preempt\_count加1

preempt\_enable\_no\_resched\(\)　 //内核抢占计数preempt\_count减1，但不立即抢占式调度

preempt\_check\_resched \(\) //如果必要进行调度

preempt\_count\(\) //返回抢占计数

preempt\_schedule\(\) //核抢占时的调度程序的入口点

　　内核抢占API函数的实现宏定义列出如下（在include/linux/preempt.h中）：

\#define preempt\_disable\(\) /

do { /

inc\_preempt\_count\(\); /

barrier\(\); / //加内存屏障，阻止gcc编译器对内存进行优化

} while \(0\)

\#define inc\_preempt\_count\(\) /

do { /

preempt\_count\(\)\+\+; /

} while \(0\)

\#define preempt\_count\(\) \(current\_thread\_info\(\)\-\>preempt\_count\)

内核抢占调度

**Linux内核在硬中断或软中断返回时会检查执行抢占调度**。分别说明如下：

（1）硬中断返回执行抢占调度

Linux内核在硬中断或出错退出时执行函数retint\_kernel，运行抢占函数，函数retint\_kernel列出如下（在arch/x86/entry\_64.S中）：

\#ifdef CONFIG\_PREEMPT

/\* 返回到内核空间，检查是否需要执行抢占\*/

/\* 寄存器rcx存放threadinfo地址，此时，中断关闭\*/

ENTRY\(retint\_kernel\)

cmpl $0,threadinfo\_preempt\_count\(%rcx\)

jnz retint\_restore\_args

bt $TIF\_NEED\_RESCHED,threadinfo\_flags\(%rcx\)

jnc retint\_restore\_args

bt $9,EFLAGS\-ARGOFFSET\(%rsp\) /\* 中断是否关闭? \*/

jnc retint\_restore\_args

call preempt\_schedule\_irq

jmp exit\_intr

\#endif

函数preempt\_schedule\_irq是出中断上下文时内核抢占调度的入口点，该函数被调用和返回时中断应关闭，保护此函数从中断递归调用。该函数列出如下（在kernel/sched.c中）：

asmlinkage void \_\_sched preempt\_schedule\_irq\(void\)

{

struct thread\_info \*ti = current\_thread\_info\(\);

 

/\* 用于捕捉需要修补的调用者 \*/

BUG\_ON\(ti\-\>preempt\_count || \!irqs\_disabled\(\)\);

 

do {

 /\*内核抢占计数加一个较大的值PREEMPT\_ACTIVE，表示正在处理抢占，由于计数值较大，基本上不会再进行抢占调度\*/

add\_preempt\_count\(PREEMPT\_ACTIVE\); 

local\_irq\_enable\(\); /\*开启中断\*/

schedule\(\); /\*内核调度，用于内核抢占，即运行优先级较高的任务\*/

local\_irq\_disable\(\); /\*关闭中断\*/

sub\_preempt\_count\(PREEMPT\_ACTIVE\);

 

/\*再次检查，避免在调度与现在时刻之间失去抢占机会\*/

barrier\(\); /\*加内存屏障\*/

} while \(unlikely\(test\_thread\_flag\(TIF\_NEED\_RESCHED\)\)\);

}

调度函数schedule会检测进程的 preempt\_counter 是否很大，避免普通调度时又执行内核抢占调度。

（2）软中断返回执行抢占调度

在打开页出错函数pagefault\_enable和软中断底半部开启函数local\_bh\_enable中，会调用函数preempt\_check\_resched检查是否需要执行内核抢占。如果不是并能调度，进程才可执行内核抢占调度。函数preempt\_check\_resched列出如下：

\#define preempt\_check\_resched\(\) /

do { / /\*如果不是普通调度，才可执行抢占调度\*/

if \(unlikely\(test\_thread\_flag\(TIF\_NEED\_RESCHED\)\)\) / 

preempt\_schedule\(\); /

} while \(0\)

函数preempt\_schedule源代码与函数preempt\_schedule\_irq基本上一样，对进程进行调度，这里不再分析。
