# 调试技术之eventtracing 的实现原理

`tracepoint` `event tracing`

event tracing 出现已久 , 大家可能对这个名字有些陌生，由于它是 ftrace 的一部分，于是经常被 ftrace 的光芒掩盖，而没有引起注意。内核代码中本身包含一些关于 ftrace 的文档，存放于 Document/trace 下，其中也包含一些对 event tracing 的介绍，它们是了解 ftrace 最好的资料，但这些文档还是有些少，很难面面俱到。本文所介绍的 event tracing，使用了大量的宏来产生代码，这使得代码写起来非常简洁，只需定义一个宏就可以，编译器在宏处理时会自动把它们展开成需要的代码，这就使得本来较为简单的实现被复杂的宏替换所掩盖。本文力图从这些宏出发，揭露 event tracing 是如何实现的。

开头部分对 ftrace 进行简单介绍，之后先介绍 tracepoint 的实现（event tracing 的实现依赖于 tracepoint），再介绍 event tracing 的实现。

## ftrace 简介

在详细介绍 event tracing 之前之前我们先对 ftrace 有个整体的认识，ftrace 的名字由 function trace 而来。function trace 是利用 gcc 编译器在编译时在每个函数的入口地址放置一个 probe 点，这个 probe 点会调用一个 probe 函数（gcc 默认调用名为 mcount 的函数），这样这个 probe 函数会对每个执行的内核函数进行跟踪（其实有少数几个内核函数不会被跟踪），并打印 log 到一个内核中的环形缓存（ring buffer）中，而用户可以通过 debugfs 来访问这个环形缓存中的内容。function trace 的框架如图 1 所示。

##### 图 1. function trace 原理框图

![da83c79b2faf0ddf988394a26c933b1a.png](image/da83c79b2faf0ddf988394a26c933b1a.png)

其它一些 trace 工具也具有相似的工作流程，需要一个 probe 点，一个 probe 函数，一种保存 log 信息的机制，还有用于户进行交互的接口（分别对应图中 A、B、C、D 点）。它们发现可以利用 function trace 的后两部分，也就是环形缓存的管理和用户交互接口部分的代码，只需要实现自己的 probe 点和 probe 函数就可以了。于是在 function trace 进入内核 mainline 后，kprobe 也借用了这样的机制。

kprobe 是很早前就存在于内核中的一种动态 trace 工具。kprobe 本身利用了 int 3（在 x86 中）实现了 probe 点，这对应于 A 部分的功能。使用 kprobe 需要用户自己实现 kernel module 来注册 probe 函数。可以看出 kprobe 并没有统一的 B、C 和 D。使用起来用户需要自己实现很多东西。不是很灵活。而在 function trace 出现后，kprobe 借用了它的一部分设计模式，实现了统一的 probe 函数（对应于图中的 B），并利用了 function trace 的环形缓存和用户接口部分，也就是 C 和 D 部分功能，用户可以使用读写 debugfs 中相关文件就可以控制 kprobe 的注册注销以及读取结果，非常方便。

之后很多其它 trace 工具也借用了这个内核中的 ring buffer 来输出打印的信息，并让用户通过 debugfs 来读取这个 ring buffer 中的信息，debugfs 会根据不同的 trace 选用不同的文件操作函数，来将 buffer 中的二进制数据输出为 ascii 字符供用户分析。ftrace 已经不再单单是最开始的 function trace 了，它已经变成了一个 trace 的框架，或者是一大类相似的 trace 工具了。这些 trace 工具在 ftrace 中叫做 tracer，包括 latency tracer 等很多，用户可以 cat /sys/kernel/debug/tracing/avaiable\_tracer 查看系统支持的 tracer。

我们文章的主题 event tracing（又叫 trace event）也是使用这一套标准流程的，也是 ftrace 中的一个重要的组成部分。这个子类型区别于其它的 tracer 的本质区别就是它有不同的管理 probe 点和 probe 函数的机制。而后面使用相同的 ring buffer 和 debugfs 作为接口。这种管理 probe 点和 probe 函数的机制就是 tracepoint。事实上，tracepoint 是早就存在于内核当中的一种 trace 工具。它的出现要比 ftrace 早很多。event tracing 可以理解为 tracepoint 加 function trace 的接口部门组合而成。如上图所示，tracepoint 提供图中的 probe 点（图中 A）和 probe 函数（图中 B）这两部分。function trace 提供 ring buffer 的管理（图中 C）以及 debugfs 相关的接口（图中 D）。内核中的很多子系统都定义了 event tracing。用户可以直接使用 debugfs 来控制它们进行分析。

