# 性能优化之loadavg

```
lilong@lilong:~$ cat /proc/loadavg
0.22 0.75 0.40         1/306             2383
                nr_runnig/nr_threads     
```

```
static int loadavg_proc_show(struct seq_file *m, void *v)
{
        unsigned long avnrun[3];

        get_avenrun(avnrun, FIXED_1/200, 0);

        seq_printf(m, "%lu.%02lu %lu.%02lu %lu.%02lu %ld/%d %d\n",
                LOAD_INT(avnrun[0]), LOAD_FRAC(avnrun[0]),
                LOAD_INT(avnrun[1]), LOAD_FRAC(avnrun[1]),
                LOAD_INT(avnrun[2]), LOAD_FRAC(avnrun[2]),
                nr_running(), nr_threads,
                task_active_pid_ns(current)->last_pid);
        return 0;
}

void get_avenrun(unsigned long *loads, unsigned long offset, int shift)
{
        loads[0] = (avenrun[0] + offset) << shift;
        loads[1] = (avenrun[1] + offset) << shift;
        loads[2] = (avenrun[2] + offset) << shift;
}

* The global load average is an exponentially decaying average of nr_running +
* nr_uninterruptible.
*
* Once every LOAD_FREQ:
*
*   nr_active = 0;
*   for_each_possible_cpu(cpu)
*      nr_active += cpu_of(cpu)->nr_running + cpu_of(cpu)->nr_uninterruptible;
*
*   avenrun[n] = avenrun[0] * exp_n + nr_active * (1 - exp_n)
```

参考资料：

[https://www.cnblogs.com/qqmomery/p/6267429.html](https://www.cnblogs.com/qqmomery/p/6267429.html)

[https://kernel.taobao.org/2017/10/Linux\-Load\-Averages\-Solving\-the\-Mystery/](https://kernel.taobao.org/2017/10/Linux-Load-Averages-Solving-the-Mystery/)
