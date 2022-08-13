# 环境工具之Linux内核开发环境

清华源：

[https://mirrors.tuna.tsinghua.edu.cn/help/linux.git/](https://mirrors.tuna.tsinghua.edu.cn/help/linux.git/)

git clone加速方法：

[https://cloud.tencent.com/developer/article/1571945](https://cloud.tencent.com/developer/article/1571945)

git 管理

[https://blog.csdn.net/emtribe/article/details/8915884](https://blog.csdn.net/emtribe/article/details/8915884)

谢宝友手把手教你

[https://zhuanlan.zhihu.com/p/87955006](https://zhuanlan.zhihu.com/p/87955006)

git clone [https://mirrors.tuna.tsinghua.edu.cn/git/linux/git](https://mirrors.tuna.tsinghua.edu.cn/git/linux/git)

```
git remote add linux-next https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
git remote add linux-stable https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git

git fetch --tags linux-next

git branch -D llbranch  删除某个分支

git branch mybranch next-20200626  创建tag为next-20200626的分支
```

设置git代理：

git config \-\-global http.proxy 'socks5://127.0.0.1:1080'  

nohup /etc/trojan/trojan \-c /etc/trojan/config.json   &

设置git邮箱：

git config \-\-global user.name "Long Li"

git config \-\-global user.email "lonuxli.64[@gmail.com](mailto:hitsjt@gmail.com)"

gmail设置

sudo apt\-get install sendemail

git config \-\-global sendemail.smtpserver [smtp.gmail.com](http://smtp.gmail.com/)

git config \-\-global sendemail.smtpserverport 587

git config \-\-global sendemail.smtpencryption tls

git config \-\-global sendemail.smtpuser [lonuxli.64@gmail.com](mailto:lonuxli.64@gmail.com)

  572  git branch my\-test

  573  git checkout my\-test

  575  vi  commit.txt

  577  git add commit.txt

  578  git commit \-s \-m "Add format strings for bb\_error\_msg\_and\_die"

  581  git format\-patch \-C \-n master..my\-test

  582  git send\-email \-\-compose \-\-no\-chain\-reply\-to \-\-suppress\-from \-\-to [smyrll@163.com](mailto:smyrll@163.com) 0001\-\*.patch

```
mutt邮件发送
//发送测试
echo "test" |mutt -s "my_first_test" smyrll@163.com

//提交
git commit -s

//patch生成
git format-patch -s -v 5 -1    

./scripts/checkpatch.pl v1-0001-mm-free-unused-pages-in-kmalloc_order.patch
./scripts/get_maintainer.pl  v1-0001-mm-free-unused-pages-in-kmalloc_order.patch
//发送邮件
mutt -H v1-0001-mm-free-unused-pages-in-kmalloc_order.patch  smyrll@163.com -c lonuxli@163.com -c lonuxli.64@gmail.com

mutt -H v1-0001-mm-migrate-fix-comment-spelling.patch  akpm@linux-foundation.org  -c linux-mm@kvack.org -c linux-kernel@vger.kernel.org

//mutt打开发件箱
按‘c’键，显示Open mailbox ('?' for list):
按 tab 键 显示信箱，选择outbox目录，里面是已发送邮件
    
//mutt中D选择删除符合搜索的partern的邮件
http://www.mutt.org/doc/manual/
```

```
$             回退到上个版本
$ git reset --hard HEAD~3        回退到前3次提交之前，以此类推，回退到n次提交之前
$ git reset --hard commit_id     退到/进到 指定commit的sha码

当然–soft 和–hard --mixed的区别可以如下理解：
* soft
只操作了HEAD，暂存区和work都没有被影响
* hard
操作了HEAD、暂存区和work，都被影响了
* mixed
操作了HEAD、暂存区，work没有被影响
```

```
Git 提供了两种补丁方案，一是用git diff生成的UNIX标准补丁.diff文件，二是git format-patch生成的Git专用.patch 文件，有commit信息。

检查patch/diff是否能正常打入:
git apply --check 【path/to/xxx.patch】
git apply --check 【path/to/xxx.diff】

打入patch/diff:
git apply 【path/to/xxx.patch】
git apply 【path/to/xxx.diff】
或者
git  am 【path/to/xxx.patch】
```

```
git查看提交记录    
1. git log filename   查看某个文件的commit记录
2. git log -p filename     查看文件每次提交的diff
3. git log --pretty=oneline filename 列出文件的所有改动历史
4. git show 提交生成的一次哈希值 filename   只查看某次提交的文件变化
```

```
git tag --list 'v-*'        list only the tags that match the pattern(s).
git tag --list 'v4.19*'    

git remote -v                列出所有remote信息
git branch -a                List both remote-tracking branches and local branches.
git branch -vv                 //查看设置的所有跟踪分支，可以使用 git branch 的 -vv 选项。 
                                这会将所有的本地分支列出来并且包含更多的信息，如每一个分支正在跟踪哪个远程分支与本地分支是否是领先、落后或是都有。
git branch -v -a               //显示当前使用仓库的所有分支
git remote show origin         // 查看本地分支与远程分支的对应关系

git checkout --track origin/branch_name  //在本地新建一个分支名叫branch_name，会自动跟踪远程的同名分支branch_name。
                                         //如果基于的分支branch_name是远程分支名，那么本地分支会自动跟踪远程分支。
git checkout -b new_branch_name branch_name  //如果想新建一个本地分支不同名字，同时跟踪一个远程分支可以利用。
git push --set-upstream origin branch_name   //在远程创建一个与本地branch_name同名的分支并跟踪
git checkout --track origin/branch_name      //在本地创建一个与branch_name同名分支跟踪远程分支.

```

```
lilong@lilong:~/gitwork/kernel/linux$ git remote show origin
* remote origin
  Fetch URL: https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
  Push  URL: https://mirrors.tuna.tsinghua.edu.cn/git/linux.git
  HEAD branch: master
  Remote branch:
    master tracked
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (fast-forwardable)
lilong@lilong:~/gitwork/kernel/linux$
lilong@lilong:~/gitwork/kernel/linux$
lilong@lilong:~/gitwork/kernel/linux$ git remote show linux-next
* remote linux-next
  Fetch URL: https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
  Push  URL: https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
  HEAD branch: master
  Remote branches:
    akpm          tracked
    akpm-base     tracked
    master        tracked
    pending-fixes tracked
    stable        tracked
  Local ref configured for 'git push':
    master pushes to master (local out of date)
```

444050990db4 mm, slab: check GFP\_SLAB\_BUG\_MASK before alloc\_pages in kmalloc\_order
