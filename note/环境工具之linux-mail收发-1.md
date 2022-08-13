# 环境工具之linux-mail收发

[https://blog.csdn.net/Nolan\_\_Roronoa/article/details/52335223](https://blog.csdn.net/Nolan__Roronoa/article/details/52335223)

msmtp是专门负责邮件发送的SMTP客户端软件，mutt是邮件用户代理客户端，由于是一个代理所以必须要用到你自己的邮箱（比如163，gmail，outlook，qq均可）。

 1、 一定要注意：在使用mutt发送邮件之前，一定要保证你的邮箱开通了SMTP/POP3服务，允许使用第三方代理客户端！

   我以163邮箱为例，先进入你的163邮箱，在设置下面的POP3/SMTP/IMAP下设置POP3/SMTP服务

   开启服务之后，163邮箱会让你设置授权码，一定要记住这个授权码，下面讲到的邮箱配置文件中的密码是这个授权码，而不是你登陆邮箱的密码\!

 2、接着，开始下载mutt和msmtp这两个软件

    sudo apt\-get install mutt

  3、  安装好之后可以进行配置，我这里是在我的用户文件夹下面进行配置，而不是全局配置，大家可以根据自己偏好选择。

    vim ~/.muttrc  新建输入下面内容

```
  1 set sendmail="/usr/bin/msmtp"
  2 set use_from=yes
  3 set realname="Long Li"
  4 set from=lonuxli@163.com
  5 set envelope_from=yes  
```

    realname没有什么所谓，但是接下来有个用户名就有明确讲究啦

    接下来，新建msmtp的日志文件

    touch ~/.msmtp.log

    然后配置msmtp文件， vim ~/.msmtprc   新建输入

```
  1 account▼▼       default
  2 host▼   ▼       smtp.163.com
  3 user▼   ▼       lonuxli
  4 from▼   ▼       lonuxli@163.com
  5 password▼       TSWNNNASHGQUXQFS
  6 auth▼   ▼       login
  7 tls▼    ▼       off
  8 logfile▼▼       ~/.msmtp.log
```

    由于密码是明文，所以修改文件的权限，只允许自己可以访问该文件

    chmod 600 ~/.msmtprc

   大家一定要注意user和password一项，由于密码是之前的授权码，所以在配置之前一定要设置好邮箱\!

4、测试

    配置好之后可以输入“msmtp \-\-host=[smtp.163.com](http://smtp.163.com/) \-\-serverinfo”命令，进行测试，输出以下结果

```
lilong@ubuntu:~$ msmtp --host=smtp.163.com --serverinfo             
SMTP server at smtp.163.com (m12-11.163.com [220.181.12.11]), port 25:
    163.com Anti-spam GT for Coremail System (163com[20141201])
Capabilities:
    PIPELINING:
        Support for command grouping for faster transmission
    STARTTLS:
        Support for TLS encryption via the STARTTLS command
    AUTH:
        Supported authentication methods:
        PLAIN LOGIN
This server might advertise more or other capabilities when TLS is active.
```

    然后就可以开始使用命令行发送邮件了\!

     echo "hello world" | mutt \-s "title" 目标邮箱

5、邮件发送
