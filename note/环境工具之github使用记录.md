# 环境工具之github使用记录

### 一、linux下搭建使用github

### 1 Linux下Git和GitHub环境的搭建

1. 安装Git， 使用命令sudo apt\-get install git
2. 创建GitHub帐号
3. 生成ssh key，使用命令 ssh\-keygen \-t rsa \-C "
4. 回到github，进入Account Settings，左边选择SSH Keys，Add SSH Key，粘贴key。（ 打开id\_rsa.pub，复制里面的key，全部字符 ）
5. 测试ssh key是否成功，使用命令ssh \-T 
6. 配置Git的配置文件：配置用户名：git config \-\-global user.name "your name" ，配置email：git config \-\-global user.email "your email"

### 2 利用Git从本地上传到GitHub

1. 进入要所要上传文件的目录, 输入命令 git init
2. 创建一个本地仓库origin，使用命令 git remote add origin 
3. 比如你要添加一个文件xxx到本地仓库，使用命令 git add xxx，可以使用 git add . 自动判断添加哪些文件
4. 然后把这个添加提交到本地的仓库，使用命令 git commit \-m "说明这次的提交"
5. 最后把本地仓库origin提交到远程的GitHub仓库，使用命令 git push origin master

### 3 从GitHub克隆项目到本地

1. 到GitHub的某个仓库，然后复制右边的那个（HTTPS clone url）
2. 回到要存放的目录下，使用命令 git clone 
3. 如果本地的版本不是最新的，可以使用命令 git fetch origin，origin是本地仓库
4. 把更新的内容合并到本地分支，可以使用命令 git merge origin/master
5. 如果你不想手动去合并，那么你可以使用： git pull \<本地仓库\> master 这个命令来拉去最新版本并自动合并

### 4 GitHub的分支管理

创建分支

1. 创建一个本地分支： git branch \<新分支名字\>
2. 将本地分支同步到GitHub上面： git push \<本地仓库名\> \<新分支名\>
3. 切换到新建立的分支： git checkout \<新分支名\>
4. 为你的分支加入一个新的远程端： git remote add \<远程端名字\> \<地址\>
5. 查看当前仓库有几个分支: git branch

删除分支

1. 从本地删除一个分支： git branch \-d \<分支名称\>
2. 同步到GitHub上面删除这个分支： git push \<本地仓库名\> :\<GitHub端分支\>

### 5 常见错误

如果出现报错为ERROR: Repository not found.fatal: The remote end hung up unexpectedly则代表你的 origin 的url 链接有误，可能是创建错误，也可能是这个 [git@github.com](mailto:git@github.com):xxx/new\-project.git url 指定不正确。重新创建。

**Git 的详细介绍**：[请点这里](http://www.linuxidc.com/Linux/2013-10/91054.htm "Git")

**Git 的下载地址**：[请点这里](http://www.linuxidc.com/down.aspx?id=1022)

克隆一个代码分支：

git clone [https://github.com/LinuxCNC/linuxcnc.git](https://github.com/LinuxCNC/linuxcnc.git)
