# 时间子系统之clock source

内核版本：4.9

CPU平台：ARM64/HI3531DV200

**一、概述**

clock source就是用来抽象一个在指定输入频率的clock下工作的一个counter。

**二、数据结构分析**

```
struct clocksource {
        cycle_t (*read)(struct clocksource *cs);  //读取时钟源的循环计数值
        cycle_t mask;  //mask最是最大的cycle数目，除以频率就是能表示的最大的时间范围（以秒为单位）
        u32 mult;
        u32 shift;
        u64 max_idle_ns;
        u32 maxadj;
#ifdef CONFIG_ARCH_CLOCKSOURCE_DATA
        struct arch_clocksource_data archdata;
#endif
        u64 max_cycles;  //类似于max_idle_ns
        const char *name;
        struct list_head list;    //注册时挂入全局的时钟源链表clocksource_list
        int rating;
        int (*enable)(struct clocksource *cs);  //使能clock source
        void (*disable)(struct clocksource *cs); //失能clock source
        unsigned long flags;
        void (*suspend)(struct clocksource *cs); //挂起clock sour
        void (*resume)(struct clocksource *cs);  //恢复clock sour

        /* private: */
#ifdef CONFIG_CLOCKSOURCE_WATCHDOG
        /* Watchdog related data, used by the framework */
        struct list_head wd_list;
        cycle_t cs_last;
        cycle_t wd_last;
#endif
        struct module *owner;
};

/*
* Clock source flags bits::
*/
#define CLOCK_SOURCE_IS_CONTINUOUS              0x01
#define CLOCK_SOURCE_MUST_VERIFY                0x02
#define CLOCK_SOURCE_WATCHDOG                   0x10
#define CLOCK_SOURCE_VALID_FOR_HRES             0x20
#define CLOCK_SOURCE_UNSTABLE                   0x40
#define CLOCK_SOURCE_SUSPEND_NONSTOP            0x80
#define CLOCK_SOURCE_RESELECT                   0x100
```

**2.1 时间换算**

read读取上来的是计数器的cycles，但是实际内核中使用纳秒等固定单位才能统一，假定A个cycles，计数器的频率为F，A个计数循环对应的时间为T，则将A转换成T\(d单位纳秒\)公式为：

```
T(转换后的纳秒数目) = (A / F)  x  NSEC_PER_SEC
```

由于内核中不适合做浮点运算及除法运算，以上公式不好直接处理，所以利用提前计算好的mult和shift来计算T，但是这种计算方法会损失精度，计算函数如下：

```
/* clocksource_cyc2ns - converts clocksource cycles to nanoseconds
* @cycles:     cycles
* @mult:       cycle to nanosecond multiplier
* @shift:      cycle to nanosecond divisor (power of two)
*/
static inline s64 clocksource_cyc2ns(u64 cycles, u32 mult, u32 shift)
{
        return ((u64) cycles * mult) >> shift;
}
```

**2.2 mask**

mask表示了该计数器的位树，也表示了计数器所能容纳的最大cycles。

函数void \_\_clocksource\_update\_freq\_scale\(struct clocksource \*cs, u32 scale, u32 freq\) 中使用到，mask / freq 即等于计数器计数满时代表的时间。           

```
                sec = cs->mask;  //将mask赋值给sec
                do_div(sec, freq);  //mask / freq等于计数器所能表示最大时间，单位秒
```

通常mask在代码中定义struct clocksource时显式确定，表明计数器的计数容量，在ARM64实例中mask为0xffffffffffffff。

**2.3 mult和shift计算**

当注册一个clocksource时，内核通过函数clocks\_calc\_mult\_shift\(\)函数为该clocksource计算mult和shift