## Tracepoint 的实现原理

### 示例

我们先看一个示例，这个示例来自内核中的一个示例，代码存在内核源码 sample/tracepoints/ 中，包含如下文件。（这个示例在 3.9 版本中被移出 kernel，原因是不期望开发人员直接使用 tracepoint，而应该使用更高层的 event tracing。我们可以使用小于 3.9 的版本的内核来学习这个示例）。

- tp\-samples\-trace.h, 定义了 tracepoint 使用的函数原型；
- tracepoint\-sample.c, 定义 probe 点的内核代码；
- tracepoint\-probe\-sample.c, 注册 probe 函数；

要编译这个示例，必须在编译内核的时候选中 CONFIG\_SAMPLES 和 CONFIG\_SAMPLE\_TRACEPOINTS， 这样示例代码会随内核一起生成。示例代码被编译为内核模块，将生成的示例代码模块安装。安装模块后会在 /proc 下创建一个普通文件 tracepoint\-sample，cat 这个文件就会触发预先定义的 probe 函数，下面是 probe 输出的 log，可以通过 dmesg 查看。相关命令见清单 1.

#### 清单 1. 示例输出

|2

3

4

5

6

7

8
1|cat: /proc/tracepoint\-sample: Operation not permitted

$dmesg

\[ 5797.665342\] sample init

\[ 5813.192993\] Event is encountered with filename tracepoint\-sample

\[ 5813.228900\] Event B is encountered

\[ 5813.249203\] Event B is encountered

...
$cat /proc/tracepoint\-sample|
|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

tracepoint\-sample.c 中定义了 probe 点。并在 /proc 目录下创建了一个 /proc/tracepoint\-sample 文件，

当这个文件被打开时，也就是调用 open 系统调用时，tracepoint\-probe\-sample.c 中注册的 probe 函数被调用，并用 printk 打印信息。借助 cat 命令执行 open，cat /proc/tracepoint\-sample 将会调用这个文件的 open 回调函数，进而触发 probe 函数。这个示例简单明了，注意 cat 返回错误信息，这是由于示例中并未实现真正的 open 函数，只是返回一个错误信息，这是正常的。

### 实现

大部分工作有两个宏完成，一个是 DECLARE\_TRACE，在 tp\-samples\-trace.h 中，见清单 2。

#### 清单 2. 声明 tracepoint

|2

3
1|      TP\_PROTO\(struct inode \*inode, struct file \*file\),

      TP\_ARGS\(inode, file\)\);
