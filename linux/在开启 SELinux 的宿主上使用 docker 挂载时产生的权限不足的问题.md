# 在开启 SELinux 的宿主上使用 docker 挂载时产生的权限不足的问题

---

##什么是 SElinux
SELinux：『 Security Enhanced Linux 』的缩写，最初由美国国家安全局基于 linux 研发，后来被贡献给开源组织。
现已成为各 linux 发行版中的一部分。

##SELinux 有什么用？
要理解 SELinux 有什么用，首先你要明白 linux 传统的权限管理模式。
linux 传统的权限管理称为『自主式存取控制 (Discretionary Access Control, DAC)』，基本上就是依据程序的拥有者与文件资源的 rwx 权限来决定有无存取的能力。

为了解决这种单一的权限管理模式，SELinux 导入了委任式存取控制 (Mandatory Access Control, MAC)。

##MAC 的工作方式
 MAC 可以针对特定的程序与文件来进行管理。MAC 的管理方式大概分为4个模块分别是：
 - 主体（ sbuject ）
 - 目标 ( object )
 - 政策 ( policy ) 
 - 安全性文本（security context）
 简而言之，MAC 通过既定的各种政策，将主体（也就是 process）通过安全性文本，与目标进行对比从而最终得出主体对于目标是否有使用权限的方式。
有没有觉得似曾相识的感觉？是的，与 RBAC(Resource-Based Access Control)的思路基本一致，MAC 的思想就是基于资源进行权限管理。

简单来说，SELinux 在本身的基于角色对文件资源进行管理的基础上，又增加了一层基于资源的权限管理功能。就是这么简单！

##问题回顾
现在在开启了 SELinux 的宿主上，尝试启动一个 nginx 容器并且尝试将 nginx.conf 挂载到容器内。
```
docker run —-name totalextripNginx  -p 80:80 -p 443:443   -v /var/totalextrip/config/nginx/nginx.conf:/etc/nginx/nginx.conf  -d nginx
```
使用 docker ps 查看容器状态![ nginx_docker_error_pic.png-63.3kB][1]
**出错了！Permission denied。**

系统明确告诉我们是权限的问题，那就检查 nginx.conf 的权限看是否符合我们的要求就行了。
使用 ls -Z 查看 nginx.conf的 DAC 与 MAC 权限信息。
![nginx_info.png-24kB][2]
文件的权限为 644。 

通过 ps aux 查看 docker 进程。
docker 进程的权限为 root，对于644的权限文件是可读可写的。
![psdocker.png-104.8kB][3]
看来，问题应该是出在 MAC 权限上。

分析 ls -Z 的结果，nginx.conf 对应的安全性文本的类型为 svirt_sandbox_file_t:s0。
我们的主体是否能够操作这种类型的 object 呢？

使用 sesearch 找出目标文件资源类别为svirt_sandbox_file_t的有关信息
![find_svirt.png-26.4kB][4]
无法找到任何相关信息。

在当前 SELinux 政策下，对于 nginx.conf 的类型的文件是无法操作的。
所以无论 docker 容器的权限是否是 root，docker 容器进程都没有权限读取宿主上的 nginx.conf。

##解决方案
docker 官方提供了一种解决方案专门用来解决与 SELinux 相关的权限问题。
在将 SELinux 上的文件挂载到容器中时，在挂载的路径最后加上:z 后:Z
如：
```
docker run -v /var/db:/var/db:z rhel7 /bin/sh
```
docker 会自动将被挂载的宿主目录的安全性文本配置为目标可读。
```shell
docker run —-name totalextripNginx  -p 80:80 -p 443:443   -v /var/totalextrip/config/nginx/nginx.conf:/etc/nginx/nginx.conf:z  -d nginx
```
```shell
[root@totalextrip nginx]# docker ps -a
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS              PORTS                          NAMES
2f6e5808292e        mysql                "docker-entrypoint.sh"   20 hours ago        Up 20 hours         0.0.0.0:3306->3306/tcp         totalextripMysql
a3403e48a139        nginx                "nginx -g 'daemon off"   20 hours ago        Up 20 hours         0.0.0.0:80->80/tcp, 443/tcp    totalextripNginx
```


##后记
SELinux 的概念与配置不是一片文章就能够说明清楚的。关于各项政策的配置，启用，安全性文本的修改以及与 docker 的配合方面，笔者掌握的也不是很透彻。
若你有相关的建议与意见，或者发现文章中有什么错误，请及时与笔者联系。


---

Copyright 2017/08/15 by Chuck Lin

若文章有幸帮到了您，您可以捐助我，以鼓励我写出更棒的作品！
![alipay.jpg-17.7kB][99]![wechat.jpg-16.7kB][98]


[99]: http://static.zybuluo.com/mikumikulch/6g65s5tsspdmsk87a8ariszo/alipay.jpg
[98]: http://static.zybuluo.com/mikumikulch/rk5hldgo4wi9fv23xu3vm8pf/wechat.jpg





  [1]: http://static.zybuluo.com/mikumikulch/mpspffwctyo3qmtc3egntqqv/%20nginx_docker_error_pic.png
  [2]: http://static.zybuluo.com/mikumikulch/fwve4e48724e2ardxhvm3dff/nginx_info.png
  [3]: http://static.zybuluo.com/mikumikulch/lb16gxe84dmhc9idsa428ks1/psdocker.png
  [4]: http://static.zybuluo.com/mikumikulch/znwk556fzlq5iq7s4yanlox8/find_svirt.png
  [5]: http://static.zybuluo.com/mikumikulch/zr0yyn6o2ypoccfeqzj0g4hv/nginx_ok.png