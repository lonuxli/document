# Linux基础之ARM GCC 内嵌汇编

参考：[http://blog.chinaunix.net/uid\-20706279\-id\-1888741.html](http://blog.chinaunix.net/uid-20706279-id-1888741.html)  ARM GCC内嵌汇编代码翻译版

分析ARM GCC内嵌汇编代码，以spin\_lock示例分析：

```
  8 #define TICKET_SHIFT▼   16
  9
10 typedef struct {
11 ▼       union {
12 ▼       ▼       u32 slock;
13 ▼       ▼       struct __raw_tickets {
18 ▼       ▼       ▼       u16 owner;
19 ▼       ▼       ▼       u16 next;
21 ▼       ▼       } tickets;
22 ▼       };
23 } arch_spinlock_t;   

73 static inline void arch_spin_lock(arch_spinlock_t *lock)
74 {
75 ▼       unsigned long tmp;
76 ▼       u32 newval;
77 ▼       arch_spinlock_t lockval;
78
79 ▼       prefetchw(&lock->slock);
80 ▼       __asm__ __volatile__(      //__volatile__ 表示指示编译器对该嵌入汇编代码不进行代码优化，修改指令顺序等，__asm__ __volatile__可以用asm和volatile，为了避免和C关键字重复警告，最好加下划线
81 "1:▼    ldrex▼  %0, [%3]\n"        //%0 %1 %2 %3 %4这些都是表示操作符（寄存器或立即数等），具体与哪个关联与output list 和inputlist 中列表元素顺序有关，分别顺序0-4的变量
82 "▼      add▼    %1, %0, %4\n"      //汇编代码段中换行符和制表符的使用可以使得指令列表看起来变得美观，增强汇编之后的代码可读性
83 "▼      strex▼  %2, %1, [%3]\n"
84 "▼      teq▼    %2, #0\n"
85 "▼      bne▼    1b"
86 ▼       : "=&r" (lockval), "=&r" (newval), "=&r" (tmp)  //汇编输出列表，可以修改C代码中的局部变量
87 ▼       : "r" (&lock->slock), "I" (1 << TICKET_SHIFT)   // 汇编输入列表，使用的是函数入参如指针地址和立即数
88 ▼       : "cc");
89
90 ▼       while (lockval.tickets.next != lockval.tickets.owner) {
91 ▼       ▼       wfe();
92 ▼       ▼       lockval.tickets.owner = ACCESS_ONCE(lock->tickets.owner);
93 ▼       }
94
95 ▼       smp_mb();
96 }
```

1、嵌入汇编格式

    asm\(code : output operand list : input operand list : clobber list\);

     output operand list  输出操作符列表

     input operand list     输入操作符列表

     clobber list    破坏符列表  目的是告诉编译器列表中被修改过

2、限制符

|**Modifier**|**Specifies**                                                                                                                         |
|------------|--------------------------------------------------------------------------------------------------------------------------------------|
|=           |Write\-only operand, usually used for all output operands   为何是只写属性，寄存器不能用作传递入参到嵌入汇编吧？                      |
|\+          |Read\-write operand, must be listed as an output operand                                                                              |
|&           |A register that should be used for output only  通知编译器该操作符使用的寄存器作为输出，即运行完嵌入汇编代码之后寄存器值是需要使用的？|

|**Constraint**|**Usage in ARM state**                                                        |**Usage in Thumb state**                                                            |
|--------------|------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
|f             |Floating point registers f0 .. f7                                             |Not available                                                                       |
|h             |Not available                                                                 |Registers r8..r15                                                                   |
|G             |Immediate floating point constant                                             |Not available                                                                       |
|H             |Same a G, but negated                                                         |Not available                                                                       |
|I             |e.g. ORR R0, R0, \#operand
Immediate value in data processing instructions     |e.g. SWI operand
Constant in the range 0 .. 255                                      |
|J             |e.g. LDR R1, \[PC, \#operand\]
Indexing constants \-4095 .. 4095               |e.g. SUB R0, R0, \#operand
Constant in the range \-255 .. \-1                        |
|K             |Same as I, but inverted                                                       |Same as I, but shifted                                                              |
|L             |Same as I, but negated                                                        |e.g. SUB R0, R1, \#operand
Constant in the range \-7 .. 7                            |
|l             |Same as r                                                                     |e.g. PUSH operand
Registers r0..r7                                                   |
|M             |e.g. MOV R2, R1, ROR \#operand
Constant in the range of 0 .. 32 or a power of 2|e.g. ADD R0, SP, \#operand
Constant that is a multiple of 4 in the range of 0 .. 1020|
|m             |Any valid memory address                                                      |                                                                                    |
|N             |Not available                                                                 |e.g. LSL R0, R1, \#operand
Constant in the range of 0 .. 31                          |
|O             |Not available                                                                 |e.g. ADD SP, \#operand
Constant that is a multiple of 4 in the range of \-508 .. 508 |
|r             |e.g. SUB operand1, operand2, operand3
General register r0 .. r15               |Not available                                                                       |
|w             |Vector floating point registers s0 .. s31                                     |Not available                                                                       |
|X             |Any operand                                                                   |                                                                                    |