```
// from：当前计数器counter的输入频率
// to： 在不考虑缩放系数scale的情况下，该入参表示单位form个cycles,可以转换成的纳秒数
        在注册clocksource的函数__clocksource_update_freq_scale中，该参数为NSEC_PER_SEC / scale
// maxsec： 最大可转换时间范围（单位秒），转换范围maxsec影响mult和shift的值，在保证计算不溢出的情况下，如果maxsec值增大，则会降低转换计算精度
        实际使用中适当降低maxsec来提高mult和shift的值，通常最大取600秒。
void clocks_calc_mult_shift(u32 *mult, u32 *shift, u32 from, u32 to, u32 maxsec)
{
        u64 tmp;
        u32 sft, sftacc= 32;

        /* Calculate the shift factor which is limiting the conversion range:*/
        // from是计数器的频率，maxsec是最大转换时间秒。maxsec * from就是将秒数转换成了最大cycle数
        tmp = ((u64)maxsec * from) >> 32;
        while (tmp) {
                tmp >>=1;
                sftacc--;
        }

        /* Find the conversion shift/mult pair which has the best accuracy and fits the maxsec conversion range*/
        // 反推计算公式为：mult = (ns<<shift)/cycles，1秒对应的cycles数为counter的频率，迭代shift从32->1，以此计算出对应的mult
        // 再判断当前计算出的mult是否会溢出，不会溢出则此mult为最优值。为何需要将shift从高到低迭代？因为shift越大，则mult也是越大
        for (sft = 32; sft > 0; sft--) {
                tmp = (u64) to << sft;
                tmp += from / 2; //此处特殊处理为通过四舍五入达到计算出最优值，详见：b5776c4a6d0afc13697e8452b9ebe1cc4d961b74
                do_div(tmp, from);
                if ((tmp >> sftacc) == 0)
                        break;
        }
        *mult = tmp;
        *shift = sft;
}
```

（a）from是count的输入频率，maxsec是最大的表示的范围。maxsec \* from就是将秒数转换成了cycle数目。对于大于32 bit的counter而言，最大的cycle数目有可能需要超过32个bit来表示，因此这里要进行右移32bit的操作。

（b）**sftacc保存了左移多少位才会造成最大cycle数（对应最大的时间范围值）的溢出**（将cycles转换成纳秒的公式为：\(cycles \* mult\) \>\> shift， 在做第一步乘法的时候可能会出现溢出，因此此处计算出的\(1\<\<sftacc\)是mult的最大值，如果大于这个值mult\*cycles会产生64位溢出\)，对于32 bit以下的counter，统一设定为32个bit，而对于大于32 bit的counter，sftacc需要根据tmp值（这时候tmp保存了最大cycle数的高32 bit值）进行计算。

  ![c591b817691382d5c728f4850486da6f.gif](image/c591b817691382d5c728f4850486da6f.gif)

（c）如何获取最佳的mult和shift组合？mult这个因子一定是越大越好，mult越大也就是意味着shift越大。当然shift总有一个起始值，设定起始值为32bit，因此sft从32开始搜索，看看是否满足最大时间范围的要求。如果满足，那么就找到最佳的mult和shift组合，否则要sft递减，进行下一轮搜索。

（d）考虑如何计算mult值。根据公式_（cycles \* mult\) \>\> shift �_�可以得到ns数，由此可以得到计算mult值的公式：

```
mult = (ns<<shift)/cycles
```

如果我们设定ns数是10^9纳秒（也就是1秒）的话，cycles数目就是频率值（所谓频率不就是1秒振荡的次数嘛）。因此上面的公式可以修改为：

```
mult = (NSEC_PER_SEC<<shift)/freq
```

在步骤b中计算得到的sftacc就是multi的最大的bit数目。因此，\(tmp \>\> sftacc\)== 0就是判断找到最优mult的条件。

总结：mult和shift的计算，以1S所拥有的ns数及其对应的cycles（频率值）数代入公式_ ns =（cycl__es \* mult\) \>\> shift�_� 计算，但是由于有两个变量，此公式有多个解，由于需要找到最大且不出现64位溢出的mult和shift组合，采用依次尝试shift从32到1各种情况，找到最先满足要求的一组解。

