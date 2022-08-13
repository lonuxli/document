# 内存管理之用户内存overcommit

**一、什么是内存的overcommit**

Memory Overcommit的意思是操作系统承诺给进程的内存大小超过了实际可用的内存。一个保守的操作系统不会允许memory overcommit，有多少就分配多少，再申请就没有了，这其实有些浪费内存，因为进程实际使用到的内存往往比申请的内存要少，比如某个进程malloc\(\)了200MB内存，但实际上只用到了100MB，按照UNIX/Linux的算法，物理内存页的分配发生在使用的瞬间，而不是在申请的瞬间，也就是说未用到的100MB内存根本就没有分配，这100MB内存就闲置了。下面这个概念很重要，是理解memory overcommit的关键：commit\(或overcommit\)针对的是内存申请，内存申请不等于内存分配，内存只在实际用到的时候才分配。

Linux是允许memory overcommit的，只要你来申请内存我就给你，寄希望于进程实际上用不到那么多内存，但万一用到那么多了呢？那就会发生类似“银行挤兑”的危机，现金\(内存\)不足了。Linux设计了一个OOM killer机制\(OOM = out\-of\-memory\)来处理这种危机：挑选一个进程出来杀死，以腾出部分内存，如果还不够就继续杀…也可通过设置内核参数 vm.panic\_on\_oom 使得发生OOM时自动重启系统。这都是有风险的机制，重启有可能造成业务中断，杀死进程也有可能导致业务中断，所以Linux 2.6之后允许通过内核参数 vm.overcommit\_memory 禁止memory overcommit。

内核参数 vm.overcommit\_memory 接受三种取值：

- 0 – Heuristic overcommit handling. 这是缺省值，它允许overcommit，但过于明目张胆的overcommit会被拒绝，比如malloc一次性申请的内存大小就超过了系统总内存。Heuristic的意思是“试探式的”，内核利用某种算法（对该算法的详细解释请看文末）猜测你的内存申请是否合理，它认为不合理就会拒绝overcommit。
- 1 – Always overcommit. 允许overcommit，对内存申请来者不拒。
- 2 – Don’t overcommit. 禁止overcommit。计算系统的commit值，判断是否超出，如果超出不能申请。

**二、代码分析**

内核在为用户态进程分配内存的路径点，都会调用到\_\_vm\_enough\_memory\(\)根据overcommit\_memory的值来判断内存是否够用能否满足分配动作。

```
#define OVERCOMMIT_GUESS                0
#define OVERCOMMIT_ALWAYS               1
#define OVERCOMMIT_NEVER                2

int __vm_enough_memory(struct mm_struct *mm, long pages, int cap_sys_admin)
{
        long allowed;

        vm_acct_memory(pages);

        /*
         * Sometimes we want to use more memory than we have
         */
        if (sysctl_overcommit_memory == OVERCOMMIT_ALWAYS) //如果设置为1，直接返回可以分配
                return 0;

        if (sysctl_overcommit_memory == OVERCOMMIT_GUESS) {  //如果设置为0，在5.8的内核中判断比较简单，只有申请的内存数太离谱就不让分配。在此前的内核版本中，有个复杂的判断，如果可用的（NR_FILE_PAGES + SWAP_PAGES + ..）物理内存大于要申请的虚拟内存量，即可成功申请。
                if (pages > totalram_pages() + total_swap_pages)
                        goto error;
                return 0;
        }

        //下面是设置为2的处理
        allowed = vm_commit_limit(); 
        /*
         * Reserve some for root
         */
        if (!cap_sys_admin)
                allowed -= sysctl_admin_reserve_kbytes >> (PAGE_SHIFT - 10);  //减去为root预留的内存量

        /*
         * Don't let a single process grow so big a user can't recover
         */
        if (mm) {
                long reserve = sysctl_user_reserve_kbytes >> (PAGE_SHIFT - 10);

                allowed -= min_t(long, mm->total_vm / 32, reserve);
        }

        if (percpu_counter_read_positive(&vm_committed_as) < allowed) //vm_committed_as代表当前系统已经commit的内存量，如果小于allowed则表示可以继续申请
                return 0;
error:
        vm_unacct_memory(pages);

        return -ENOMEM;
}

//该函数计算overcommit所允许的内存申请数量
unsigned long vm_commit_limit(void)
{
        unsigned long allowed;

        if (sysctl_overcommit_kbytes)
                allowed = sysctl_overcommit_kbytes >> (PAGE_SHIFT - 10);  //如果设置了sysctl_overcommit_kbytes参数则使用该参数指定的数值
        else
                allowed = ((totalram_pages() - hugetlb_total_pages())  //否则通过sysctl_overcommit_ratio计算，该值默认为50
                           * sysctl_overcommit_ratio / 100); 
        allowed += total_swap_pages;

        return allowed;
}

//内存申请时，增加commit统计
static inline void vm_acct_memory(long pages)
{
        percpu_counter_add_batch(&vm_committed_as, pages, vm_committed_as_batch);
}

//内存释放时，减少commit统计
static inline void vm_unacct_memory(long pages)
{
        vm_acct_memory(-pages);
}
```

参考资料：

[http://linuxperf.com/?p=102](http://linuxperf.com/?p=102)
