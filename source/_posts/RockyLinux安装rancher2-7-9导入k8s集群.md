---
title: RockyLinux安装rancher2-7-9导入k8s集群
date: 2024-01-20 10:07:47
tags:
  - k8s 
  - Rocky Linux
  - rancher2.7.9
---

>rancher2.7.9官网有好几种安装方式，目前不推荐使用docker安装的方式了，但是作为个人学习使用，使用docker方式安装是最方便快捷的了。

当前采用的系统是Rocky Linux 9.3,已装好docker。

打开rancher官网：https://www.rancher.cn/quick-start

  ![](/images/rancher_required.jpeg)

拷贝图中的命令执行即可：

```bash
  sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:stable
```

>截止目前为止(2024-01-20)，如果把镜像版本更改为rancher:2.7.9，会提示找不到对应镜像的，所以直接拷贝官网的staled

  ![](/images/rancher_not_found.jpeg)

正常情况下，是能够直接安装并启动成功的（之前在debian12系统上这样就能直接安装成功）；但是在Rocky Linux 9.3系统上，拉取镜像后启动容器会失败，docker logs 容器id查看相关错误信息如下：

  ![](/images/rancher_ps.jpeg)

  ![](/images/rancher_error.jpeg)

从rancher论坛上查询相关错误问题信息，得知是
>由于较新版本的docker使用了cgroupv2引起的，使用cgroupv2本身并没有错，一些今年发布的Linux发行版也都开始切换到cgroupv2，这也是大势所趋。不过，Rancher server的docker镜像，还要考虑历史兼容的问题，所以server内置的k3s还不兼容cgroupv2的方式。

>Cgroup（Control Group）是Linux内核提供的一个重要的资源管理工具，用于在容器化环境中对进程进行资源限制和控制。它可以将一组进程组织在一个层次结构中，并通过资源控制机制对每个层次的进程进行限制和分配资源。Cgroup能够控制CPU、内存、磁盘I/O等资源的使用情况，确保容器中的进程不会无限制地占用系统资源。Docker作为一个流行的容器化平台，也使用了cgroup来进行资源管理。

查看

```bash
  mount | grep cgroup 
```

得到如下信息：cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)

rocky linux9.3 默认确实是cgroup v2版本

论坛相关人员提出解决方式可以将docker的cgroup版本切回到v1，但是我在rocky linux上试了无效。具体流程如下：

>启动cgroup2的方式，如果系统内核版本 > 4.5 ，那么就可以通过内核参数的形式启用 cgroups v2 ，不过需要注意的是，部分未默认开启的发行版强制开启后，仓库内的软件可能存在不兼容的问题。对于使用 systemd 引导的系统，可以在引导文件 /etc/default/grub 中添加如下一行，启用 v2 版本。GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=yes"

```bash
  uname -r # 查看系统内核版本
  5.14.0-362.8.1.el9_3.x86_64
```
rocky linux9.3内核版本为5.14.0-362.8.1.el9_3.x86_64

打开引导文件/etc/default/grub，在GRUB_CMDLINE_LINUX_DEFAULT项追加：systemd.unified_cgroup_hierarchy=0

```bash
  vim /etc/default/grub
  退出保存再执行
  /usr/sbin/grub2-mkconfig -o /boot/grub2/grub.cfg
```

reboot重启系统,执行mount | grep cgroup 再次查看，cgroup版本

  ![](/images/cgroupv1.jpeg)

再次执行docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:stable，结果还是会报上面的“Waiting for server to become available: Get "https://127.0.0.1:6443/version?timeout=15m0s": dial tcp 127.0.0.1:6443: connect: connection refused”错误。


最终在git hub上找到了相关问题，并提出了一个方法，试了下确实可以解决，链接如下：
>https://github.com/rancher/rancher/issues/36238

先停止掉启动失败的容器

  ![](/images/rancher_error_stop.jpeg)

  ![](/images/rancher_error_solution.jpeg)

  不需要安装iptables,只需要将缺失的iptables模块加载到内核中：

```bash
  sudo cat<<EOF >/etc/modules-load.d/modules.conf
iptable_nat
iptable_filter
EOF
``` 

然后reboot重启系统，再次执行docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:stable，等候几十秒后，成功启动rancher容器。

打开浏览器，访问地址https://192.168.56.110  (192.168.56.110我虚拟机的ip地址)，看到如下页面：

  ![](/images/rancher_succ.jpeg)

打开终端，输入图中圈着的命令，将container-id替换为实际的容器id

  ![](/images/rancher_first_passwd.jpeg)

将得到的密码复制到rancher的登陆密码框中登陆即可，然后重新设置自己的新密码。至此，rancher2.7.9安装成功。下一步导入之前创建好的k8s集群即可。

下面是具体的导入步骤：

  rancher集群管理界面进入导入

  ![](/images/k8s_import1.jpeg)

  选择通用

  ![](/images/k8s_import2.jpeg)

  输入集群相关名称信息，点击创建

  ![](/images/k8s_import3.jpeg)  

  在k8s集群master节点输入如下命令，将k8s集群导入到rancher中

  ![](/images/k8s_import4.jpeg)    

  等待k8s拉取相关镜像，通过kubectl命令查看pod状态，与rancher相关的pod都是running状态的时候就代表导入成功了。

  ![](/images/k8s_import5.jpeg)  


  回到rancher界面，查看集群此时的状态，应该就是active,至此导入k8s集群到rancher成功。

  ![](/images/k8s_import6.jpeg)    

  接下来就可以在rancher中管理k8s集群了。

  ![](/images/k8s_import7.jpeg)      

进入到集群内部，左侧的工作负载下面的菜单分别按照名称对应k8s中的定时任务，守护进程，有状态应用，无状态应用等；服务发现对应k8s中的NodePort，Ingress等；存储对应k8s中的ConfigMap，存储卷等。
