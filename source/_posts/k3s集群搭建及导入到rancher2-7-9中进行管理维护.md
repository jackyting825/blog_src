---
title: k3s集群搭建及导入到rancher2.7.9中进行管理维护
date: 2024-01-29 21:09:51
tags:
  - k8s 
  - Rocky Linux
  - virtualbox
  - centos7.9
  - k3s
  - k8s
---

    摘要:
      之前在rocky linux上学习使用搭建了k8s集群，本以为能用的很丝滑，奈何k8s集群占用的资源太大了，几个虚拟机一开再跑一套k8s集群，本本16G的内存都吃不消了，
      了解到k3s集群占用的资源少，所以又搭建了一个k3s集群，学习一下k3s集群的使用，本文就是k3s集群搭建及导入到rancher2.7.9中进行维护管理的记录。

k3s和k8s的联系及区别，官网具体介绍如下：

>https://docs.rancher.cn/docs/k3s/_index

    K3s 是一个轻量级的 Kubernetes 发行版，它针对边缘计算、物联网等场景进行了高度优化。K3s 有以下增强功能：

    打包为单个二进制文件。
    使用基于 sqlite3 的轻量级存储后端作为默认存储机制。同时支持使用 etcd3、MySQL 和 PostgreSQL 作为存储机制。
    封装在简单的启动程序中，通过该启动程序处理很多复杂的 TLS 和选项。
    默认情况下是安全的，对轻量级环境有合理的默认值。
    添加了简单但功能强大的batteries-included功能，例如：本地存储提供程序，服务负载均衡器，Helm controller 和 Traefik Ingress controller。
    所有 Kubernetes control-plane 组件的操作都封装在单个二进制文件和进程中，使 K3s 具有自动化和管理包括证书分发在内的复杂集群操作的能力。
    最大程度减轻了外部依赖性，K3s 仅需要 kernel 和 cgroup 挂载。 K3s 软件包需要的依赖项包括：
    1.containerd
    2.Flannel
    3.CoreDNS
    4.CNI
    5.主机实用程序（iptables、socat 等）
    6.Ingress controller（Traefik）
    7.嵌入式服务负载均衡器（service load balancer）
    8.嵌入式网络策略控制器（network policy controller）

    适用场景#
    K3s 适用于以下场景：

    1.边缘计算-Edge
    2.物联网-IoT
    3.CI
    4.Development
    5.ARM
    6.嵌入 K8s
    由于运行 K3s 所需的资源相对较少，所以 K3s 也适用于开发和测试场景。在这些场景中，如果开发或测试人员需要对某些功能进行验证，或对某些问题进行重现，
    那么使用 K3s 不仅能够缩短启动集群的时间，还能够减少集群需要消耗的资源。与此同时，Rancher 中国团队推出了一款针对 K3s 的效率提升工具：AutoK3s。
    只需要输入一行命令，即可快速创建 K3s 集群并添加指定数量的 master 节点和 worker 节点。

安装k3s集群要求:

>https://docs.rancher.cn/docs/k3s/installation/installation-requirements/_index

    1.先决条件
    节点不能有相同的主机名。如果您的所有节点都有相同的主机名，请使用--with-node-id选项为每个节点添加一个随机后缀，或者为您添加到集群的每个节点设计一个独特的名称，
    用--node-name或$K3S_NODE_NAME传递。
    2.硬件
    CPU： 最低1核  内存： 最低 512MB（建议至少为 1GB）
    3.网络 
    K3s server 需要 6443 端口才能被所有节点访问


>系统环境 Rocky Linux9.3，采用1主2从的架构

官网安装手册文档如下： https://docs.rancher.cn/docs/k3s/quick-start/_index

### 安装k3s集群

#### 安装master节点
    curl -sfL https://get.k3s.io | sh -

    也可以使用下面的国内镜像，加快下载速度：

    curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -

    检查结果
    k3s check-config # 检查安装是否成功

    得到nodetoken

    cat /var/lib/rancher/k3s/server/node-token

>注： 如果系统是Rocky Linux9.3的话，k3s check-config的结果应该是成功。如果系统是centos7的话，k3s check-config的结果应该是失败，提示如下： 

    (RHEL7/CentOS7: User namespaces disabled; add 'user_namespace.enable=1' to boot command line)

