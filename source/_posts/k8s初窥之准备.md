---
title: k8s初窥之准备
date: 2024-01-17 13:15:50
tags:
  - k8s 
  - Rocky Linux
  - virtualbox
---

    摘要:
      最近在学习了解k8s，将学习动手的过程记录下。首先需要一个集群环境，便使用virtualbox模拟集群环境；
      由于centos停更（改为滚动更新的Stream版），所以系统采用Rocky Linux (9.3)。
      
下面是对Rocky Linux的介绍，摘选自Rocky Linux的百度百科。

>Rocky Linux是一个开源、社区拥有和管理、免费的企业Linux发行版，提供强大的生产级平台。旨在与RHEL（Red Hat Enterprise Linux） 100%兼容。可作为CentOS停止维护（改为滚动更新的Stream版）后，RHEL的下游Linux操作系统替代方案，并继承了原CentOS的开源免费特点。

 ### 从Rocky Linux的官方网站下载对应的镜像iso文件，下载Minimal版本，大约1.6G左右。

 > 官网地址：https://rockylinux.org/download

 下载页面如下图

 ![](/images/rocky_linux_download.jpeg)

 ### 使用VirtualBox创建虚拟机并安装Rocky Linux系统

 > k8s对机器有要求，建议最低2vCPU，内存的话笔者设置的3G，硬盘100G。

  ![](/images/k8s_req.jpeg)

  笔者这里准备3台虚拟机，1台用于部署k8s集群的master节点，另外2台作为node节点。对对于需要3台虚拟机，可以装好一台然后通过virtualbox的clone功能来创建另外2台（clone的过程中生成新的网卡MAC地址）。
  
  因为是集群环境，需要环境中的每台机器网络互通，并且还能访问外网。访问外网，此处使用NAT(网络地址转换)模式，集群内部的机器，使用Host-only模式。virtualbox全局对网络的设置如下图; dhcp配置与否，都可以，反正后续会把集群中的机器的ip设置成静态的。

  ![](/images/virtual_host_only_global_cfg.jpeg)

  安装过程很简单，此处就不截图展示了。
  
  ### 安装完之后

  #### 关闭防火墙

  ```bash
    [root@master ~]# systemctl status firewalld # Active显示running代表防火墙开启状态
    [root@master ~]# systemctl stop firewalld # 停止防火墙
    [root@master ~]# systemctl status firewalld # Active显示inactive代表防火墙关闭状态
    [root@master ~]# systemctl disable firewalld # 关闭开启自启防火墙

  ```

  #### 设置hostname

  需要保证集群中的每台机器的hostname都不一样。

  ```bash
    [root@master ~]# hostnamectl set-hostname master #master 为主机名
  ```

  #### 关闭swap

  ```bash
    [root@master ~]# swapoff -a  # 关闭swap，临时
    [root@master ~]# sed -ri 's/.*swap.*/#&/' /etc/fstab # 关闭swap，永久
  ```

  #### 关闭selinux

  ```bash
    [root@master ~]# setenforce 0 # 关闭selinux
    [root@master ~]# sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config ## 将 SELinux 设置为 permissive 模式（相当于将其禁用）
  ```

  或者通过修改/etc/selinux/config文件，将SELINUX的值改为disabled
  

  #### 配置静态ip地址，以便于集群中机器的通信。

>在Rocky9中，丢弃使用了传统的network 而使用新的NetworkManager管理方式,较之前的版本配置方式有所不同

先查看当前网卡的相关信息

>```bash
    [root@master ~]# ip addr
      1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
          link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
          inet 127.0.0.1/8 scope host lo
            valid_lft forever preferred_lft forever
          inet6 ::1/128 scope host 
            valid_lft forever preferred_lft forever
      2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
          link/ether 08:00:27:9a:9d:14 brd ff:ff:ff:ff:ff:ff
          inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
            valid_lft 86056sec preferred_lft 86056sec
          inet6 fe80::a00:27ff:fe9a:9d14/64 scope link noprefixroute 
            valid_lft forever preferred_lft forever
      3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
          link/ether 08:00:27:1a:1d:86 brd ff:ff:ff:ff:ff:ff
          inet 192.168.56.102/24 brd 192.168.56.255 scope global noprefixroute enp0s8
            valid_lft forever preferred_lft forever
          inet6 fe80::a00:27ff:fe1a:1d86/64 scope link noprefixroute 
            valid_lft forever preferred_lft forever
    [root@master ~]# 
```

找到virtualbox的host-only网卡对应的名字是enp0s8，然后进行设置静态ip