**2.4 max\_idle\_ns**

传统的Unix都是有一个周期性的tick，如100HZ设定为10ms，但是如果linux kernel也允许你配置成NO\_HZ，这时候系统就不存在周期性的tick了，如果系统一直idle不去获取或者隔很长时间去获取计数器的计数值时，此时去获取会出现计算溢出的情况。由于counter value和纳秒的转换限制，这个idle的时间不能超过max\_idle\_ns。

计算方法：

从cycles转换成ns使用公式：_ns =（cycl__es \* mult\) \>\> shift     �_�

如果cycles太大，则会造成计算溢出 max\(ns\)=max\(cycles\)  \*  mult \>\> shift，考虑到余量和溢出问题，具体计算如下

```
static inline void clocksource_update_max_deferment(struct clocksource *cs)
{
        cs->max_idle_ns = clocks_calc_max_nsecs(cs->mult, cs->shift,
                                                cs->maxadj, cs->mask,
                                                &cs->max_cycles);
}

u64 clocks_calc_max_nsecs(u32 mult, u32 shift, u32 maxadj, u64 mask, u64 *max_cyc)
{
        u64 max_nsecs, max_cycles;

        /*
         * Calculate the maximum number of cycles that we can pass to the
         * cyc2ns() function without overflowing a 64-bit result.
         */
        max_cycles = ULLONG_MAX;
        do_div(max_cycles, mult+maxadj);  //在保证不出现64位溢出的情况下，计算出最大的cycles，因此上面直接用64的最大数据代入计算

        /*
         * The actual maximum number of cycles we can defer the clocksource is
         * determined by the minimum of max_cycles and mask.
         * Note: Here we subtract the maxadj to make sure we don't sleep for
         * too long if there's a large negative adjustment.
         */
        max_cycles = min(max_cycles, mask);  
        max_nsecs = clocksource_cyc2ns(max_cycles, mult - maxadj, shift);

        /* return the max_cycles value as well if requested */
        if (max_cyc)
                *max_cyc = max_cycles;

        /* Return 50% of the actual maximum, so we can detect bad values */
        max_nsecs >>= 1;  //预留50%余量

        return max_nsecs;
}
```

使用场景：

在函数tick\_nohz\_stop\_sched\_tick中会获取max\_idle\_ns，用于nohz的。

**2.5 ratting**

时钟源评分，ratting反应时钟源的质量，其值为1\-499之间，值越高则时钟源精度越好。ratting在代码中定义struct clocksource时显式确定，表明定时器的精度。

```
1-99: Unfit for real use Only available for bootup and testing purposes.
100-199: Base level usability. Functional for real use, but not desired.
200-299: Good. A correct and usable clocksource.
300-399: Desired. A reasonably fast and accurate clocksource.
400-499: Perfect. The ideal clocksource. A must-use where available.
```

clocksource挂入全局的clocksource\_list链表时，是以ratting降序排列，便于系统选择一个精度最高的时钟源。

**三、clock soure实例**

**3.1 vmcore看ARM64 clocksource实例**

\+\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\+

\+\-\-\-\>clocksource\_list \-\-\-\> clocksource\_counter \-\-\-\> clocksource\_jiffies\-\-\-\+

