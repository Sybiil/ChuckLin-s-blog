# vsftpDaemon搭建问题手册

---

##VSFTPD是什么
vsftpd 的全名是『Very Secure FTP Daemon 』，它是一个ftp服务器。由于其支持change root directory且相对其他ftp更安全的特点，常常作为公司搭建ftp服务器时的首选。

如果你是通过yum下载RPM版的vsftpd并保持默认选项安装的话，与其他通过yum安装的软件一样，根据linux FHS标准，vsftpd的配置文件将默认安装到/etc/vsftpd下面，而执行文件则会安装到/usr/sbin下面。

请注意，尝试在centOS7.x版本的操作系统上安装vsftpd时由于不会在etc/init.d下面安装启动daemon的shell，所以你也无法通过service vsftpd start来启动ftpd。新版本的vsftpd的启动方式非常简单，直接执行sbin中的执行文件就可以了，如下：
```shell
/usr/sbin/vsftpd
```
位于/etc/vsftpd中的vsftpd.conf是vsftpd最重要的配置文件。关于ftp的大多数主要设定都在这个配置文件中进行。由于篇幅所限，本文不涉及关于该文件的具体配置方法。若如果你对如何配置vsftpd有疑问的话，请参阅其他资料。

##发生异常与解决方法
###500 OOPS: run two copies of vsftpd for IPv4 and IPv6

笔者本次在尝试启动vsftpd时，vsftpd通过shell返回了一个异常信息。
```shell
/usr/sbin/vsftpd
500 OOPS: run two copies of vsftpd for IPv4 and IPv6
```
产生这个异常的原因是因为你在配置文件中同时存在ipv4与ipv6的相关配置。如果你只需要在ipv4的环境中使用ftpd的话，直接注释掉文件中的“listen_ipv6=YES”就可以了。

虽然VSFTPD Ftp Server本身支持IPv6网络，而且在配置文件（/etc/vsftpd/vsftpd.conf）中就有一个“#listen_ipv6=YES”选项，但是默认情况下IPv6服务是不生效的。在实际应用中，如果我们想让VSFTPD同时支持IPv4和IPv6，似乎只要把“#listen_ipv6=YES”取消注释、然后重启VSFTPD服务就OK了。不过我在实际操作中发现，如果只是简单地取消掉“listen_ipv6=YES”的注释，重启服务时会提示VSFTPD在IPv4和IPv6网络下同时运行产生冲突。

这可咋办，难道VSFTPD不能同时在IPv4和IPv6网络下运行吗？当然不是这样，VSFTPD是可以同时运行在IPv4和IPv6网络下的。
不过需要使用两个配置文件，将“listen=YES”和“listen_ipv6=YES”分别放在两个配置文件中，一个负责监听IPv4，一个负责监听IPv6，这样就不会冲突了。

经过测试，IPv4和IPv6确实都可以正常连接。


###500 oops:refusing to run with writable root inside chroot
历经千辛万苦，终于将ftpd运行起来了。但是通过ftp客户端软件进行测试时又产生了异常信息！
```shell
ftp localhost...
username...
password...
500 OOPS: vsftpd: refusing to run with writable root inside chroot ()  
```
看异常信息我们就知道这一定和change root directory有关！
原来，从2.3.5之后，vsftpd增强了安全检查，如果用户被限定在了其主目录下，则该用户的主目录不能再具有写权限了！如果检查发现还有写权限，就会报该错误。
解决方法也很简单，编辑vsftpd.conf文件，在文件末尾加上
```shell 
allow_writeable_chroot=YES
```
就可以了。

###本地连接ftp服务器正常，局域网或因特网尝试连接ftp服务器失败
通过tty7接口，图形操作界面打开你的防火墙，将ftp服务添加到信任列表中。
![ftp.png-224.6kB][1]

##总结
1. vsftpd搭建时最重要的配置文件为vsftpd.conf文件。
2. vsftpd在默认安装后就能满足基本使用。但是鉴于chroot优秀的安全性，建议最好还是掌握vsftpd.conf的配置方法。
3. 由于配置了chroot，所以每个不同的登录ftp的账号都拥有自己的root目录。其实际位置位于账户home目录下。由于受到chroot管理的账号无法进入其他目录，当你需要在多个用户之间共享文件时，请使用唯一的ftp账号。
4. 类似于防火墙这样的设置，通过tty7接口的图形操作界面来进行设置会取得事半功倍的效果。

  [1]: http://static.zybuluo.com/mikumikulch/4ladtk4ybdd8qv4wcqpn1rez/ftp.png
  
---
Copyright 2017/08/15 by Chuck Lin



若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！

![alipay.jpg-17.7kB][99]![wechat.jpg-16.7kB][98]


[99]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[98]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg




