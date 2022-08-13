# 调试技术之tracing文件说明

**1. current\_tracer：**

这用于设置或显示配置的当前跟踪器。 更改当前跟踪器会清除环形缓冲区内容以及“快照”缓冲区。比如 "nop" 这个tracer 在 trace\_nop.c 中通过 struct tracer nop\_trace 结构描述。

**2. available\_tracers:**

这包含已编译到内核中的不同类型的跟踪器。可以通过将其名子 echo 到 current\_tracer 中来配置选中此处列出的跟踪器。由于 available\_tracers 为nop，所有跟踪器功能无法使用。

**3. tracing\_on:**

这将设置或显示是否启用写入trace环缓冲区。 在此文件中写 0 以禁用跟踪器或写 1 以启用它。 请注意，这只是禁用对环形缓冲区的写入，跟踪开销可能仍在发生。

内核函数 tracing\_off\(\) 可以在内核中使用来禁用对环形缓冲区的写入，这会将这个文件设置为“0”。 用户空间可以通过在文件中写“1”来重新启用跟踪。

请注意，函数和事件触发器 “traceoff” 也会将此文件设置为零并停止跟踪。 用户空间也可以使用此文件重新启用。

**4. race:**

该文件以人类可读的格式保存跟踪的输出（如下所述）。 使用 O\_TRUNC 标志打开此文件进行写入会清除环形缓冲区内容。 请注意，此文件不是消费者。

如果跟踪关闭（没有跟踪器运行，或者 tracing\_on 为零），则每次读取时都会产生相同的输出。 当跟踪打开时，它可能会产生不一致的结果，因为它试图读取整个缓冲区而不消耗它。

**5. trace\_pipe:**

输出与 "trace" 文件相同，但此文件旨在通过实时跟踪进行流式传输。 从此文件读取将阻塞，直到检索到新数据。 与 "trace" 文件不同，此文件是消费者。

这意味着从该文件读取会导致顺序读取显示更多当前数据。 一旦从这个文件中读取数据，它就会被消耗掉，并且不会通过顺序读取再次读取。 "trace" 文件是静态的，

如果跟踪器没有添加更多数据，则每次读取时都会显示相同的信息。

**6. trace\_options:**

该文件允许用户控制在输出上述文件中之一的数据量。 还可以选择修改跟踪器或事件的工作方式（堆栈跟踪、时间戳等）。

补充：echo 1 \> options/hex 和 echo hex \> trace\_options 是一样的效果，可有通过 echo nohex \> trace\_options 取消16进制显示。

**7. options:**

这是一个目录，其中包含每个可用跟踪选项的文件（也在 trace\_options 中）。 也可以通过将“1”或“0”分别写入具有选项名称的相应文件来设置或清除选项。

8. tracing\_max\_latency:

一些跟踪器记录最大延迟。 例如，禁用中断的最长时间。 最长时间保存在此文件中。 最大跟踪也将被存储，并通过 "trace" 文件显示。 只有当延迟大于此文件中的值（以微秒为

单位）时，才会记录新的最大延迟trace。

通过echo一个时间到此文件中，不会记录其它任何延迟，除非它大于此文件中的时间。

9. tracing\_thresh:

每当延迟大于此文件中的数字时，某些延迟跟踪器将记录跟踪。 仅当文件包含大于 0 的数字时才有效。（以微秒为单位）

10. buffer\_size\_kb:

这将设置或显示每个 CPU 缓冲区保存的千字节数\(注意是每个\) \#\#\#\#\#。 默认情况下，每个 CPU 的跟踪缓冲区大小相同。 显示的数字是 CPU 缓冲区的大小，而不是所有缓冲区的总大

小。 跟踪缓冲区按页（内核用于分配的内存块，大小通常为 4KB）进行分配。 可以分配一些额外的页面来容纳缓冲区管理元数据。 如果分配的最后一个页面的空间比请求的字节多，

则将使用页面的其余部分，使实际分配大于请求或显示的空间。 （注意，由于缓冲区管理元数据，大小可能不是页面大小的倍数。）

各个 CPU 的缓冲区大小可能会有所不同（请参阅下面的“per\_cpu/cpu0/buffer\_size\_kb”），如果这样做，此文件将显示“X”。

11. buffer\_total\_size\_kb:

这将显示所有\(CPU的\)跟踪缓冲区的总大小。

12. free\_buffer:

如果一个进程正在执行跟踪，并且在进程完成时应该收缩“freed”环形缓冲区，即使它被信号杀死，该文件也可以用于该目的。 关闭此文件时，环形缓冲区将调整到其最小大小。

让一个正在跟踪的进程也打开这个文件，当进程退出时，这个文件的文件描述符将被关闭，这样做时，环形缓冲区将被“释放”。

如果设置了 disable\_on\_free 选项，它也可能会停止跟踪。

13. trace\_cpumask:

这是一个让用户只跟踪指定 CPU 的掩码。 格式是表示 CPU 的十六进制字符串。echo ff 或 01 分别表示跟踪全部CPU或只CPU0。

14. set\_ftrace\_filter:

在配置动态 ftrace 时（请参阅下面的"dynamic ftrace"部分），代码将被动态修改（code text 重写）以禁用函数分析器 \(mcount\) 的调用。 这允许在几乎没有性能开销的情况下配

置跟踪。 这也具有启用或禁用要跟踪的特定功能的副作用。 将函数的名称echo到此文件中会将跟踪限制为仅这些函数。 这会影响跟踪器“function”和“function\_graph”，因此也会影

响函数分析（参见“function\_profile\_enabled”）。

15. set\_ftrace\_notrace:

这与 set\_ftrace\_filter 的效果相反。 在此添加的任何功能都不会被追踪。 如果 set\_ftrace\_filter 和 set\_ftrace\_notrace 中都存在一个函数，则该函数将\_不\_被跟踪。

16. set\_ftrace\_pid:

让函数跟踪器只跟踪其 PID 列在此文件中的线程。

如果设置了“function\-fork”选项，那么当一个PID列在这个文件中的任务fork时，子进程的PID会自动添加到这个文件中，子进程也会被函数跟踪器跟踪。 此选项还将导致退出的任务

的 PID 从文件中删除。

17. set\_ftrace\_notrace\_pid:

让函数跟踪器忽略其 PID 列在此文件中的线程。

如果设置了“function\-fork”选项，那么当一个PID列在这个文件中的任务fork时，子进程的PID会自动添加到这个文件中，子进程也不会被函数跟踪器跟踪。 此选项还将导致退出的任务

的 PID 从文件中删除。

如果这个文件和“set\_ftrace\_pid”中都有一个PID，那么这个文件优先，线程不会被跟踪。

18. set\_event\_pid:

让事件仅跟踪具有此文件中列出的 PID 的任务。 请注意， sched\_switch 和 sched\_wake\_up 还将跟踪此文件中列出的事件。

要在 fork 上添加带有此文件中的 PID 的任务子项的 PID，请启用“event\-fork”选项。 该选项还将导致在任务退出时从该文件中删除任务的 PID。

19. set\_event\_notrace\_pid:

让事件不使用此文件中列出的 PID 跟踪任务。 请注意，如果 sched\_switch 或 sched\_wakeup 事件也跟踪应跟踪的线程，则 sched\_switch 和 sched\_wakeup 将跟踪未在此文件中列出

的线程，即使线程的 PID 在文件中。

要在 fork 上添加带有此文件中的 PID 的任务子项的 PID，请启用“event\-fork”选项。 该选项还将导致在任务退出时从该文件中删除任务的 PID。

20. set\_graph\_function:

此文件中列出的函数将导致函数图跟踪器仅跟踪这些函数及其调用的函数。 （有关更多详细信息，请参阅"dynamic ftrace"部分）。 请注意， set\_ftrace\_filter 和

set\_ftrace\_notrace 仍然会影响正在跟踪的函数。

21. set\_graph\_notrace：

类似于 set\_graph\_function，但会在函数被命中时禁用函数图跟踪，直到它退出函数。 这使得可以忽略由特定函数调用的跟踪函数。

22. available\_filter\_functions:

这列出了 ftrace 已处理并可跟踪的函数。 这些是您可以传递给“set\_ftrace\_filter”、“set\_ftrace\_notrace”、“set\_graph\_function”或“set\_graph\_notrace”的函数名称。

