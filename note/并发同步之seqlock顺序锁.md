# 并发同步之seqlock顺序锁

内核版本：Linux 4.9

**一、概述**

普通的spin lock对待reader和writer是一视同仁，RW spin lock给reader赋予了更高的优先级，那么有没有让writer优先的锁的机制呢？答案就是seqlock。顺序锁为写者赋予了较高的优先级，即使在读者正在读的时候，也允许写着继续运行。这种策略的好处是，写者永远不会等待，缺点是有时候读者不得不反复多次读相同的数据，直到它获得有效的副本。

**二、数据结构**

```
typedef struct seqcount {
        unsigned sequence;
} seqcount_t;

typedef struct {
        struct seqcount seqcount;  //seqcount/sequence 只是记录当前写者进入临界区的情况，写者进临界区时+1，出临界区加+1，奇数表示当前有写者进入临界区
        spinlock_t lock;  //自旋锁保护写者互斥，同一时刻只有一个写者进入临界区
} seqlock_t;
```

**三、实现原理**

**3.1 初始化**

顺序锁的初始化本质主要由两部分组成，一是初始化spinlock锁，二是初始化seqcount计数为0

**静态初始化**

```
#define __SEQLOCK_UNLOCKED(lockname)                    \
        {                                               \
                .seqcount = SEQCNT_ZERO(lockname),      \
                .lock = __SPIN_LOCK_UNLOCKED(lockname)  \
        }

#define DEFINE_SEQLOCK(x) \
                seqlock_t x = __SEQLOCK_UNLOCKED(x)
```

**动态初始化**

```
#define seqlock_init(x)                                 \
        do {                                            \
                   (&(x)->seqcount);          \
                spin_lock_init(&(x)->lock);             \
        } while (0)

#define seqcount_init(s)                               \
        do {                                            \
                static struct lock_class_key __key;     \
                __seqcount_init((s), #s, &__key);       \
        } while (0)

static inline void __seqcount_init(seqcount_t *s, const char *name,
                                          struct lock_class_key *key)
{
        /*
         * Make sure we are not reinitializing a held lock:
         */
        lockdep_init_map(&s->dep_map, name, key, 0);
        s->sequence = 0;
}
```

**3.2 写操作**

对于写操作，对临界区的操作如下：

```
write_seqlock(&seq_lock);
/* 修改数据 */
......
write_sequnlock(&seq_lock);
```

顺序锁的本质是通过spinlock保护写者独占临界区，在进入临界区上锁时sequence加1，退出临界区解锁时sequence也加1。

所以顺序锁不能多个写者同时进入临界区。

```
static inline void write_seqlock(seqlock_t *sl)
{
        spin_lock(&sl->lock);
        write_seqcount_begin(&sl->seqcount);
}
static inline void write_seqcount_begin(seqcount_t *s)
{
        write_seqcount_begin_nested(s, 0);
}
static inline void write_seqcount_begin(seqcount_t *s)
{
        write_seqcount_begin_nested(s, 0);
}
static inline void write_seqcount_begin_nested(seqcount_t *s, int subclass)
{
        raw_write_seqcount_begin(s);
        seqcount_acquire(&s->dep_map, subclass, 0, _RET_IP_);
}
static inline void raw_write_seqcount_begin(seqcount_t *s)
{
        s->sequence++;  //上锁时将计数+1
        smp_wmb();  //内存屏障保证多核数据同步
}

static inline void write_sequnlock(seqlock_t *sl)
{
        write_seqcount_end(&sl->seqcount);
        spin_unlock(&sl->lock);
}
static inline void write_seqcount_end(seqcount_t *s)
{
        seqcount_release(&s->dep_map, 1, _RET_IP_);
        raw_write_seqcount_end(s);
}
static inline void raw_write_seqcount_end(seqcount_t *s)
{
        smp_wmb();  //内存屏障保证临界资源已经同步好，避免出现计数增加了而被保护的数据却还未真正同步掉
        s->sequence++;  //解锁时将计数+1
}
```

**3.3 读操作**

通常读操作临界区如下：

```
unsigned int seq;
do {
    seq = read_seqbegin(&seq_lock);
    /* 读取数据 */
    ......
} while read_seqretry(&seq_lock, seq);  //通过读取临界数据前后，判断sequence的是否相等来判断当前读过程中是否有写者进入临界区，如果有写者进入，则读端重新读取次数据
```

代码实现

```
static inline unsigned read_seqbegin(const seqlock_t *sl)
{
        return read_seqcount_begin(&sl->seqcount);
}
static inline unsigned read_seqcount_begin(const seqcount_t *s)
{
        seqcount_lockdep_reader_access(s);
        return raw_read_seqcount_begin(s);
}
static inline unsigned raw_read_seqcount_begin(const seqcount_t *s)
{
        unsigned ret = __read_seqcount_begin(s);
        smp_rmb();
        return ret;
}
static inline unsigned __read_seqcount_begin(const seqcount_t *s)
{
        unsigned ret;
repeat:
        ret = READ_ONCE(s->sequence);
        if (unlikely(ret & 1)) {  //如果是奇数，说明当前有写者进入临界区，则继续自旋读取sequence，知道写者退出临界区
                cpu_relax(); //空操作，自旋
                goto repeat; 
        }
        return ret;
}
```

在读操作中可以看到有两层循环

```
for (;;) {
  
    //第一层循环：通过在读取数据前后获取sequence是否一致来判断读取数据的过程中是否有写操作，如果有写操作写会导致读取的数据不完整
    /*读取数据*/
    for (;;) {
        //第二层循环：通过判断sequence的奇偶性，来判断写操作是否完成，如果写操作未完成，直接返回上去此时读取到的数据也无意义
        /*判断sequence的奇偶性*/
    }
}
```

**四、适用场景**

顺序锁并不是万能的，适合它的使用场景要满足下面的条件：

**A、比较适合读较多写较少的获取修改数据的场景**_。_

写者是有自旋锁保护的，因此一次只能有一个写者写入数据，而读者没有任何其它锁保护，是并发读取的。所以本来写的性能就不高，而读者要保证在读数据的整个期间不会有写者写入，如果写者有很多的话，就会不停的重新尝试读取，也会严重影响性能。 一个writer只会被其他writer造成starve，而不再会被reader造成starve。

**B、被保护的不应该是真正被保护的数据的指针。**

写者可能会在读者正在读指针指向的数据的时候就将该指针变失效了。

    

**C、读者的临界区代码除了读数据外没有别的会引起其它副作用的操作。**

**D、被保护的数据不应该太大，不然读者可能总是被写者打断，造成读者延迟和耗时增�**�

在Linux内核中典型的应用场景是时间子系统中读取时间数据：

```
do {
    seq = read_seqbegin(&jiffies_lock);
    ret = jiffies_64;
} while (read_seqretry(&jiffies_lock, seq));
```

根据写者的上下文（中断上半部、中断下半部）需要，使用的spinlock类型也需要不同，因此写者的API也有所不同

```
static inline void write_seqlock_bh(seqlock_t *sl)
static inline void write_sequnlock_bh(seqlock_t *sl)

static inline void write_seqlock_irq(seqlock_t *sl)
static inline void write_sequnlock_irq(seqlock_t *sl)
```

**五、总结**

从spinlock的“一读或一写”，到rwlock的“并发多读 或 一写”，再到seqlock的“多读和一写并发”，读不会阻塞写，RCU的读写并发。

rwlock和seqlock比较相似，那么其主要的区别呢？

**rwlock存在的问题：**

看起来rwlock比原生的spinlock控制更加精细，应用起来应该更加高效对不对？但rwlock存在一个问题，如果现在有多个reader在使用某个rwlock，那么writer需要等到所有的reader都释放了这个rwlock，才可以获取到，这容易造成writer执行的延迟，俗称饥饿\(starve\)。而且，在writer等待期间，reader还可以不断地加入进来执行，这对writer来说实在是太不公平了。即便writer的优先级更高，也不能先于优先级更低的reader执行，身份（是reader还是writer）决定一切。

现在的内核开发已经不建议再使用rwlock了，之前的Linux代码中使用到的rwlock也在逐渐被移除，或者替换为普通的spinlock或者RCU。