使用下面命令开启

    grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

    重新启动
    reboot


#### 加入worker节点

    curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -

    也可以使用下面的国内镜像，加快下载速度：

    curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken sh -

>设置K3S_URL参数会使 K3s 以 worker 模式运行。K3s agent 将在所提供的 URL 上向监听的 K3s 服务器注册。K3S_TOKEN使用的值存储在你的服务器节点上的/var/lib/rancher/k3s/server/node-token路径下。

官网上提供的流程就是这样的，理论上是可以的，但是实际操作发现，master节点无任何问题，k3s服务也确实能启动。但是加入worker节点时，发现worker节点的k3s-agent服务启动一直卡住，过了很长一段时间，最终失败，错误信息如下：

    # 查看worker节点的k3s-agent服务状态
    systemctl status k3s-agent.service

    Remotedialer proxy error" error="dial tcp 10.0.2.15:6443: c...n refused"

what? 哪儿来的10.0.2.15这个ip了。通过ip addr 查看发现，我虚拟机环境下每台机器的enp0s3网卡(NAT网络地址转换)的ip地址都是10.0.2.15。网卡使用的是NAT网络地址转换模式，为了访问外网必须的。而且手动将enp0s3网卡的ip设置成静态的话，则会导致虚拟机内部的系统无法访问外网。于是查询了很多文档，最终得知<b>k3s集群部署默认是使用eth0网卡</b>,即我这儿的enp0s3网卡

  我虚拟机目前2张网卡，第一张是enp0s3网卡(NAT网络地址转换；访问外网必须)；第二张是enp0s8网卡(hostonly模式，静态ip地址;集群内部互通使用)。

 ![](/images/k3s_ip.jpeg)

 此时，可以使用搜索引擎搜索"k3s 多网卡"相关关键字，或者翻看官网文档的安装选项菜单下的内容，可以通过配置选项作为命令标志和环境变量传递进去来改变相关的网络配置。

  ![](/images/k3s_server1.jpeg)

  ![](/images/k3s_server2.jpeg)

至此重新安装master节点

    curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
    INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC="--advertise-address=192.168.56.121 --node-ip=192.168.56.121" \
    K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
    sh -

 --advertise-address apiserver 用来向集群成员发布的 IP 地址;192.168.56.121 是我的master节点地址

 --node-ip 为节点发布的 IP 地址 (server 内嵌了 agent 进程)

 同样加入worker节点也需要指定节点发布的ip

   ![](/images/k3s_worker1.jpeg)

    curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    K3S_URL=https://192.168.56.121:6443 \
    K3S_TOKEN=nodetoken \
    K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
    INSTALL_K3S_EXEC="--node-ip=192.168.56.122" \
    sh -

nodetoken 换成cat /var/lib/rancher/k3s/server/node-token 得到的token值

192.168.56.122 是我第一台worker节点的ip地址

等待差不多1分钟，就安装完成，并且k3s-agent服务启动成功。

同样将上述命令中的节点ip稍作修改，在第二台worker节点上执行即可。

此时在k3s server节点上，可以使用kubectl命令查看集群状态

```shell
  kubectl get nodes 或者 k3s kubectl get nodes
```

结果如下：
   ![](/images/k3s_cluster.jpeg)

>默认集群创建成功后，worker节点的roles值为none。我图中的worker节点 roles值是我通过kubectl命令手动修改的。在k3s server节点上，可以使用kubectl命令修改worker节点的roles值为worker 

```shell
  kubectl label no node3(节点机器的hostname) kubernetes.io/role=worker(设置的值)
```


### rancher2.7.9导入k3s集群

>rancher 导入k3s的集群跟之前导入k8s的集群流程一模一样。相关流程在上一篇笔记中已经写过，这里不再重复。这儿展示下导入成功后的效果。

![](/images/rancher_k3s_import_succ.jpeg)

![](/images/rancher_k3s_import_succ1.jpeg)

部署应用相关的流程跟k8s集群的操作一模一样。至此，本本在16G内存的上也能愉快的学习使用k3s(k8s)集群了。