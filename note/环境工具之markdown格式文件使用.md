# 环境工具之markdown格式文件使用

**一、介绍**

    1、用于写文档，颜色格式不够丰富，可以满足基本需求

    2、用于写书，加上工具的使用，可以满足生成pdf书籍

    3、代码中用于README，但是不合适加入图片，源码中不合适放入图片

    4、如果输出的文档为pdf格式，虽然md文件本身不包含文件，但是可以在本地插入图片，生成的时候pdf有图片

**二、语法**

1、 标题

代码：

```
# 一级标题 ## 二级标题 ### 三级标题 #### 四级标题 ##### 五级标题 ###### 最小只有六级标题
```

备注： \#后面留个空格，否则有些渲染工具不兼容

2 、加粗

代码：

```
**我被加粗了**
```

3 、斜体

代码：

```
*我倾斜了了*
```

4、 高亮

代码：

```
==我高亮了==
```

备注： chrom的插件和MP2工具无法渲染

5 、上标 （2的2次方）

代码：

```
2^2^
```

备注： chrom的插件和MP2工具无法渲染

6、下标 （水的分子式）

代码：

```
H~2~o
```

备注： chrom的插件和MP2工具无法渲染

7、 引用（\>式）

代码：

```
> hello markdown! 
>> hello markdown!
```

8、 代码引用（\`\`\`式）

代码：

```
```python print('hello nick')```
```

9、 代码引入（\`式）

代码：

```
`print('hello nick')`
```

10、 插入链接（只显示链接显示本身）

代码：

```
<https://www.cnblogs.com/nickchen121/p/10718112.html>
```

11、 插入链接（链接描述显示，不显示具体链接）

代码：

```
[nickchen博客](https://www.cnblogs.com/nickchen121/p/10718112.html%20"nickchen博客")
```

备注：中间的空格用%20转义

12、 插入图片（链接，跟链接类似前面多一个！符号）

代码：

```
![数据类型总结-搞笑结束.jpg?x-oss-process=style/watermark](http://www.chenyoude.com/Python从入门到放弃/数据类型总结-搞笑结束.jpg?x-oss-process=style/watermark '描述信息')
```

13、 插入图片（图片路径）

代码：文件linuxcnc001.png和md文件的相对路径

```
MP2同时可以支持linux和windows的路径格式：
![linuxcnc](.\images\linuxcnc001.png)
![linuxcnc](./images/linuxcnc001.png)

chrome只支持md文件和图片同一路径：
![linuxcnc](linuxcnc001.png)
```

14、 有序列表

代码：

```
1. one 
2. two 
3. three
```

15、 无序列表（以黑色o开头，用\-也可以）

代码：

```
* one
* two
* three
```

16 、分割线（一条分割线 ）

代码：

```
---
或者
___
```

17、 表格而且第二行必须得有，并且第二行的冒号代表对齐格式，分别为居中；右对齐；左对齐）：

```
|学号|姓名|序号|
|-|-|-|
|小明明|男|5|
|小红|女|79|
|小陆|男|192|

或
| name | age | sex
| :-:|:-|-:
|tony|20|男
|lucy|18|女
```

备注：MP2不支持表格渲染  chrome插件支持

18、 数学公式（行内嵌）

代码：

```
$\sum_{i=1}^{10}f(i)\,\,\text{thanks}$
```

备注： chrom的插件和MP2工具无法渲染

19、 数学公式（块状）

代码：

```
$$ \sum_{i=1}^{10}f(i)\,\,\text{thanks} $$
```

备注： chrom的插件和MP2工具无法渲染

20、代码块\(代码前面空一个table\)

代码：

```
    __irq_svc:
        svc_entry
        irq_handler

    #ifdef CONFIG_PREEMPT
        get_thread_info tsk
        ldr     r8, [tsk, #TI_PREEMPT]          @ get preempt count
        ldr     r0, [tsk, #TI_FLAGS]            @ get flags
        teq     r8, #0                          @ if preempt count != 0
        movne   r0, #0                          @ force flags to 0
        tst     r0, #_TIF_NEED_RESCHED
        blne    svc_preempt
    #endif

        svc_exit r5, irq = 1                    @ return from exception
    UNWIND(.fnend          )
    ENDPROC(__irq_svc)
```

备注： 宏定义会影响渲染，前面也需要有table符

参考资料：[https://markdown.com.cn/basic\-syntax/horizontal\-rules.html](https://markdown.com.cn/basic-syntax/horizontal-rules.html)

**三、编辑与输出**

1、md文件编辑

    MarkdownPad 可以进行实时渲染

2、md文件转pdf

    （A）使用chrom浏览器，安装Markdown Viewer插件，打印输出pdf文件

    具体参考：

[    https://blog.csdn.net/twingao/article/details/105170034](https://blog.csdn.net/twingao/article/details/105170034)

    （B）linux 下使用pandoc \+  LeTex 工具转换

**四、测试**

    E:\\test\\test.md

**%\!\(EXTRA markdown.ResourceType=, string=, string=\)**
