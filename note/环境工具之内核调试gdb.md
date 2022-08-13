# 环境工具之内核调试gdb

```
gcc -g  main.c                      //在目标文件加入源代码的信息
gdb a.out       

(gdb) start                         //开始调试
(gdb) n                             //一条一条执行
(gdb) step/s                        //执行下一条，如果函数进入函数
(gdb) backtrace/bt                  //查看函数调用栈帧
(gdb) info/i locals                 //查看当前栈帧局部变量
(gdb) frame/f                       //选择栈帧，再查看局部变量
(gdb) print/p                       //打印变量的值
(gdb) finish                        //运行到当前函数返回
(gdb) set var sum=0                 //修改变量值
(gdb) list/l 行号或函数名             //列出源码
(gdb) display/undisplay sum         //每次停下显示变量的值/取消跟踪
(gdb) break/b  行号或函数名           //设置断点
(gdb) continue/c                    //连续运行
(gdb) info/i breakpoints            //查看已经设置的断点
(gdb) delete breakpoints 2          //删除某个断点
(gdb) disable/enable breakpoints 3  //禁用/启用某个断点
(gdb) break 9 if sum != 0           //满足条件才激活断点
(gdb) run/r                         //重新从程序开头连续执行
(gdb) watch input[4]                //设置观察点
(gdb) info/i watchpoints            //查看设置的观察点
(gdb) x/7b input                    //打印存储器内容，b--每个字节一组，7--7组
(gdb) disassemble                   //反汇编当前函数或指定函数
(gdb) si                            // 一条指令一条指令调试 而 s 是一行一行代码
(gdb) info registers                // 显示所有寄存器的当前值
(gdb) x/20 $esp                    //查看内存中开始的20个数
```

## 命令

[GDB](https://www.gnu.org/software/gdb/) 是 Linux 下的命令行调试工具。

启动 GDB 有如下几种方式：

1. gdb \<program\> 直接启动执行程序
2. gdb \<program\> core 用gdb 同时调试一个可执行程序和core文件。core 是程序非法执行后 core dump 产生的文件
3. gdb \<program\> \<PID\> 指定进程， gdb会自动 attach 上去。program 应该在 PATH 环境变量中可以搜索得到。

### 常用的 gdb 命令如下

#### 信息 info

info 可以简写成 i

- info args 列出参数
- info breakpoints info break i b 列出所有断点
- info break number i b number 列出序号为 number 的的断点
- info watchpoints i watchpoints 列出所在 watchpoints
- info threads 列出所有线程
- inifo registers 列出寄存器的值
- info set 列出当前 gdb的所的设置
- i frame
- i stack
- i locals
- i catch

#### 断点和监视 break & watch

break 可以简写为 b

- break fun\_name b fun\_name 在 fun\_name 处打断点
- b line\_number 在 line\_number 处打断点
- b \+offset 在offset 行后加断点
- b \-offset 同上
- b file\_name:fun\_name 文件的方法名处打断点
- b fine\_name:line\_number 同上
- b \*address 在某地址处打断点。适用于没有源码的情况
- b line\_number if condition 条件断点。当条件为 true 时会中断
- watch 可以添加监视。watch 没有简写
    tbreak tb 单次命中断点。如 tb 12 表示只会在第12行命中一次。
- watch var 监视var变量
- watch condition 带条件的监视。如 watch a\>1

删除和禁用断点

- clear 清除当前行的断点
- clear fun\_name 清除 fun\_name 中的所有断点
- clear line\_number 删除该行的断点
- delete 简写 d . 删除所有的 breakpoints, watchpoints, or catchpoints.
- d num 删除序号为 num 的断点
- d num1\-num2 删除序号从 num1 到 num2 的所有断点
- disable/enable num 禁用/启用序号为 num 的断点
- disable/enable num1\-num2 禁用/启用 序号从 num1到 num2 的断点

#### 调试

- run 简写 r 运行程序
- step 简写 s 单步，可以进入方法（相当于 VS 的 F11）
- finish 跳出当前方法\(相当于VS 的 sh\+F11\)
- next 简写 n 单步，不会进入方法（相当于 VS 的 F10）
- until line\-number 运行到 line\-number 行。line\-number 只能比当前行数大。 until 还可以接 function name, address, filename:function or filename:line\-number
- where 显示当前行数和方法
- backtrace 简写 bt 显示当前的栈信息
- bt full 打印完整的栈信息
- frame 简写 f 显示当前栈的 frame 信息
- f number 选择frame
- up / down / up number /down number 选择 frame

#### 源码

- list 简写 l 列出源码
- l num 列出 num 行前后的源码
- l fun 列出方法 fun 的源码
- l start\_num, end\_num
- l file\_name:fun\_name
- set listsize count 设置一次显示多少行源码\(默认为10\)
- show listsize 显示listsize
- directiory dir\_name 简写 dir dir\_name , 将指定的目录加入源码文件的前缀
- show dir
- i line 显示 l 所指的行\(注意不是当前行\)在obj中的起始地址
- i line line\_number 同上
- stepi si 汇编级调试
- nexti ni

#### 变量

- print var 简写 p var 打印变量var
- p file\_name:var
- p/x var 以16进制打印整型变量 。
- p/d var 10进制
- p/u var unsigned int
- p/o var 8进制
- p/t var 2进制 \(1byte/8bits\)
- p/c var 以字符形式
- p/f var 以 floating 形式
- p/a var 以十六进制地址
- x/4b &var 以 4 byte 打印var 的内存
- ptype var 显示var 的类型
- ptype date\-type 显示原类型

#### 启动

- run 简写 r
- continue 简写 c
- kill 杀掉当前调试的程序
- quit 简写 q 退出gdb

Controlling GDB

   

set gdb\-option value

 设置 GDB 的选项。

 

set print array on

set print array off

show print array

 以可读形式打印数组。默认是 off 。

 

set print array\-indexes on

set print array\-indexes off

show print array\-indexes

 打印数组元素的下标。默认是 off 。

 

set print pretty on

set print pretty off

show print pretty

 格式化打印 C 结构体的输出。

 

set print union on

set print union off

show print union

 打印 C 中的联合体。默认是 on

```
mount -t ramfs /dev/ram0 /var/
```
