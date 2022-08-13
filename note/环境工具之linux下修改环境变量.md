# 环境工具之linux下修改环境变量

`linux` `环境变量`

**方法一：**

　　在/etc/profile文件中添加变量【对所有用户生效\(永久的\)】

　　用VI在文件/etc/profile文件中增加变量，该变量将会对[Linux](http://www.chinabyte.com/keyword/Linux/)下所有用户有效，并且是“永久的”。

　　要让刚才的修改马上生效，需要执行以下代码

　　\# source /etc/profile

**方法二：**

　　在用户目录下的.bash\_profile文件中增加变量【对单一用户生效\(永久的\)】

　　用VI在用户目录下的.bash\_profile文件中增加变量，改变量仅会对当前用户有效，并且是“永久的”。

　　要让刚才的修改马上生效，需要在用户目录下执行以下代码

　　\# source .bash\_profile

**方法三：**

　　直接运行export命令定义变量【只对当前shell\(BASH\)有效\(临时的\)】

　　在shell的命令行下直接使用\[export变量名=变量值\]定义变量，该变量只在当前的shell\(BASH\)或其子shell\(BASH\)下是有效的，shell关闭了，变量也就失效了，再打开新shell时就没有这个变量，需要使用的话还需要重新定义。

　　例如：export PATH=/usr/local/webserver/php/bin:$PATH