（有关更多详细信息，请参阅下面的"dynamic ftrace"部分。）

23. dyn\_ftrace\_total\_info:

此文件用于调试目的。 已转换为 nops 函数是可用于跟踪的。

**24. enabled\_functions:**

此文件更多用于调试 ftrace，但也可用于查看是否有任何函数附加了回调。 不仅跟踪基础设施使用 ftrace 函数跟踪实用程序，而且其他子系统也可能使用。 此文件显示所有附加了回调的函数以及已附加的回调数。 请注意，一个回调也可能调用多个函数，这些函数不会在此计数中列出。

如果回调注册为由具有“save regs”属性的函数跟踪（因此开销更大），则“R”将显示在与返回寄存器的函数相同的行上。

如果回调注册为由具有“ip modify”属性的函数跟踪（因此可以更改 regs\-\>ip），则“I”将显示在与可以覆盖的函数相同的行上。

如果架构支持它，它还会显示函数直接调用的回调。 如果计数大于 1，则很可能是 ftrace\_ops\_list\_func\(\)。

如果函数的回调跳转到特定于回调跳点，则将打印其地址以及跳点调用的函数。

例1：设置current\_tracer为nop时的回调函数

```
alarm_timer_remaining (1)       tramp: 0xffffffffc0bb2000 (function_trace_call+0x0/0x150) ->function_trace_call+0x0/0x150
```

例2：设置current\_tracer为function时的回调函数

```
hrtimers_resume (2)     ->ftrace_ops_list_func+0x0/0x190
```

25. function\_profile\_enabled：

设置后，它将使用函数跟踪器或函数图跟踪器（如果已配置）启用所有功能。 它将保留被调用函数数量的直方图，如果配置了函数图跟踪器，它还将跟踪在这些函数中花费的时间。

直方图内容可以显示在以下文件中：

trace\_stat/function\<cpu\>（function0、function1 等）。

26. trace\_stat:

保存不同跟踪统计信息的目录。

27. kprobe\_events：

启用动态跟踪点。 参见 kprobetrace.rst。

28. kprobe\_profile:

动态跟踪点统计信息。 参见 kprobetrace.rst。

29. max\_graph\_depth：

与函数图跟踪器一起使用。 这是它将追踪到函数的最大深度。 将此设置为值 1 将仅显示从用户空间调用的第一个内核函数。

30. printk\_formats:

这是用于读取原始格式文件的工具。 如果环形缓冲区中的事件引用了一个字符串，则只会将指向该字符串的指针记录到缓冲区中，而不是字符串本身。 这可以防止工具知道该字符串是

什么。 该文件显示字符串的字符串和地址，允许工具将指针映射到字符串的内容。例如 /rcu/tree.c:trace\_rcu\_utilization\(TPS\("Start scheduler\-tick"\)\); 中的 "Start scheduler\-tick"

将会显示在该文件中。

31. saved\_cmdlines:

只有任务的 pid 会记录在跟踪事件中，除非该事件还专门保存了任务的comm。 Ftrace 将 pid 映射缓存到comm以尝试显示事件的comm。 如果未列出通信的 pid，则输出中将显示“\<...\>”。\#\#\#\#\#\#\#\#\#\#\#\#\#\#\#

如果选项“record\-cmd”设置为“0”，则在记录过程中不会保存任务的comm。 默认情况下，它是启用的。

32. saved\_cmdlines\_size:

默认情况下，会保存 128 条 comms（请参阅上面的“saved\_cmdlines”）。 要增加或减少缓存的comm数量，请将要缓存的comm的数量echo到此文件中。

33. saved\_tgids:

如果设置了“record\-tgid”选项，则在每次调度上下文切换时，任务的任务组 ID 将保存在将线程的 PID 映射到其 TGID 的表中。 默认情况下，禁用“record\-tgid”选项。

34. snapshot:

这将显示"snapshot"缓冲区，并且还允许用户获取当前运行跟踪的快照。 有关更多详细信息，请参阅下面的"Snapshot"部分。

35. stack\_max\_size:

当堆栈跟踪器被激活时，这将显示它遇到的最大堆栈大小。 请参阅下面的"Stack Trace"部分。

36. stack\_trace:

