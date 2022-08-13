# 并发同步之spin_lock

**spin\_lock：**

spin\_lock主要实现了一个无休眠忙等的排队原子锁，同时还会关闭抢占。

```
  8 #define TICKET_SHIFT▼   16
  9
10 typedef struct {
11 ▼       union {
12 ▼       ▼       u32 slock;
13 ▼       ▼       struct __raw_tickets {
14 #ifdef __ARMEB__
15 ▼       ▼       ▼       u16 next;  //当前排队序号，每个获取锁的路径持有一个本地的排队序号。
                                       //如果排队序号和全局的锁当前运行序号相等，则相应的内核路径获取到锁
16 ▼       ▼       ▼       u16 owner;  //锁当前运行序号,是一个全局的状态
17 #else
18 ▼       ▼       ▼       u16 owner;
19 ▼       ▼       ▼       u16 next;
20 #endif
21 ▼       ▼       } tickets;
22 ▼       };
23 } arch_spinlock_t;                                                                                                                                                                                                  

141 static inline void __raw_spin_lock(raw_spinlock_t *lock)
142 {
143 ▼       preempt_disable();
            //如果系统没有打开CONFIG_LOCKDEP和CONFIG_LOCK_STAT选项，spin_acquire函数实际是一个空函数
144 ▼       spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);       
145 ▼       LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
146 }

110 void do_raw_spin_lock(raw_spinlock_t *lock)
111 {
112 ▼       debug_spin_lock_before(lock);
113 ▼       arch_spin_lock(&lock->raw_lock);
114 ▼       debug_spin_lock_after(lock);
115 }

73 static inline void arch_spin_lock(arch_spinlock_t *lock)
74 {
75 ▼       unsigned long tmp;
76 ▼       u32 newval;
77 ▼       arch_spinlock_t lockval;
78
79 ▼       prefetchw(&lock->slock);
80 ▼       __asm__ __volatile__(
81 "1:▼    ldrex▼  %0, [%3]\n"    //读取入参lock到局部变量lockval=lock->slock
82 "▼      add▼    %1, %0, %4\n" 
83 "▼      strex▼  %2, %1, [%3]\n"  //原子的完成lock->tickets.next加1，加完之后lock->tickets.next就是该上下文获取锁的顺序
84 "▼      teq▼    %2, #0\n"
85 "▼      bne▼    1b"
86 ▼       : "=&r" (lockval), "=&r" (newval), "=&r" (tmp)
87 ▼       : "r" (&lock->slock), "I" (1 << TICKET_SHIFT)
88 ▼       : "cc");
89
90 ▼       while (lockval.tickets.next != lockval.tickets.owner) {  
            //判断当前锁lock->tickets.owner是否与局部变量即该上下文锁获得到的排序lockval.tickets.next相等
            //如果不相等的话，cpu通过函数wfe继续进入休眠
91 ▼       ▼       wfe();
92 ▼       ▼       lockval.tickets.owner = ACCESS_ONCE(lock->tickets.owner);  //更新锁当前所运行到的序号
93 ▼       }
94
95 ▼       smp_mb();
96 }

124 static inline void arch_spin_unlock(arch_spinlock_t *lock)
125 {
126 ▼       smp_mb();
127 ▼       lock->tickets.owner++;   //释放锁时，将当前锁运行序号加1
128 ▼       dsb_sev();                //唤醒其他因为锁调用了WFE指令的cpu
129 }

#define preempt_disable() \
do { \
        preempt_count_inc(); \
        barrier(); \
} while (0)

#define preempt_count_inc() preempt_count_add(1)

#define preempt_count_add(val)  __preempt_count_add(val)

static __always_inline void __preempt_count_add(int val)
{
        *preempt_count_ptr() += val;
}

static __always_inline int *preempt_count_ptr(void)
{
        return &current_thread_info()->preempt_count;
}
```

**spin\_lock\_irq：**

spin\_lock\_irq是在spin\_lock的基础上关闭本地CPU中断，避免在该cpu出现进程被中断抢占，但是实际是进程上下文持有锁，又无法再次被调度到，出现死锁的场景。

那为何只是关闭本地CPU中断呢？CPU0进上下文进入临界区，这时在CPU1上发生中断并进入尝试获取锁。不会导致持有锁的CPU0 进程得不到运行，不会出现死锁的场景。

```
126 static inline void __raw_spin_lock_irq(raw_spinlock_t *lock)
127 {
128 ▼       local_irq_disable();
129 ▼       preempt_disable();
130 ▼       spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
131 ▼       LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
132 }

166 static inline void __raw_spin_unlock_irq(raw_spinlock_t *lock)
167 {
168 ▼       spin_release(&lock->dep_map, 1, _RET_IP_);
169 ▼       do_raw_spin_unlock(lock);
170 ▼       local_irq_enable();
171 ▼       preempt_enable();
172 }
```

**spin\_lock\_bh：**

spin\_lock\_bh是在spin\_lock的基础上上关闭软中断，在软中断和进程上下文竞争时使用

```
134 static inline void __raw_spin_lock_bh(raw_spinlock_t *lock)
135 {
136 ▼       __local_bh_disable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
137 ▼       spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
138 ▼       LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
139 }

174 static inline void __raw_spin_unlock_bh(raw_spinlock_t *lock)
175 {
176 ▼       spin_release(&lock->dep_map, 1, _RET_IP_);
177 ▼       do_raw_spin_unlock(lock);
178 ▼       __local_bh_enable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
179 }
```

**spin\_lock\_irqsave:**

```
106 static inline unsigned long __raw_spin_lock_irqsave(raw_spinlock_t *lock)
107 {
108 ▼       unsigned long flags;
109
110 ▼       local_irq_save(flags); //spin_lock_irqsave只是在spin_lock_irq基础上，在关中断时保留了当前中断状态标志
111 ▼       preempt_disable();
112 ▼       spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
121 ▼       do_raw_spin_lock_flags(lock, &flags);
123 ▼       return flags;
124 }

157 static inline void __raw_spin_unlock_irqrestore(raw_spinlock_t *lock,
158 ▼       ▼       ▼       ▼       ▼           unsigned long flags)
159 {
160 ▼       spin_release(&lock->dep_map, 1, _RET_IP_);
161 ▼       do_raw_spin_unlock(lock);
162 ▼       local_irq_restore(flags);
163 ▼       preempt_enable();
164 }
```