>```bash
    [root@master ~]# vim /etc/NetworkManager/system-connections/enp0s8.nmconnection
```

将[ipv4]下的内容修改为如下：

```bash
    [ipv4]
    method=manual # 设置获取IP地址方式auto-->manual
    address1=192.168.56.102/24,192.168.56.1 #设置IP地址为192.168.56.102,掩码为255.255.255.0，可直接紧跟ip地址后写/24，网关地址为192.168.56.1
```

重启网卡配置：
  
```bash
    nmcli c reload
    nmcli c up enp0s8
```

>重新通过ip addr查看ip地址是否设置成功;

> <b>其他2台机器按照同样的流程进行配置。注意ip地址和主机名</b>

#### 开启ipv4转发

>此项非必须，后续rancher相关操作时会用到，所以此处一起设置了

```bash
   [root@master ~]# cat /proc/sys/net/ipv4/ip_forward #查看是否开启 ipv4 转发，结果为 1 则是已开启。
   [root@master ~]# echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
   [root@master ~] sysctl -p # 重新加载配置
```

或者直接编辑/etc/sysctl.conf文件，在文件中添加net.ipv4.ip_forward=1
```bash
   [root@master ~]# vim /etc/sysctl.conf
   [root@master ~] sysctl -p # 重新加载配置
```


配置好3台机器的网络信息后，可在3台机器上通过ping命令互相ping查看通讯是否正常。

笔者这里3台机（192.168.56.102、192.168.56.103、192.168.56.104）配置好的结果如下图：
 ![](/images/cluster_net_cfg.jpeg)




### 安装docker

对3台机器进行docker安装。Rocky Linux兼容了Centos，所以安装docker的方法和Centos一致。详细过程可参考docker官网地址：https://docs.docker.com/engine/install/centos

#### Set up the repository 设置添加docker仓库
Install the yum-utils package (which provides the yum-config-manager utility) and set up the repository.

```bash
  [root@master ~]# yum install -y yum-utils
  [root@master ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### Install Docker Engine 安装docker引擎

>注： 自2024年6月起，docker官方地址已经完全被墙了，国内安装docker需要添加国内的docker仓库源了

```bash 
[root@master ~]# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo //  阿里镜像源

```
    yum安装包指定版本的命令
    yum install [package-name]-[version].[architecture] // 如：yum install docker-ce-27.2.0

```bash
  [root@master ~]# yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin  # 安装docker
  [root@master ~]# systemctl start docker  # 启动docker
  [root@master ~]# systemctl enable docker # 设置docker自启动
  [root@master ~]# usermod -aG docker $USER  # 将当前用户加入docker用户组，这样当前用户就可以直接使用docker命令了
  [root@master ~]# newgrp docker # 更新用户组docker,立即生效
  [root@master ~]# docker version  # 查看docker版本
  [root@master ~]# docker run hello-world  # 测试docker是否安装成功
```


>为了后续下载镜像时方便，可以将docker镜像仓库地址设置为阿里云的镜像仓库，如下：
```bash
  [root@master ~]# vim /etc/docker/daemon.json
                {
                  "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"] // xxxx替换为自己的镜像仓库地址
                }
```

重启docker

```bash
  [root@master ~]# systemctl daemon-reload
  [root@master ~]# systemctl restart docker
```


>注： 自2024年6月起，国内各大docker镜像仓库都不能访问了，拉取镜像需要科学上网；Docker容器和守护进程运行在一个隔离的环境中，默认情况下不会继承主机系统的代理设置，所以需要对其进行设置；docker代理分为docker-client的代理和docker-daemon的代理;

client的代理可直接在/etc/docker/daemon.json文件中直接配置:
```json 
{
  "proxies": {
      "default": {
        "httpProxy":"",
        "httpsProxy"："",
        "noproxy"; "localhost,127.0.0.1,.example.com"
      }
  }
}
```

daemon的代理需要在系统服务中进行配置:

创建一个 Docker 配置文件（如果不存在），并在其中添加代理设置
```bash
[root@master ~]# mkdir  /etc/systemd/system/docker.service.d 
[root@master ~]# vim /etc/systemd/system/docker.service.d/proxy.conf 
```
在文件中添加以下内容：
```bash
[Service]
Environment="HTTP_PROXY=http://proxy_server:port"
Environment="HTTPS_PROXY=http://proxy_server:port"
Environment="NO_PROXY=localhost,127.0.0.1"
```
重启docker即可

```bash
  [root@master ~]# systemctl daemon-reload
  [root@master ~]# systemctl restart docker
  [root@master ~]# docker info // 查看代理配置是否生效
```
