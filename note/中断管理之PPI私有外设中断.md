# 中断管理之PPI私有外设中断

基于Linux 4.9

ARM32 GIC\-V2

PPI（Private Peripheral Interrupts）私有外设中断，硬件中断号为16\-32

**一、硬件描述**

PPI中断只能分配给一个确定的CPU核。同一个硬件中断，每个CPU核有一个中断源，每个中断源产生的中断只能送到其对应的CPU核上。

**二、中断处理过程**

1、为PPI设置不一样的处理函数

```
static int gic_irq_domain_map(struct irq_domain *d, unsigned int irq,
                                irq_hw_number_t hw)
{
        if (hw < 32) {
                irq_set_percpu_devid(irq);
                irq_domain_set_info(d, irq, hw, &gic_chip, d->host_data,
                                    handle_percpu_devid_irq, NULL, NULL);  //小于32的中断处理函数
                set_irq_flags(irq, IRQF_VALID | IRQF_NOAUTOEN);
        } else {
                irq_domain_set_info(d, irq, hw, &gic_chip, d->host_data,   //大于32的中断处理函数
                                    handle_fasteoi_irq, NULL, NULL);
                set_irq_flags(irq, IRQF_VALID | IRQF_PROBE);

                gic_routable_irq_domain_ops->map(d, irq, hw);
        }
        return 0;
}

//设置PPI中断描述符处理函数为handle_percpu_devid_irq
__irq_do_set_handler(struct irq_desc *desc, irq_flow_handler_t handle,
                     int is_chained, const char *name)
{
        ...
        desc->handle_irq = handle;  //    
        desc->name = name;
        ...
}
```

2、进入PPI中断处理

```
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
        u32 irqstat, irqnr;
        struct gic_chip_data *gic = &gic_data[0];
        void __iomem *cpu_base = gic_data_cpu_base(gic);

        do {
                irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);  //读取寄存器GICC_IAR 产生ACK信号
                irqnr = irqstat & GICC_IAR_INT_ID_MASK; //获取中断号

                if (likely(irqnr > 15 && irqnr < 1021)) {
                        handle_domain_irq(gic->domain, irqnr, regs); //普通中断走这里，包括PPI中断
                        continue;
                }
                if (irqnr < 16) {
                        writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);
#ifdef CONFIG_SMP
                        handle_IPI(irqnr, regs);
#endif
                        continue;
                }
                break;
        } while (1);
}

static inline int handle_domain_irq(struct irq_domain *domain,
                                    unsigned int hwirq, struct pt_regs *regs)
{
        return __handle_domain_irq(domain, hwirq, true, regs);
}

int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
                        bool lookup, struct pt_regs *regs)
{
        struct pt_regs *old_regs = set_irq_regs(regs);
        unsigned int irq = hwirq;
        int ret = 0;

        irq_enter();

#ifdef CONFIG_IRQ_DOMAIN
        if (lookup)
                irq = irq_find_mapping(domain, hwirq); //根据domain和hwirq找到对应的virq
#endif

        /*
         * Some hardware gives randomly wrong interrupts.  Rather
         * than crashing, do something sensible.
         */
        if (unlikely(!irq || irq >= nr_irqs)) {
                ack_bad_irq(irq);
                ret = -EINVAL;
        } else {
                generic_handle_irq(irq);  //进一步处理中断
        }

        irq_exit();
        set_irq_regs(old_regs);
        return ret;
}

int generic_handle_irq(unsigned int irq)
{
        struct irq_desc *desc = irq_to_desc(irq);  //根据virq获取到中断描述符
        if (!desc)
                return -EINVAL;
        generic_handle_irq_desc(irq, desc);
        return 0;
}

static inline void generic_handle_irq_desc(unsigned int irq, struct irq_desc *desc)
{
        desc->handle_irq(irq, desc); //调用描述符中的回调函数进行处理，由上一节可知gic_irq_domain_map函数中设置回调函数
                                 //小于32：handle_percpu_devid_irq 大于32：handle_fasteoi_irq
}
```

3、handle\_percpu\_devid\_irq详细处理

```
void handle_percpu_devid_irq(struct irq_desc *desc)
{
        struct irq_chip *chip = irq_desc_get_chip(desc);
        struct irqaction *action = desc->action;
        unsigned int irq = irq_desc_get_irq(desc);
        irqreturn_t res;

        kstat_incr_irqs_this_cpu(desc);

        if (chip->irq_ack)
                chip->irq_ack(&desc->irq_data); //对于gic来说，chip->irq_ack是空实现，在读取硬件中断号时就ACK了

        if (likely(action)) {
                trace_irq_handler_entry(irq, action);
                //action->percpu_dev_id存的指针指向设备ID
                res = action->handler(irq, raw_cpu_ptr(action->percpu_dev_id));  //具体的PPI中断处理
                trace_irq_handler_exit(irq, action, res);
        } else {
                unsigned int cpu = smp_processor_id();
                bool enabled = cpumask_test_cpu(cpu, desc->percpu_enabled);

                if (enabled)
                        irq_percpu_disable(desc, cpu);

                pr_err_once("Spurious%s percpu IRQ%u on CPU%u\n",
                            enabled ? " and unmasked" : "", irq, cpu);
        }

        if (chip->irq_eoi)
                chip->irq_eoi(&desc->irq_data);  //gic有实现irq_eoi，写EOI寄存器产生EOI信号
}
```

4、进入具体的PPI处理函数

```
#0  twd_handler (irq=16, dev_id=0xeefc30c0) at arch/arm/kernel/smp_twd.c:233
#1  0xc00d1a70 in handle_percpu_devid_irq (irq=16, desc=0xee806d80) at kernel/irq/chip.c:714
#2  0xc00c9ab8 in generic_handle_irq_desc (desc=0xee806d80, irq=16) at include/linux/irqdesc.h:129
#3  generic_handle_irq (irq=16) at kernel/irq/irqdesc.c:351
#4  0xc00c9c50 in __handle_domain_irq (domain=0xee804600, hwirq=29, lookup=true, regs=0xc0d95c68 <init_thread_union+7272>) at kernel/irq/irqdesc.c:388
#5  0xc00089e4 in handle_domain_irq (regs=0xc0d95c68 <init_thread_union+7272>, hwirq=29, domain=0xee804600) at include/linux/irqdesc.h:147
#6  gic_handle_irq (regs=0xc0d95c68 <init_thread_union+7272>) at drivers/irqchip/irq-gic.c:280
```

**三、PPI中断的注册**

以twd为例，PPI中断的twd模块实例：

ARM 11MP, Cortex\-A5 and Cortex\-A9 are often associated with a per\-core Timer\-Watchdog \(aka TWD\), which provides both a per\-cpu local timer

and watchdog.在ARM 11MP, Cortex\-A5  Cortex\-A9几种架构中，每个CPU核都有一个每CPU timer\-watchdog。

twd \(timer watch dog\)模块在dts中的配置

```
        timer@1e000600 {
                compatible = "arm,cortex-a9-twd-timer";
                reg = <0x1e000600 0x20>;
                //实际硬件中断号是13+16=29，因为第一个元素是1，所以只加16
                //在gic_irq_domain_xlate中转换
                interrupts = <1 13 0xf04>;  
        };

        watchdog@1e000620 {
                compatible = "arm,cortex-a9-twd-wdt";
                reg = <0x1e000620 0x20>;
                interrupts = <1 14 0xf04>;
        };
```

```
static struct clock_event_device __percpu *twd_evt; 

//主要的区别是入参void *dev_id为per_cpu类型，在具体处理时函数handle_percpu_devid_irq
//会根据cpu取出对应的传入参数
static int __init twd_local_timer_common_register(struct device_node *np)
{
    //硬件终端号是twd_ppi通过irq_of_parse_and_map从dts中解析出来的应该是29或30
    request_percpu_irq(twd_ppi, twd_handler, "twd", twd_evt); 
}

//request_percpu_irq与普通的request_irq整体上没有什么区别，主要是设置了per_cpu相关的参数
int request_percpu_irq(unsigned int irq, irq_handler_t handler,
                       const char *devname, void __percpu *dev_id)
{
        struct irqaction *action;
        struct irq_desc *desc;
        int retval;

        if (!dev_id)
                return -EINVAL;
        desc = irq_to_desc(irq);
        if (!desc || !irq_settings_can_request(desc) ||
            !irq_settings_is_per_cpu_devid(desc))
                return -EINVAL;

        action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
        if (!action)
                return -ENOMEM;
        action->handler = handler;
        action->flags = IRQF_PERCPU | IRQF_NO_SUSPEND;
        action->name = devname;
        action->percpu_dev_id = dev_id;
        retval = irq_chip_pm_get(&desc->irq_data);
        if (retval < 0) {
                kfree(action);
                return retval;
        }
        chip_bus_lock(desc);

        retval = __setup_irq(irq, desc, action);
        chip_bus_sync_unlock(desc);
        if (retval) {
                irq_chip_pm_put(&desc->irq_data);
                kfree(action);
        }
        return retval;
}

void handle_percpu_devid_irq(struct irq_desc *desc)
{
        struct irq_chip *chip = irq_desc_get_chip(desc);
        struct irqaction *action = desc->action;
        unsigned int irq = irq_desc_get_irq(desc);
        irqreturn_t res;

        kstat_incr_irqs_this_cpu(desc);

        if (chip->irq_ack)
                chip->irq_ack(&desc->irq_data);

        if (likely(action)) {
                trace_irq_handler_entry(irq, action);
                //hander的入参，通过raw_cpu_ptr取出cpu核对应的参数，对于普通中断的话这个参数在不同核上是相同的
                res = action->handler(irq, raw_cpu_ptr(action->percpu_dev_id));
                trace_irq_handler_exit(irq, action, res);
        } else {
                unsigned int cpu = smp_processor_id();
                bool enabled = cpumask_test_cpu(cpu, desc->percpu_enabled);

                if (enabled)
                        irq_percpu_disable(desc, cpu);

                pr_err_once("Spurious%s percpu IRQ%u on CPU%u\n",
                            enabled ? " and unmasked" : "", irq, cpu);
        }

        if (chip->irq_eoi)
                chip->irq_eoi(&desc->irq_data);
}
```

**四、总结**

PPI中断和普通中断的比较

在硬件结构上，一个普通SPI中断号只有一个中断源，而一个PPI硬件中断号有多个中断源，每个CPU核对应一个中断源，这些中断源设备都是相同只是面向不同设备。

在linux中的使用上，区别也不大，只是注册中断和中断处理函数不同，而其中核心的不同是中断处理函数中传入的参数是per\-cpu变量，每个CPU核处理起来都使用不同的参数或设备指针。