这将显示激活堆栈跟踪器时遇到的最大堆栈的堆栈回溯跟踪。 请参阅下面的"Stack Trace"部分。

37. stack\_trace\_filter:

这类似于“set\_ftrace\_filter”，但它限制了堆栈跟踪器将检查的功能。

**38. trace\_clock:**

每当将事件记录到环形缓冲区时，都会添加一个“时间戳”。 此戳记来自指定的时钟。 默认情况下，ftrace 使用"local"时钟。 这个时钟非常快，并且严格按照每cpu计算，但在某些

系统上，相对于其他 CPU，它可能不是单调的。 换句话说，本地时钟可能与其他 CPU 上的本地时钟不同步。

用于跟踪的常用时钟::

/sys/kernel/tracing \# cat trace\_clock

local global counter uptime perf mono mono\_raw \[boot\]

带有方括号的时钟是有效时钟，就是当前选中的。

\(1\) local: 默认时钟，但可能不会跨 CPU 同步

\(2\) global: 该时钟与所有 CPU 同步，但可能比本地时钟慢一些。

\(3\) counter: 这根本不是一个时钟，而是一个原子计数器。它会一一计数，但与所有 CPU 同步。当您需要确切地知道在不同 CPU 上彼此发生的顺序事件时，这很有用。\#\#\#\#\#\#\#\#\#\#

\(4\) uptime: 这使用 jiffies 计数器，时间戳是相对于启动时间的。

\(5\) perf: 这使得 ftrace 使用与 perf 相同的时钟。最终 perf 将能够读取 ftrace 缓冲区，这将在交错数据中有帮助。

\(6\) x86\-tsc：架构可以定义自己的时钟。例如，x86 在这里使用自己的 TSC 周期时钟。

\(7\) ppc\-tb：这使用 powerpc 时基寄存器值。这是跨 CPU 同步的，如果 tb\_offset 已知，也可用于关联管理程序/访客之间的事件。

\(8\) mono: 这使用快速单调时钟 \(CLOCK\_MONOTONIC\)，它是单调的并且受 NTP 速率调整的影响。

\(9\) mono\_raw：这是原始单调时钟 \(CLOCK\_MONOTONIC\_RAW\)，它是单调的，但不受任何速率调整的影响，并且以与硬件时钟源相同的速率滴答。

\(10\) boot: 这是启动时钟 \(CLOCK\_BOOTTIME\)，基于快速单调时钟，但也考虑了挂起所花费的时间。 由于时钟访问被设计用于挂起路径中的跟踪，如果在更新快速单声道时钟之前考虑挂起时

间之后访问时钟，则可能会产生一些副作用。 在这种情况下，时钟更新似乎比正常情况发生得稍早。 同样在 32 位系统上，64 位引导偏移可能会看到部分更新。 这些效果很少见，后期处理应该能够处理它们。 有关更多信息，请参阅ktime\_get\_boot\_fast\_ns\(\) 函数中的注释。

要设置时钟，只需将时钟名称回显到此文件中：

\# echo global \> trace\_clock

设置时钟会清除环形缓冲区内容以及"snapshot"缓冲区。

39. trace\_marker:

这是一个非常有用的文件，用于将用户空间与内核中发生的事件同步。 将字符串写入此文件将写入 ftrace 缓冲区。

在应用程序开始时打开这个文件并只引用文件的文件描述符在应用程序中很有用：

```
void trace_write(const char *fmt, ...)
{
    va_list ap;
    char buf[256];
    int n;

    if (trace_fd < 0)
        return;

    va_start(ap, fmt);
    n = vsnprintf(buf, 256, fmt, ap);
    va_end(ap);

    write(trace_fd, buf, n);
}

start::
trace_fd = open("trace_marker", WR_ONLY);
```

注意：写入 trace\_marker 文件也可以启动写入 /sys/kernel/tracing/events/ftrace/print/trigger 的触发器

请参阅 Documentation/trace/events.rst 中的 "Event triggers" 和 Documentation/trace/histogram.rst 中的示例（第 3 节）

40. trace\_marker\_raw:

这与上面的 trace\_marker 类似，但用于写入二进制数据，其中可以使用工具解析来自 trace\_pipe\_raw 的数据。

41. upprobe\_events：

在程序中添加动态跟踪点。 参见 uprobetracer.rst

42. uprobe\_profile:

Uprobe 统计。 参见 uprobetrace.txt

43. instances:

这是一种制作多个跟踪缓冲区的方法，其中不同的事件可以记录在不同的缓冲区中。 请参阅下面的 "Instances" 部分。

44. events:

这是跟踪事件目录。 它保存已编译到内核中的事件跟踪点（也称为静态跟踪点）。 它显示了存在哪些事件跟踪点以及它们如何按系统分组。 不同级别的 "enable" 文件可以在向跟踪

点写入“1”时启用跟踪点。也就是说向 events 根目录下的enable写1就是使能所有的event，向i2c子目录下写1就是使能i2c相关的所有trace event。

有关更多信息，请参阅 events.rst。

45. set\_event:

通过将event echo到此文件中，将启用该事件。 有关更多信息，请参阅 events.rst。

46. available\_events:

可以在tracing中启用的事件列表。有关更多信息，请参阅 events.rst。

47. timestamp\_mode:

某些跟踪器可能会更改将跟踪事件记录到事件缓冲区时使用的时间戳模式。 具有不同模式的事件可以在缓冲区中共存，但记录事件时生效的模式决定了该事件使用哪种时间戳模式。

默认时间戳模式是“delta”。

用于跟踪的常用时间戳模式：

\# cat timestamp\_mode

\[delta\] absolute

带有方括号的时间戳模式是设置为有效的模式。

delta：默认时间戳模式 \- 时间戳是针对每个缓冲区时间戳的增量。

absolute：时间戳是一个完整的时间戳，而不是相对于其他值的增量。 因此，它占用更多空间并且效率较低。

48. hwlat\_detector:

硬件延迟检测器的目录。 请参阅下面的“Hardware Latency Detector”部分。

49. per\_cpu：

这是一个包含跟踪 per\_cpu 信息的目录。

per\_cpu/cpu0/buffer\_size\_kb：

ftrace 缓冲区定义为 per\_cpu。 也就是说，每个 CPU 都有一个单独的缓冲区，允许以原子方式进行写入，并且不会出现cache bouncing。 这些缓冲区可能有不同大小的缓冲区。

该文件类似于 buffer\_size\_kb 文件，但它仅显示或设置特定 CPU 的缓冲区大小。 （这里是 cpu0）。

50. per\_cpu/cpu0/trace:

这类似于“trace”文件，但它只会显示特定于 CPU 的数据。 如果写入，它只会清除特定的 CPU 缓冲区。

51. per\_cpu/cpu0/trace\_pipe

这类似于“trace\_pipe”文件，并且是消耗性读取，但它只会显示（和消耗）特定于 CPU 的数据。

52. per\_cpu/cpu0/trace\_pipe\_raw

对于可以解析ftrace ring buffer二进制格式的工具，可以直接使用trace\_pipe\_raw文件从ring buffer中提取数据。 通过使用 splice\(\) 系统调用，可以将缓冲区数据快速传输到文

件或服务器正在收集数据的网络。和 trace\_pipe 一样，这是一个消费读取器，多次读取总是会产生不同的数据。

53. per\_cpu/cpu0/snapshot:

这类似于主“snapshot”文件，但只会对当前 CPU 进行快照（如果支持）。 它只显示给定 CPU 的快照内容，如果写入，只清除该 CPU 缓冲区。

54. per\_cpu/cpu0/snapshot\_raw：

与 trace\_pipe\_raw 类似，但将从给定 CPU 的快照缓冲区读取二进制格式。

55. per\_cpu/cpu0/stats:

这将显示有关环形缓冲区的某些统计信息：

entries: 仍在缓冲区中的事件数。

overrun: 缓冲区已满时由于覆盖而丢失的事件数。

commit overrun: 应该始终为零。 如果在嵌套事件中发生了太多事件（环形缓冲区是可重入的），它会被设置，它会填充缓冲区并开始丢弃事件。

bytes: 实际读取的字节数（未覆盖）。

oldest event ts: 缓冲区中最旧的时间戳

now ts: 当前时间戳

dropped events: 由于覆盖选项关闭而丢失的事件。

read events: 读取的事件数。
