---
title: k8s初窥之实战集群搭建
date: 2024-01-18 12:15:58
tags:
  - k8s 
  - Rocky Linux
  - virtualbox
---


    摘要:
      k8s集群搭建有2种方式，一种是通过二进制方式搭建，另一种是通过kubeadm命令行工具搭建，笔者此次主要是kubeadm命令行工具搭建k8s集群。

> 此次搭建的是k8s 1.28.5版本；官方文档地址：https://v1-28.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### 主机配置
> ip，hostname，防火墙，selinux，swap,docker等配置在前篇k8s初探之准备中已做好。

#### 在3台机的hosts文件写入对应的ip和主机名映射、允许iptables检查桥接流量

```bash
  cat >> /etc/hosts << EOF
192.168.56.102 master
192.168.56.103 node1
192.168.56.104 node2
EOF
```

配置允许iptables 检查桥接流量

```bash
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

  sysctl --system # 立即生效
```


### 安装kubelet、kubeadm、kubectl

    你需要在每台机器上安装以下的软件包：
    kubeadm：用来初始化集群的指令。
    kubelet：在集群中的每个节点上用来启动 Pod 和容器等。
    kubectl：用来与集群通信的命令行工具。
    kubeadm 不能帮你安装或者管理 kubelet 或 kubectl， 所以你需要确保它们与通过 kubeadm 安装的控制平面的版本相匹配。 
    如果不这样做，则存在发生版本偏差的风险，可能会导致一些预料之外的错误和问题。 然而，控制平面与 kubelet 之间可以存在一个次要版本的偏差，
    但 kubelet 的版本不可以超过 API 服务器的版本。 例如，1.7.0 版本的 kubelet 可以完全兼容 1.8.0 版本的 API 服务器，反之则不可以。

  >以上这段话摘自官网，相关图示如下
  
  ![](/images/k8s_install_readme.jpeg)


  ![](/images/k8s_install.jpeg) 

#### 添加 Kubernetes 的 yum 仓库

```bash
    # 此操作会覆盖 /etc/yum.repos.d/kubernetes.repo 中现存的所有配置
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

#### 安装 kubelet、kubeadm 和 kubectl，并启用 kubelet 以确保它在启动时自动启动

```bash
    yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    systemctl enable --now kubelet
```


### 使用 kubeadm 创建集群

> https://v1-28.docs.kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

  ![](/images/k8s_init_master1.jpeg) 

所有机器添加master域名映射，以下需要修改为自己的

```bash
  echo "192.168.56.102  cluster-endpoint" >> /etc/hosts
```

master节点初始化

```bash
    kubeadm init \
  --apiserver-advertise-address=192.168.56.102 \
  --control-plane-endpoint=cluster-endpoint \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.28.5 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

>--apiserver-advertise-address值是master节点的ip地址，--control-plane-endpoint值是master节点的域名，必须和上一步写入hosts文件的值一致，--image-repository值是镜像仓库地址，--kubernetes-version值是k8s版本，--service-cidr值是service的网段，此值不要修改，--pod-network-cidr值是pod的网段，此值不要修改。--service-cidr、--pod-network-cidr、--apiserver-advertise-address三个的值保证各自在不同的网段即可，不要重叠。

如果不出意外的话，就要出意外，意外如图：

  ![](/images/k8s_init_error.jpeg)

查看错误提示信息，container run time is not runing.发现问题是在containerd，于是重新生成/etc/containerd/config.toml，把上面报错的“registry.k8s.io/pause:3.6”修改为阿里云的镜像，如下：

```bash
    containerd config default | sudo tee /etc/containerd/config.toml #重新生成/etc/containerd/config.toml
    sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
    sed -i 's#sandbox_image = "registry.k8s.io/pause:3.6"#sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9"#g' /etc/containerd/config.toml
    systemctl restart containerd
```

执行上述操作后，再次执行kubeadm init命令，等一会后，k8s集群就初始化成功了。


记住上述圈着的几处地方，第一处提示节点初始化成功，然后执行第二处里面的相关命令，然后进入第三处的链接地址，安装网络组件，第四处是worker节点加入集群的命令，需要记录下来。然后开始安装网络组件：

k8s支持很多网络组件，地址如下：https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/


  ![](/images/k8s_net_plugins.jpeg)

此处我安装calico组件，对应的地址从上图链接进入官网首页，然后找到安装地址入口，最终地址如下：https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-more-than-50-nodes

  ![](/images/k8s_calico_install.jpeg)


```bash
    curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O  # 网络不好可以单独下载下来上传到master节点的某个目录

    kubectl apply -f calico.yaml # 在calico.yaml文件目录下执行
```

安装成功如下

  ![](/images/k8s_calico_success.jpeg)

执行 kubectl get nodes命令，查看集群节点情况，如下图

  ![](/images/k8s_master_result.jpeg)

此时，master节点已全部部署成功，可以开始添加worker节点了。

> 注意：高可用部署方式，也是在这一步的时候，使用添加主节点的命令即可

执行上述master节点初始化成功截图的第四处命令，添加worker节点

```bash
  kubeadm join cluster-endpoint:6443 --token kzbiee.o1nfrwwz5x8zshr3 \
        --discovery-token-ca-cert-hash sha256:494621342b7b363815d711ac8f1677760d7e94b5a1fbdd84e042e983bdc94035 
```

不出意外，仍然会报containerd相关的错误

  ![](/images/k8s_init_worker_error.jpeg)

在worker节点仍然执行对containerd问题的相关处理

```bash
    containerd config default | sudo tee /etc/containerd/config.toml #重新生成/etc/containerd/config.toml
    sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
    sed -i 's#sandbox_image = "registry.k8s.io/pause:3.6"#sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9"#g' /etc/containerd/config.toml
    systemctl restart containerd
```

再次执行kubeadm join 操作，即可。成功如下：

  ![](/images/k8s_worker_success.jpeg)

默认的token时效是24小时，加入token失效，需要重新生成token，在master执行如下命令：

>kubeadm token create --print-join-command

回到master节点，执行kubectl get nodes命令，查看worker节点是否成功加入集群，如下图

  ![](/images/k8s_all_node_success.jpeg)

如果显示node1和node2是NotReady状态，需要等待一会。可以执行 kubectl get pods -A 查看所有pod的状态，如果当前所有pod状态都为Running，此时node1和node2的状态就会变为Ready。

  ![](/images/k8s_all_node_pods.jpeg)  

到此，k8s集群已经成功部署完毕。接下来，可以开始使用k8s部署项目了。k8s命令部署应用比较繁琐，接下来将集群导入到rancher(2.7.9)中，使用rancher来对k8s集群进行管理，后续会介绍rancher的相关导入及使用。