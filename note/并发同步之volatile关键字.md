# 并发同步之volatile关键字

**一、Linux中volatile的两种类型**

Linux中使用的volatile关键字主要分为两类，一是C语言标准中定义的volatile，这种类型应用广；另一种是GCC嵌入汇编中volatile。

**二、C语言中volatile**

**2.1 GCC规范文档中volatile定义**

**2.1.1 gcc中对volatile的描述**

```
C has the concept of volatile objects. These are normally accessed by pointers and used for accessing hardware or inter-thread communication. The standard encourages compilers to refrain from optimizations concerning accesses to volatile objects, but leaves it implementation defined as to what constitutes a volatile access. The minimum requirement is that at a sequence point all previous accesses to volatile objects have stabilized and no subsequent accesses have occurred. Thus an implementation is free to reorder and combine volatile accesses that occur between sequence points, but cannot do so for accesses across a sequence point. The use of volatile does not allow you to violate the restriction on updating objects multiple times between two sequence points.
```

C语言中关键字volatile，定义了易变性的概念，<span style="background-color: #ffaaaa">它通常由指针访问，并用于访问硬件或线程间通信</span>（硬件内容自身的变化以及其他线程对变量的修改两种场景下需要抑制编译器优化来保证正确结果）。C语言标准鼓励编译器避免对volatile对象的访问进行优化，但是并没有对volatile有明确的实现定义。C语言标准对其最低的要求为：在序列点\(sequence point \)之前对volatile对象的访问必须已经稳定，并且序列点之后对volatile对象的访问还没有发生；<span style="background-color: #ffaaaa">在两个序列点之间对volatile对象的reorder和combine access规则是自由不作限制的\(reorder和combine访问意味着可以有一定的编译器优化操作空间，如使用寄存器临时值而减少内存访问，但也会在如硬件访问时带来异常问题\)，但是在两个序列点之外则不能存在这种优化行为</span>；

**2.1.2 序列点\(sequence point \)概念**

序列点是程序执行序列中一些特殊的点。 当有序列点存在时，序列点前面的表达式必须求值完毕，并且副作用也已经发生， 才会计算序列点后面的表达式和其副作用。什么是副作用？举例子来说明。

```
int a = 5;
int b = a ++;
```

在给b赋值的语句中，表达式a\+\+就有副作用，它返回a当前的值5后，要对a进行加1的操作。哪些符号会生成序列点呢？

**（1）","会生成序列点**。

","用于把多条语句拼接成一条语句。 例如：

```
int b = 5;
++ b;
```

可由","拼接成

```
int b = 5, ++b;
```

因为","会产生序列点，所以","左边的表达式必须先求值，如果有副作用，副作用也会生效。然后才会继续处理","右边的表达式。

**（2） &&和||会产生序列点**

逻辑与 && 和逻辑或 || 会产生序列点。

因为&&支持短路操作，必须先将&&左边的表达式计算完毕，如果结果为false，则不必再计算&&右边的表达式，直接返回false。

||和&&类似。

**（3）?:中的"?"会产生序列点**

三元操作符 ?:中的"?"会产生序列点。 如：

```
int a = 5;
int b = a++ > 5? 0 : a;
```

b的结果是什么？因为"?"处有序列点，其左边的表达式必须先求值完毕。 a\+\+ \> 5在和5比较时，a并没有自增，所以表达式求值为false。 因为"?"处的序列点，其左边表达式的副作用也要立即生效，即a自增1，变为6。 因为"?"左边的表达式求值为false，所以三元操作符?:返回:右边的值a。 此时a的值是6，所以b的值是6。

**（4） 序列点之间的执行顺序**

奇怪的C代码中给出的例子。

```
int i = 3;
int ans = (++i)+(++i)+(++i);
```

\(\+\+i\)\+\(\+\+i\)\+\(\+\+i\)之间并没有序列点，它们的执行顺序如何呢？ gcc编译后，先执行两个\+\+i，把它们相加后，再计算第三个\+\+i， 再相加。而Microsoft VC\+\+编译后，先执行三个\+\+i，再相加。 两者得到的结果不同，谁对谁错呢？谁也没有错。C标准规定：两个序列点之间的执行顺序是任意的。 当然这个任意是在不违背操作符优先级和结合特性的前提下的。 这个规定的意义是为编译器的优化留下空间。知道这个规定，我们就应该避免在一行代码中重复出现被递增的同一个变量， 因为编译器的行为不可预测。 试想如果\(\+\+i\)\+\(\+\+i\)\+\(\+\+i\)换成\(\+\+a\)\+\(\+\+b\)\+\(\+\+c\)（其中a、b、c是不同的变量）， 不管\+\+a，\+\+b和\+\+c的求值顺序谁先谁后，结果都会是一致的。

**2.2 C\+\+规范文档中volatile隐含意义**