DECLARE\_TRACE\(subsys\_event,|
|---|--------------------------------------------------------------------------------------------------------------------------|

这个宏在 \<linux/tracepoint.h\> 被定义，清单 3 是它的定义，可以看出 DECLARE\_TRACE 在编译时会被展开成下面三个函数。这里 name 是 subsys\_event，\#\# 的作用是将它两边的符号合并成一个。reg/unregister\_trace\_subsys\_event 负责注册或注销 probe 函数，probe 函数的原型由 TP\_PROTO 宏给出，trace\_subsys\_event 被放置在要探测的点，也就是图 1 中的 A 处，它在运行时会检查当前的这个 tracepoint 上有没有已经注册了 probe 函数，如果没有注册，那么什么也不做，如果通过之前的注册函数注册过了，那么就调用这个 probe 函数，这是由 \_DO\_TRACE 这个宏完成调用的，具体可参考源码。DECLARE\_TRACE 还声明引用了一个类型为 struct tracepoint 的变量 \_\_tracepoint\_subsys\_event。这个就是 tracepoint 的核心数据结构，系统中所有的 tracepoint 被放在一个 hash 表里方便查找，reg/unregister 函数查找并改变这个结构。

#### 清单 3. 宏展开结果

|2

3

4

5

6

7

8

9

10

11

12

13

14

15

16
1|       extern struct tracepoint \_\_tracepoint\_\#\#name;                   \\

       static inline void trace\_\#\#name\(proto\)                          \\

       {                                                               \\

               if \(unlikely\(\_\_tracepoint\_\#\#name.state\)\)                \\

                       \_\_DO\_TRACE\(&\_\_tracepoint\_\#\#name,                \\

                               TP\_PROTO\(proto\), TP\_ARGS\(args\)\);        \\

       }                                                               \\

       static inline int register\_trace\_\#\#name\(void \(\*probe\)\(proto\)\)   \\

       {                                                               \\

               return tracepoint\_probe\_register\(\#name, \(void \*\)probe\); \\

       }                                                               \\

       static inline int unregister\_trace\_\#\#name\(void \(\*probe\)\(proto\)\) \\

       {                                                               \\

               return tracepoint\_probe\_unregister\(\#name, \(void \*\)probe\);\\

       }
\#define DECLARE\_TRACE\(name, proto, args\)                               \\|
|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

另一个宏是 DEFINE\_TRACE，在 tracepoint\-sample.c 中，见清单 4，这个宏也是在 \<linux/tracepoint.h\> 中定义，这个宏展开后的效果就是在这个 c 文件里定义了一个变量 struct tracepoint\_\_tracepoint\_subsys\_name。这个变量就是之前声明引用的那个变量的定义。这个变量被放在 \_\_tracepoints 的 section 里，这是这条语句的作用 \_\_attribute\_\_\(\(section\("\_\_tracepoints"\), aligned\(32\)\)\)。同时定义了一个字符串，这个字符串就是这个 tracepoint 的名字，在这里是 subsys\_event。

#### 清单 4. DEFINE\_TRACE 的定义

|2

3

4

5

6

7

8

9

10

11

12
1| 

\# 定义

\#define DEFINE\_TRACE\_FN\(name, reg, unreg\)                               \\

       static const char \_\_tpstrtab\_\#\#name\[\]                           \\

       \_\_attribute\_\_\(\(section\("\_\_tracepoints\_strings"\)\)\) = \#name;      \\

       struct tracepoint \_\_tracepoint\_\#\#name                           \\

       \_\_attribute\_\_\(\(section\("\_\_tracepoints"\), aligned\(32\)\)\) =        \\

               { \_\_tpstrtab\_\#\#name, 0, reg, unreg, NULL }

 

\#define DEFINE\_TRACE\(name\)                                              \\

       DEFINE\_TRACE\_FN\(name, NULL, NULL\);
DEFINE\_TRACE\(subsys\_event\);|
|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

通过这两个宏，我们只需要在想要探测的点上放置 trace\_subsys\_event 函数，并给出相应参数就可以了，参看清单 5，本示例中在 open 的的回调函数 my\_open 中放置了这个函数，注意这个函数就是在前面提到的 DECLARE\_TRACE 中定义的，代码中 trace\_subsys\_eventb 是定义的另一个 tracepoint。如果没有任何 probe 函数，只会消耗一些系统性能（判断是否有 probe 函数注册）。当需要使用这个 probe 点时，就调用注册函数，并实现 probe 函数。probe 函数的行为没有限制，取决于实现者。

#### 清单 5. 设置 trace 点

|2

3

4

5

6

7

8

9
1|{

       int i;

 

       trace\_subsys\_event\(inode, file\);

       for \(i = 0; i \< 10; i\+\+\)

               trace\_subsys\_eventb\(\);

       return \-EPERM;

}
static int my\_open\(struct inode \*inode, struct file \*file\)|
|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

使用 tracepoint 需要自己实现 probe 函数，而这个 probe 函数通常就是打印一些信息，每次都自己实现比较麻烦，如果要有一个统一的机制完成这块的功能就好了，而且实现 probe 函数通常需要用一个内核模块来进行，也是一个不方便的因素。于是，建立在 tracepoint 上的 trace event 出现了，它的 probe 函数使用 ftrace 的 ring buffer 记录输出，并且提供一个 debugfs 的接口给用户空间，极大的方便了使用。

## event tracing 的实现原理

### 示例

我们还是通过一个示例着手，内核目录中同样提供一个 trace event 的示例，位于 sample/trace\_events/，为保持简单，本示例使用的内核版本需低于 4.0。4.0 的内核对这个示例有较大的改变。要编译这个示例，需要选择 CONFIG\_SAMPLES 和 CONFIG\_SAMPLE\_TRACE\_EVENTS。编译后会生成相应的内核模块。由于 event tracing 使用了 debugfs 作为接口，使用示例前请检查是否挂载了 debugfs，一般 debugfs 挂载在 /sys/kernel/debug/ 上。之后安装这个示例的内核模块。安装后在 debugfs 下会发现多出如下文件和文件夹。相关命令见清单 6。

#### 清单 6. 安装示例模块

|2

3

4

5

6

7
1|$mount \-t debugfs none /sys/kernel/debug/

\# 安装示例模块

$modprobe trace\-events\-sample

$cd /sys/kernel/debug/

$ls events/sample/foo\_bar/

enable  filter  format  id
\# 挂载 debugfs|
|-------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

通过对 enable 文件写 1 或 0 来控制打开或关闭这个事件的 event tracing。使用清单 7 打开这个 event tracing。使用清单 8 命令读取结果，使用清单 9 命令关闭这个 event tracing。

#### 清单 7. 打开这个事件的 event tracing

|1|$echo 1 \> events/sample/enable|
|-|-------------------------------|

#### 清单 8. 读取结果

|2

3

4

5

6
1|\#           TASK\-PID    CPU\#    TIMESTAMP  FUNCTION

\#              | |       |          |         |

   event\-sample\-6469  \[001\]   516.660504: foo\_bar: foo hello 241

   event\-sample\-6469  \[001\]   517.660247: foo\_bar: foo hello 242

   event\-sample\-6469  \[001\]   518.659902: foo\_bar: foo hello 243
$cat trace|
|------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

#### 清单 9. 关闭这个事件的 event tracing

|1|$echo 0 \> events/sample/enable|
|-|-------------------------------|

示例实现了一个名为 foo\_bar 的 trace event。它属于名为 sample 的子系统。这种层级结构也表现在 debugfs 中的文件和目录的关系。sample 是目录，而 foo\_bar 是 sample 下的子目录。示例使用一个内核线程，在每次调度到这个内核线程时输出一条 log 信息。如清单 10，内核线程反复调用这个函数，trace\_foo\_bar 就是 probe 点，每次这个内核线程在 sleep 后都会调用这个 probe 点上的 probe 函数，并将"hello"和一个计数器 cnd 传递给 probe 函数。当然，如果没有注册就不会。probe 函数会将这两个参数记录在 ftrace 的环形缓存中，用户读取结果时，将环形缓存的内容格式化输出。完成整个过程。probe 函数可以通过 tracepoint 的机制来注册或者注销，因为它本身就是一个 tracepoint。

#### 清单 10. probe 点

|2

3

4

5

6
1|{

       set\_current\_state\(TASK\_INTERRUPTIBLE\);

       schedule\_timeout\(HZ\);

       trace\_foo\_bar\("hello", cnt\);

}
static void simple\_thread\_func\(int cnt\)|
|------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|

### 实现原理

event tracing 的实现是建立的 tracepoint 上的，如第一章所说，event tracing 的 probe 点和 probe 函数由 tracepoint 实现，所以 event tracing 肯定也许要实现一个 tracepoint，就像前文介绍的 tracepoint 的实现一样。并且 probe 函数需要调用 ftrace 的一些 API 来实现向 ftrace 的 ring buffer 内写数据，还需要用 debugfs 的接口实现控制。所以 event tracing 的实现从功能上也可以分为图 1 中 A、B、C、D 这四个部分。

由于内核各个子系统大量使用 event tracing 来 trace 不同的事件，每有一个需要 trace 的事件就实现这么一套函数，这样内核就会存在大量类似的重复的代码，为了避免这样的情况，内核开发者使用一个宏，让宏自动展开成具有相似性的代码。这个宏就是 TRACE\_EVENT，要为某个事件添加一个 trace event，只需要声明这样一个宏就可以了，下面我们看看这样的宏如何展开成具有 A、B、C、D 四部分功能的代码。

本示例声明如清单 11 这样的一个宏：

#### 清单 11. 一个宏实现 trace event

|2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18
1| 

       TP\_PROTO\(char \*foo, int bar\),

 

       TP\_ARGS\(foo, bar\),

 

       TP\_STRUCT\_\_entry\(

               \_\_array\(        char,   foo,    10        \)

               \_\_field\(        int,    bar              \)

       \),

 

       TP\_fast\_assign\(

               strncpy\(\_\_entry\-\>foo, foo, 10\);

               \_\_entry\-\>bar    = bar;

       \),

 

       TP\_printk\("foo %s %d", \_\_entry\-\>foo, \_\_entry\-\>bar\)

\);
TRACE\_EVENT\(foo\_bar,|
|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

TP\_PROTO 定义函数的原型。 也就是将来的 probe 函数带有两个参数，一个 char \*， 一个 int 类型。

TP\_ARGS 定义函数的参数，注意与上一个的区别，上面只是定义了函数的类型，它可以用来做类型转换。

TP\_STRUCT\_\_entry 定义记录在 ftrace 环形内存中数据的类型。它会被替换到定义一个结构体的变量 \_\_array 代表数组类型，\_\_field 是普通类型，它们分别等同于 char foo\[10\] int bar，我们会在后面看到它们怎么被替换到某个结构体中。

TP\_fast\_assign 说明将来在 probe 中这些 entry 会怎么记录在环形缓存中。

TP\_printk 当我们通过 debugfs 查看 trace 的结果是，环形缓存中的数据并不是直接以二进制显示的，而是结构化输出的。TP\_printk 会被宏替换到输出的函数中。

通过这一个定义，多次宏替换，生成需要进行 event tracing 的所有函数，如果要是手写的化，可能会产生很多很多重复的代码。这个宏就是实现 event tracing 的核心。所有的事情用这一个定义就可以完成，TRACE\_EVENT 被反复 \#define \#undef 用不同的代码模板制造出需要的代码。

第一次包含这个头文件，把 TRACE\_EVENT 定义成 tracepoint。这是在 include/tracepoint.h 中实现的，如清单 12.

#### 清单 12. 实现 tracepoint

|2
1|       DECLARE\_TRACE\(name, PARAMS\(proto\), PARAMS\(args\)\)
\#define TRACE\_EVENT\(name, proto, args, struct, assign, print\)   \\|
|--|------------------------------------------------------------------------------------------------------------------------------------|

TRACE\_EVENT 被定义成 DECLARE\_TRACE ，而 DECLARE\_TRACE 正是上一章介绍的实现一个 tracepoint 需要的两个宏当中的一个，这里可以看到 event\_tracing 确实是建立在 tracepoint 上的。这时我们有了这几个函数的定义。

- register\_trace\_foo\_bar
- unregister\_trace\_foo\_bar
- trace\_foo\_bar

这时，还没有注册 probe 函数，trace\_foo\_bar 本质上不做任何事情。

这个 TRACE\_EVENT 宏已经被替换了，内核接下来会再次包含这个头文件，然后重新定义这个宏，以便把这个宏再展开成其它的代码，一个头文件再包含自己，这样不是形成一个死循环了么，事实上内核开发者使用一些标志巧妙的避开了这样的错误。TRACE\_EVENT 在整个预编译过程中要被重新定义很多次，然后重新包含这个头文件，从而产成不同的需要的代码。这种宏虽然复杂，也不直观，但是它减少了代码的重复。试想，内核中也很多很多 event tracing，如果每一个子系统的定义自己的，那会产生多少类似的代码实现类似的功能。

第二次包含这个头文件，接下来会从新定义 TRACE\_EVENT，以便产生其它代码。这是在 include/trace/define\_trace.h，如清单 13.

#### 清单 13. 实现 tracepoint（续）

|2

3
1|\#define TRACE\_EVENT\(name, proto, args, tstruct, assign, print\)  \\

       DEFINE\_TRACE\(name\)
\#undef TRACE\_EVENT|
|---|----------------------------------------------------------------------------------------------------------------------|

这里先 undef 了 TRACE\_EVENT 的定义，然后又重行 define 它。这里 TRACE\_EVENT 又被定义成 DEFINE\_TRACE，而 DEFINE\_TRACE 是上一章说的另一个宏。通过这两次宏展开，一个完整的 tracepoint 已经被实现了。

接下来 TRACE\_EVENT 将致力于实现一个通用的 probe 函数。这个 probe 函数使用 ftrace 的环形缓存来输出数据，TRACE\_EVENT 还将实现 debugfs 的接口，以便交互和控制。

第三次第一这个宏在 include/trace/ftrace.h 里 , 它会产生一个记录在环形缓存中的数据类型，在这个示例中使用清单 14 中的数据类型。具体可见代码。前面提到的 TRACE\_EVENT 宏中的 TP\_STRUCT\_\_entry 被定义成了这个数据结构。第一个元素是固定存在每条 trace 的记录中的。其余 foo 和 bar 是自定义的两个数据类型。它们其实是 probe 函数的两个参数。 probe 函数会将这两个参数记录到环形缓存区中。

#### 清单 14. 记录在 ring buffer 中的数据类型

|2

3

4

5

6
1|       struct trace\_entry ent;

        char foo\[10\];

       int bar;

       char \_\_data\[0\];

};
struct ftrace\_raw\_foo\_bar {|
|------|-------------------------------------------------------------------------------------------------------------------------------|

接下来还会有接连几次从新定义这个宏，分别产生用于输出这条记录格式的相关函数，可以通过 debugfs 的 format 查看，我们不再关注，其中主要关注 probe 函数，也就是用于向环形缓存区写数据的函数，如清单 15，这个函数就是通过 tracepoint 注册的 probe 函数，用于将来被注册的 probe 函数，该函数在每次事件发生时记录相关的数据（foo, bar）到 ring buffer。这个函数是宏替换产生的，并不存在以内核代码中。可以通过 gcc 的 \-E 来产生这个代码片段，事实上前文提到的也都是通过 gcc 的 \-E 选项显示出来的。另一个比较重要的函数是读 debugfs 中的 trace 文件时候的回调函数。这个函数格式化环形缓存区中的输出。转化为用户易读取的格式。这个函数由 TP\_printk 而来。

#### 清单 15. probe 函数

|2

3

4

5

6

7

8

9

10
1|                                        char \*foo, int bar\)

{

 …

 entry = ring\_buffer\_event\_data\(event\);

 { strncpy\(entry\-\>foo, foo, 10\);

   entry\-\>bar = bar;; }

  f \(\!filter\_current\_check\_discard\(buffer, event\_call, entry, event\)\)

      trace\_nowake\_buffer\_unlock\_commit\(buffer, event, irq\_flags, pc\);

};
static void ftrace\_raw\_event\_id\_foo\_bar \(struct ftrace\_event\_call \*event\_call,|
|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

最后，这些实现的所有函数被记录在清单 16 中的数据结构中，这个结构供 debugfs 文件系统使用。用户的与 debugfs 的交互将根据这张表来找到相应的操作，如本章开头示例的操作是，先使能这个事件的 trace event，也就是清单 7 中的命令，将通过 odebugfs 调用 event\_foo\_bar.regfunc 这个函数，如清单 17，它将注册 ftrace\_raw\_event\_id\_foo\_bar 这个 probe 函数到 probe 点。然后关闭就会调用 event\_foo\_bar.unregfunc 注销函数。tprobe 函数会在每次被调用是输出记录到 ring buffer，读取 trace 结果存在 .event 字段中的子系统实现。这样整个 event traceing 的实现就清晰了。

#### 清单 16. 与 debugfs 交互数据结构

|2

3

4

5

6

7

8

9

10

11

12
1|       .name = "foo\_bar",

       .system = "sample",

       .event = &ftrace\_event\_type\_foo\_bar,

       .raw\_init = trace\_event\_raw\_init,

       .regfunc = ftrace\_raw\_reg\_event\_foo\_bar,

       .unregfunc = ftrace\_raw\_unreg\_event\_foo\_bar ,

       .show\_format = ftrace\_format\_foo\_bar ,

       .define\_fields = ftrace\_define\_fields\_foo\_bar ,

       .profile\_enable = ftrace\_profile\_enable\_foo\_bar,

       .profile\_disable = ftrace\_profile\_disable\_foo\_bar,

};
struct ftrace\_event\_call event\_foo\_bar = {|
|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

#### 清单 17. 注册注销函数

|2

3

4

5

6

7

8

9
1|{

       return register\_trace\_foo\_bar \( ftrace\_raw\_event\_foo\_bar \);

}

 

static void ftrace\_raw\_event\_foo\_bar \(char \*foo, int bar\)

{

       ftrace\_raw\_event\_id\_foo\_bar \(& event\_foo\_bar , foo, bar\);

}
static int ftrace\_raw\_reg\_event\_foo\_bar \(struct ftrace\_event\_call \*unused\)|
|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

include/trace 中利用这个宏定义了很多 trace event。它们都是非常有效且使用方便的调试工具。trace 系统博大精深，作者水平有限，不足之处望指正。目前 linux kernel 的最新的发展有望把 BPF 加入到 trace event 中。BPF 是 Berkeley Packet Filter 的缩写，BPF 实现了一个内核中的虚拟机，本来用来让内核快速的丢掉不需要的网络中的包，这一设计也适用于 trace event。trace event 在打开时会产生大量信息，将 BPF 加入到 trace event 中可以在第一时间过滤这些信息，只保存用户感兴趣的数据。