```
struct clocksource {
  read = 0xffffff800866ae50 <arch_counter_read>,
  mask = 0xffffffffffffff,
  mult = 0x29aaaaab,
  shift = 0x18,
  max_idle_ns = 0x66a1710420,
  maxadj = 0x4955555,
  archdata = {
    vdso_direct = 0x1
  },
  max_cycles = 0x588fe9dc0,
  name = 0xffffff800898e0f8 "arch_sys_counter",
  list = {
    next = 0xffffff8008b07b20 <clocksource_jiffies+56>,
    prev = 0xffffff8008b076c0 <clocksource_list>
  },
  rating = 0x190,
  enable = 0x0,
  disable = 0x0,
  flags = 0xa1,
  suspend = 0x0,
  resume = 0x0,
  mark_unstable = 0x0,
  tick_stable = 0x0,
  owner = 0x0
}

struct clocksource {
  read = 0xffffff800810cb50 <jiffies_read>,
  mask = 0xffffffff,
  mult = 0x98968000,
  shift = 0x8,
  max_idle_ns = 0x43e6cfffbc1930,
  maxadj = 0x10c8e000,
  archdata = {
    vdso_direct = 0x0
  },
  max_cycles = 0xffffffff,
  name = 0xffffff8008916c20 "jiffies",
  list = {
    next = 0xffffff8008b076c0 <clocksource_list>,
    prev = 0xffffff8008b35bc8 <clocksource_counter+56>
  },
  rating = 0x1,
  enable = 0x0,
  disable = 0x0,
  flags = 0x0,
  suspend = 0x0,
  resume = 0x0,
  mark_unstable = 0x0,
  tick_stable = 0x0,
  owner = 0x0
}
```

3.2 clocksource dts描述

```
            arm-timer {
                compatible = "arm,armv8-timer";
                interrupts = <1 13 0xf04>,
                             <1 14 0xf04>;
                clock-frequency = <50000000>;
            };

            timer@12000000 {
                compatible = "hisilicon,hisp804";
                reg = <0x12000000 0x20>, /* clocksource */
                      <0x1d840000 0x20>, /* local timer for each cpu */
                      <0x1d840020 0x20>,
                      <0x1d850000 0x20>,
                      <0x1d850020 0x20>;
                interrupts = <0 113 4>, /* irq of local timer0/1 */
                             <0 114 4>, /* irq of local timer2/3 */
                             <0 115 4>, /* irq of local timer4/5 */
                             <0 116 4>; /* irq of local timer6/7 */
                clocks = <&clk_3m>;
                clock-names = "apb_pclk";
            };
```

**3.3 arch\_sys\_counter定义**

```
static struct clocksource clocksource_counter = {
        .name   = "arch_sys_counter",
        .rating = 400,
        .read   = arch_counter_read,
        .mask   = CLOCKSOURCE_MASK(56),
        .flags  = CLOCK_SOURCE_IS_CONTINUOUS,
};
```

**3.4 clocksource 注册**

```
static void __init arch_counter_register(unsigned type)
{
    clocksource_register_hz(&clocksource_counter, arch_timer_rate);
}

static inline int clocksource_register_hz(struct clocksource *cs, u32 hz)
{
        return __clocksource_register_scale(cs, 1, hz);  
}

int __clocksource_register_scale(struct clocksource *cs, u32 scale, u32 freq)
{
        unsigned long flags;

        /* Initialize mult/shift and max_idle_ns */
        __clocksource_update_freq_scale(cs, scale, freq);  //更新计算clocksource中各个元素的值

        /* Add clocksource to the clocksource list */
        mutex_lock(&clocksource_mutex);

        clocksource_watchdog_lock(&flags);
        clocksource_enqueue(cs);           //以ratting逆序排列加入clocksource_list全局链表
        clocksource_enqueue_watchdog(cs);  //加入watchdog_list链表头部，以cs->wd_list元素连接
        clocksource_watchdog_unlock(&flags);

        clocksource_select();               //选择一个最好的时钟源
        clocksource_select_watchdog(false); //选择一个时钟源作为watch_dog时钟
        __clocksource_suspend_select(cs);
        mutex_unlock(&clocksource_mutex);
        return 0;
}
```

**3.5 时钟源选择**

**3.6 clocksource watchdog �**�

五、jiffies

参考资料：

[https://blog.csdn.net/DroidPhone/article/details/7975694](https://blog.csdn.net/DroidPhone/article/details/7975694)
