# 环境工具之vim插件使用

[http://vimdoc.sourceforge.net/htmldoc](http://vimdoc.sourceforge.net/htmldoc)

：ls   查看buffer

切换窗口 Ctrl \+ w 按一次加上下键可以自己选择切换方向，按两次则从上到下自动遍历窗口

扩大窗口 Ctrl\-w \+ 扩大窗口

缩小窗口 Ctrl\-w \- 缩小当前编辑窗口

在vim中执行shell命令 :\! ls 这样可以执行shell命令，或者也可以 :shell top

放大当前窗口，缩小其他窗口 :res ，后面可以设置行数，比如 :res 10 则将当前窗口设置为10行

c.vim 使能配置

   let  g:C\_UseTool\_cmake    = 'yes'

   let  g:C\_UseTool\_doxygen = 'yes'

omnicomplete配置

插入模式编辑 C/C\+\+ 源文件时按下 . 或 \-\> 或 ::，或者手动按下 Ctrl\+X Ctrl\+O 后就会弹出自动补全窗口，此时可以用 Ctrl\+N 和 Ctrl\+P 上下移动光标进行选择。

注释掉mouse即可解决点击鼠标进入可视模式

58 "set mouse=a       " 可以在buffer的任何地方使用鼠标（类似office中在工作区双击鼠标定位）

find \`pwd\`  \-name "\*.h" \-o \-name "\*.c" \-o \-name "\*.cc" \-name "\*.cpp" \-name "\*.s" \-name "\*.S" \> cscope.files                                     

 cscope \-bkq \-i cscope.files

lookupfile

```
#!/bin/sh

#generate tag file for lookupfile plugin
echo -e "!_TAG_FILE_SORTED\t2\t/2=foldcase/" > filename.tags
find . -not -regex '.*\.\(png\|gif\)' -type f -printf "%f\t%p\t1\n" | sort -f >> filename.tags
```