```
This raises the question of what exactly the standard guarantees for volatile. A more detailed exposition on volatile is said to be in preparation, and should that ever emerge from the shadows, this paper will defer to it. In the meantime, referring to N4762(Working Draft, Standard for Programming Language C++.pdf):
1. 4.4.1p6.1 says “Accesses through volatile glvalues are evaluated strictly according to the rules of the abstract machine.”
2. 6.8.1p7 states that volatile accesses are side effects.
3. 6.8.2.1p21 calls out volatile accesses as one of the four forward-progress indicators.
4. 9.1.7.2p5 states that the semantics of an access through a volatile glvalue are implementation-defined. Which should not be a surprise to anyone who does not expect the MMIO registers of every device to be ensconced into the standard.
5. 9.1.7.2p6 (a non-normative note) states:
volatile is a hint to the implementation to avoid aggressive optimization involving the object because the value of the object might be changed by means undetectable by an implementation. Furthermore, for some implementations, volatile might indicate that special hardware instructions are required to access the object. See 6.8.1 for detailed semantics. In general, the semantics of volatile are intended to be the same in C ++ as they are in C.

It is hard to imagine someone intuiting the required semantics of volatile based on the above wording. However, one helpful guideline is that device drivers must work correctly, resulting in the following constraints:
1. Implementations are forbidden from tearing an aligned volatile access when machine instructions of the access's size and type are available. (Note that this intentionally leaves unspecified what to do with 128-bit loads and stores on CPUs having 128-bit CAS but not 128-bit loads and stores.) Concurrent code relies on this constraint to avoid unnecessary load and store tearing.
(在进行volatile修饰的对齐访问时，如果有与访问大小和类型匹配的机器指令可用时，是不能出现tearing访问)
2. Implementations must not assume anything about the semantics of a volatile access, nor, for any volatile access that returns a value, about the possible set of values that might be returned. (This is strongly implied by the implementation-defined semantics called out above.) Concurrent code relies on this constraint to avoid optimizations that are inapplicable given that other processors might be concurrently accessing the location in question.
（在进行volatile修饰的访问时，不能有语义上的假设？）
3. Aligned machine-sized non-mixed-size volatile accesses interact naturally with volatile assembly-code sequences before and after. This is necessary because some devices must be accessed using a combination of volatile MMIO accesses and special-purpose assembly-language instructions. Concurrent code relies on this constraint in order to achieve the desired ordering properties from combinations of volatile accesses and memory-barrier instructions.
（在进行machine-sized non-mixed-size volatile对齐访问时，其前后的代码自然交互？）
```

并行代码依赖于第1,2条约束，来避免在访问非atomic且非volatile类型的对象时，出现因为data race而导致的未定义行为。

**2.3 C语言中volatile使用位置差异**

在C语言中，指针变量存在两次内存访问操作：一方面指针变量本身值，另一方面是指针地址指向的内容；而普通变量只有一次内存访问。

```
char * volatile reg;  //volatile修饰指针变量本身，实现对取变量本身优化抑制
volatile char *reg;  //volatile修饰指针变量的内容，实现对取指针内容的优化抑制
volatile char  * volatile reg; // 以上两者的结合
volatile char val;  //volatile修饰普通变量，实现对取变量优化抑制
```

**三、 GCC扩展汇编中volatile**

GCC的优化有时会将嵌入式asm汇编代码优化丢弃掉，或者是在认为不影响结果的情况下重排代码，这都将导致出现意料之外的问题。如下对硬件的访问函数中asm内需要用volatile防止GCC丢弃掉这段其看来可能没有用的代码。

```
static __always_inline void __raw_writel(u32 val, volatile void __iomem *addr)
{
        asm volatile("str %w0, [%1]" : : "rZ" (val), "r" (addr));
}
```

\_\_volatile\_\_ 表示指示编译器对该嵌入汇编代码不进行代码优化，修改指令顺序等，\_\_asm\_\_ \_\_volatile\_\_可以用asm和volatile，为了避免和C关键字重复警告，最好加下划线。

**四、volatile在linux内核中的应用场景**

1、访问I/O寄存器的函数可能会使用volatile类型。（C语言volatile在访问硬件）

2、 被inline并改变内存值得汇编代码可能会由于没有其他可见的副作用而被GCC编译器优化掉。在asm语句前加上volatile可以避免被优化。（GCC扩展汇编）

3、 Linux中jiffies变量是一个特殊的存在，因为每次引用时它都可能是不同的值，同时读取该变量值的地方又是无锁的。因此jiffies可以被定义成volatile类型， 但是强烈反对对其他变量使用volatile类型。在这方面，jiffies被认为是“遗留的愚蠢”问题（linus说的），修复它带来的麻烦远比其价值更多。（C语言volatile在线程间共享变量通信）

4、指向一致性内存中可能由I/O设备修改的数据结构的指针有时可以合理声明volatile。（C语言volatile在访问硬件时，用于修饰指针变量值本身）

四、参考资料

1、[https://www.cnblogs.com/jiqingwu/p/c\_sequence\_point.html](https://www.cnblogs.com/jiqingwu/p/c_sequence_point.html)

2、[https://zhuanlan.zhihu.com/p/102406978](https://zhuanlan.zhihu.com/p/102406978)

3、[https://www.open\-std.org/JTC1/SC22/WG21/docs/papers/2020/p0124r7.html](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2020/p0124r7.html)
