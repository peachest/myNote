# kubernetes

[toc]

[Kubernetes Architecture Explained [Comprehensive Guide\] (devopscube.com)](https://devopscube.com/kubernetes-architecture-explained/)

![img](https://devopscube.com/wp-content/uploads/2022/12/k8s-architecture.drawio-1.png)





k8s中，对象指任何一个pods，secets，daemonsets等可创建的对象，资源指对象的api



## 架构

### 控制平面

#### Kube-api server

api服务器向外部暴露api，与客户端使用REST APIs通信。

api服务器与内部组件使用gRPC规范进行通信。

主要作用：

* 暴露API并处理请求
* 身份验证
* 验证验证
* 与etcd数据库通信
* 统筹控制所有节点的组件



#### etcd

etcd是强一致性，分布式的键值对数据库：

* 强一致性：任何节点上的更新会**立刻同步**到其他节点
* 键值对存储：非关系型数据库



etcd存储了所有kubernetes对象的信息，包括：配置、状态、元数据。

例如：每创建一个pod在工作节点中，etcd会新增一项



etcd通过`Watch()`方法允许用户监听数据的变化，以便在数据变化时执行某些操作。

etcd使用gRPC暴露键值对API。apiserver上的gRPC网关可以将http REST 翻译为gRPC请求

etcd所有对象数据放在`/registry`目录下：

```shell
# 格式
/rdtry/对象类型/名空间/对象名/*

# 例子，表示一个在default名空间下名为nginx的pod对象的数据
/registry/pods/default/nginx/*
```



**etcd是控制平面中唯一一个有状态的组件**



#### Kube-scheduler

scheduler负责调度工作节点上的pods对象



将容器部署到一个pods时需要指定**环境需求**，scheduler负责处理**创建pods**的请求，**并选择满足需求的最优工作节点**。工作流描述如下：

| 步骤 | 源组件        | 目的组件   | 请求                                                         |
| ---- | ------------- | ---------- | ------------------------------------------------------------ |
| 1    | 客户端kubectl | api server | 创建pod                                                      |
| 2    | api server    | etcd       | 保存pod状态                                                  |
| 3    | api server    | 客户端     | 响应请求                                                     |
| 4    | scheduler     | api server | 发现未分配的pod，<br />根据算法找到最优的node，<br />发送一个binding事件 |
| 5    | api server    | etcd       | 保存pod状态                                                  |
| 6    | kublet        | api server | 监听绑定的pod                                                |
| 7    | api server    | kublet     | 发送新绑定的pod的数据                                        |
| 8    | kublet        | containerd | 创建pod，启动容器                                            |
| 9    | kublet        | api server | 绑定pod与node                                                |
| 10   | api server    | etcd       | 保存pod状态                                                  |



上述工作流中，发生了三次pod状态保存，应该分别对应pod的不同生命周期状态



scheduler选择最佳pod的方法：筛选与打分

* 筛选：筛选满足运行时资源需求的节点，如果没有节点满足，放入调度队列中等待
  * 大集群中，通常不会遍历所有节点，根据参数`percentageOfNodesToScore`确定遍历多少，默认值为50%。更大的集群可能设置为5%
* 打分：调用多个调度插件对筛选出来的节点进行打分，最终选择最高分的节点。如果存在多个节点有相同的最高分，则任选一个



scheduler有两个阶段：

* 调度周期：选择最佳的pod。对应第4~5步
* 绑定周期：将变化应用到集群中。对应6~10步



可以自定义scheduler，并与原装scheduler在控制平面中共存。并在部署pod时手动指定使用自定义调度器进行调度。

可以为scheduler编写自定义插件



#### Kube Controller 与 Kube Controller Manager

控制器Controller负责监听k8s对象的状态，若与期望状态有偏离时，将对象迁移到期望状态。

**期望的状态通常放在YAML配置文件中**

每种资源对应一个控制器：

* Deployment
* Replicaset
* DaemonSet
* Job
* ConJob
* endpoints
* namespace
* service accounts
* Node



控制器管理器负责管理所有控制器。



#### Cloud Controller Manager

云控制器管理器作为云平台API与k8s集群的中介

云上特有的组件包括：节点，负载均衡，持久化存储



三种主要控制器：负责保证云组件达到期望状态

* node：与云平台的api通信，传递节点相关信息
* route：配置云平台上的网络路由，使得不同节点的pods可以相互通信
* service：为k8s服务部署负载均衡，分配IP地址





### 工作节点

#### Kubelet

每个node必备组件，作为daemon进程运行



kubelet主要作用：

* 将工作节点注册到API server
* 高层运行时：拉取、解压镜像
* 低层运行时：创建、修改、启动、删除容器
* 处理对容器的probe
* 读取pod配置，并创建对应目录以便于挂载数据卷
* 收集并报告节点与pods的状态，需要通过请求api server



用户通过编写podSpec来定义需要运行的容器、及其所需的cpu、内存、环境变量、数据卷等资源

kublet可以通过api server（通常情况）、本地文件、http endpoint等途径获得PodSpecs



kublet读取本地PodSpecs文件创建的pod，即为k8s静态pods，由kublet负责管理而不是API server



kublet使用 CRI gRPC接口与container runtime（cri-o或containerd） 通信

kublet会暴露一个http endpoint来流式输出日志，并为client提供exec会话





#### Kube proxy

一个service objects包括若干个pod，并将这组pods向内或向外暴露。

endpoint object包含了一个service对象下所有pods的所有ip以及端口

endpoint controller维护了一个所有pod的ip地址列表

service controller负责配置pod endpoints到一个service





每个pod endpoint会有一个ip与port，这个service对象会被分配一个clusterIP，且仅能在集群内部使用。

service的**应用场景**：这组pod部署了同一服务，因此需要将这组pod包装为一个service对象，仅暴露一个统一的接口。此时集群内部的组件直接，或者外部的client通过访问node间接访问service对象，以获得服务。



proxy负责**service发现**，以及为service下的所有pod创建路由并进行**负载均衡**。

kube-proxy是运行在所有node上的daemon进程，从api server获得并监听service的clusterIP，以及该service下所有pod的ip与port



proxy的模式：

1. **IPTables**：默认模式。随机选择一个pod进行负载均衡，与pod的链接建立后将一直保持不变，直到结束。
2. **IPVS**：用于超过1000个service的集群，可以提高性能。有多种负载均衡算法可以选择：
   * `rr`：
   * `lc`：
   * `dh`：
   * `sh`：
   * `sed`：
   * `nq`：
3. Kernelspace：仅用于windows系统
4. Userspace：不推荐使用





#### Container Runtime

容器运行所依赖的软件组件。

* 高层运行时：镜像拉取、解压
* 低层运行时：容器生命周期管理



OCI定义了容器的格式以及运行时标准



k8s支持多种容器运行时，（CRI-O，Docker Engine，containerd）

Docker Engine即最初的DockerDaemon，由外到内分割被为三个组件：

* 新的dockerd：镜像创建
* containerd：高层运行时，捐给cncf，cncf内嵌了CRI plugin，因此kublet可以直接访问containerd
* runc：底层运行时

其中containerd即为k8s支持的第三种容器运行时。



CRI是k8s提出的接口，用于屏蔽不同的容器运行时标准，可以用统一的接口调用容器运行时。CRI包括了高层与低层运行时



**kublet使用gRPC，访问CRI APIs来与container runtime进行通信**，对容器进行管理



### 附加组件



#### CNI Plugin

容器网络接口（Container Network Interface，CNI）包括了一个定义了如何编写用于配置Linux容器上的网络接口的插件的规范，以及若干插件支持

即CNI关注于容器间的网络连接与资源分配



CNI plugins应用场景：

* pod网络连接
* pod网络安全与隔离，使用策略进行控制



container runtime通过CRI plugin使用CRI APIs与CNI plugin进行通信，用于配置pod 之间的网络





#### CoreDNS







#### Metrics Server





#### Web UI



## Kubeconfig

Kubeconfig使用YAML文件配置k8s集群，kubectl使用kubeconfig文件的信息，连接到k8s集群API。

**kubeconfig的默认路径为`$HOME/.kube/config`**



与k8s集群建立连接的关键字段：

* `certificate-authority-data`：
* `server`：集群的endpoint
* `name`：集群名称
* `user`：用户账户的名称
* `token`：用户账户的密码







## 使用kubeadm

参考资料：[How To Setup Kubernetes Cluster Using Kubeadm - Easy Guide (devopscube.com)](https://devopscube.com/setup-kubernetes-cluster-kubeadm/)

k8s官方文档：[Bootstrapping clusters with kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)

[《k8s 集群搭建》不要让贫穷扼杀了你学 k8s 的兴趣！ - 掘金 (juejin.cn)](https://juejin.cn/post/6950166816182239246)很全面

```bash
Linux distribution: CentOS-7-x86_64-DVD-2009

Linux kernel: 3.10.0-1160.el7.x86_64(mockbuild@kbuilder.bsys.centos.org)(gccversion 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) )

VMWare: VMware-workstation pro-17
```



**所有节点：**包括master node（即control plane） 以及 worker node

有一些操作是所有节点都需要的，建议在一个base虚拟机上执行，再复制若干份分别作为master节点与node节点，可以减少工作量





### 虚拟机准备

**最新版本最完整的指令请参考脚本**





#### 命令行开机

`sudo systemctl set-default multi-user.target`

#### 添加用户到sudoer

当前用户未添加到sudoer中，无法执行`sudo`，但是可以使用`su`命令

visudo会强制检查sudoer的文件是否语法正确

`/etc/sudoers.d/`下的配置文件会被包含在sudoers中，因此不需要直接修改`/etc/sudoers`文件

```bash
su												# 切换到root用户
touch /etc/sudoers.d/username					# 创建新的配置文件
EDITOR=nano visudo -f /etc/sudoers.d/username	# 指定visudo指令使用nano编辑器打开文件，-f选项指定文件

# 文件中添加一行
username ALL=(ALL) NOPASSWD:ALL
# 保存并退出
```



#### 消除vmware蜂鸣器

1. 在脚本`/etc/rc.d/rc.local`中添加一行`rmmod pcspkr`，表示移除模块
2. 修改脚本权限：`chmod 744 /etc/rc.d/rc.local`，添加可执行权限
3. 运行脚本：`sh /etc/rc.d/rc.local`





修改api-server的地址

```shell
sed -ri "s/advertiseAddress: 1.2.3.4$/advertiseAddress: $(ifconfig ens33 | grep "inet " | awk '{print$2}')/" kubeadm.yaml
```





```sh
containerd config default > /etc/containerd/config.toml
sed -ri 's/SystemdCgroup = false$/SystemdCgroup = true/' /etc/containerd/config.toml
sed -ri 's/sandbox_image = "registry.k8s.io/pause:3.6"$/sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"/' /etc/containerd/config.toml
```





### 脚本

添加sudoer

```shell
su						
echo "<username> ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/<username>
exit
```



配置linux基础环境

```shell
# ./env.sh

# boot in command line mode
# systemctl set-default multi-user.target

# yum install open-vm-tools

# close bell
sh -c "echo 'rmmod pcspkr' >> /etc/rc.d/rc.local"
sh /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local	# enable executing when boot

# time synchromize
yum install ntp
systemctl enable ntpd --now
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# close swap
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab  

# stop firewall service 
systemctl stop firewalld.service
systemctl disable firewalld.service 
systemctl list-unit-files | grep firewalld.service

# stop selinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config


#### config iptable ####
# ref https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
vm.swappiness                       = 0
EOF

# Apply sysctl params without reboot
sysctl --system

# validate
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward


#### install necessary system tool ####
yum install -y yum-utils

#### config yum repo ####
# repo for containerd
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# repo for kubeadm, kubectl, kubelet
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#### download component ####
yum makecache fast # update yum cache

# docker
# yum install -y docker-ce
# systemctl enable --now docker 
# docker version

# containerd 
yum install containerd.io -y	# 
# rm /etc/containerd/config.toml	# ref https://forum.linuxfoundation.org/discussion/862825/kubeadm-init-error-cri-v1-runtime-api-is-not-implemented
containerd config default > /etc/containerd/config.toml
sed -ri 's/SystemdCgroup = false$/SystemdCgroup = true/' /etc/containerd/config.toml
sed -ri 's/sandbox_image = "registry.k8s.io\/pause:3.6"$/sandbox_image = "registry.aliyuncs.com\/google_containers\/pause:3.6"/' /etc/containerd/config.toml
systemctl enable --now containerd
	
# kubeadm
yum install -y kubelet kubectl kubeadm
systemctl enable --now kubelet

# set default config
touch kubeadm.yaml
kubeadm config print join-defaults >> kubeadm.yaml
echo -e '\n---' >> kubeadm.yaml
kubeadm config print init-defaults >> kubeadm.yaml
cat << EOF | tee -a kubeadm.yaml
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
<<EOF
# custom config
sed -ri "s/advertiseAddress: 1.2.3.4$/advertiseAddress: $(ifconfig ens33 | grep "inet " | awk '{print$2}')/" kubeadm.yaml
sed -ri 's/^imageRepository: registry.k8s.io$/imageRepository: registry.aliyuncs.com\/google_containers/' kubeadm.yaml
sed -ri 's/4m0s$/1m0s/' kubeadm.yaml

# config crictl
crictl config runtime-endpoint unix:///run/containerd/containerd.sock
crictl config image-endpoint unix:///run/containerd/containerd.sock

# pull master component
kubeadm config images pull --config kubeadm.yaml
crictl image
```









**最后执行**

```sh
sudo sh env.sh
```



```sh
sudo kubeadm init --config /path/to/kubeadm.yaml
```



`init`成功的话，log信息会提示执行下列命令

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



以及给出了将worker node加入集群的指令

```shell
# token 可以在kubeadm.yaml配置文件中设置
kubeadm  join 192.168.88.140:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:920efe06c68aed1fa9d2f98fa69aef2b03c3a82ae4ecc212fecae32e0d55dfa8 
```







使用`kubectl`查看集群状态

```sh
kubectl cluster-info # 显示kubernetes control plane is running at 

# 使用下列命令查看，或发现node都是notReady或者挂起状态，因为还没有配置CNI网络插件
kubectl get nodes --all-namespaces # 或
kubectl get nodes -A

kubectl get service
```



kubernetes可以使用很多种CNI（[安装扩展（Addons） | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/#networking-and-network-policy)），这里选择flannel

官方安装教程：[flannel-io/flannel: flannel is a network fabric for containers, designed for Kubernetes (github.com)](https://github.com/flannel-io/flannel#deploying-flannel-manually)

```shell
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```











已执行但，未加入正式脚本，未知影响的参考

[wait-control-plane\] Waiting for the kubelet to boot up the control plane as static Pods from di_liufuling14的博客-CSDN博客](https://blog.csdn.net/liufuling14/article/details/120801912)





### deprecated



containerd的沙箱环境依赖pause:3.6镜像，但是最新版kubead依赖pause:3.9，因此需要额外拉取一个pause:3.6 (deprecated)

```shell
# config crictl
crictl config runtime-endpoint unix:///run/containerd/containerd.sock
crictl config image-endpoint unix:///run/containerd/containerd.sock

# pull & tag pause3.6 for containerd sandbox
crictl pull registry.aliyuncs.com/google_containers/pause:3.6
ctr -n=k8s.io images tag registry.aliyuncs.com/google_containers/pause:3.6 registry.k8s.io/pause:3.6

kubeadm config images pull --config kubeadm.yaml
```





尝试使用docker拉取kubeadm相关的组件镜像(deprecated)

```shell
# pull necessary component images for kubeadm
docker pull docker/desktop-kubernetes-apiserver:v1.25.4
docker pull docker/desktop-kubernetes-scheduler:v1.25.4
docker pull docker/desktop-kubernetes-proxy:v1.25.4
docker pull docker/desktop-kubernetes-controller-manager:v1.25.4
docker pull docker/desktop-kubernetes-etcd:3.5.5-0
docker pull docker/desktop-kubernetes-coredns:v1.9.3
docker pull docker/desktop-kubernetes-pause:3.8

# retag images for containerd
docker tag docker/desktop-kubernetes-apiserver:v1.25.4 k8s.gcr.io/kube-apiserver:v1.25.4
docker tag docker/desktop-kubernetes-scheduler:v1.25.4 k8s.gcr.io/kube-scheduler:v1.25.4
docker tag docker/desktop-kubernetes-proxy:v1.25.4 k8s.gcr.io/kube-proxy:v1.25.4
docker tag docker/desktop-kubernetes-controller-manager:v1.25.4 k8s.gcr.io/kube-controller-manager:v1.25.4
docker tag docker/desktop-kubernetes-etcd:3.5.5-0 k8s.gcr.io/etcd:3.5.5-0
docker tag docker/desktop-kubernetes-coredns:v1.9.3 k8s.gcr.io/coredns/coredns:v1.9.3
docker tag docker/desktop-kubernetes-pause:3.8 k8s.gcr.io/pause:3.8

docker save k8s.gcr.io/kube-apiserver:v1.25.4 -o kube-apiserver.tar
docker save k8s.gcr.io/kube-scheduler:v1.25.4 -o kube-scheduler.tar
docker save k8s.gcr.io/kube-proxy:v1.25.4 -o kube-proxy.tar
docker save k8s.gcr.io/kube-controller-manager:v1.25.4 -o kube-controller-manager.tar
docker save k8s.gcr.io/etcd:3.5.5-0 -o etcd.tar
docker save k8s.gcr.io/coredns/coredns:v1.9.3 -o coredns.tar
docker save k8s.gcr.io/pause:3.8 -o pause.tar

ctr -n k8s.io images import kube-apiserver.tar
ctr -n k8s.io images import kube-scheduler.tar
ctr -n k8s.io images import kube-proxy.tar
ctr -n k8s.io images import kube-controller-manager.tar
ctr -n k8s.io images import etcd.tar
ctr -n k8s.io images import coredns.tar
ctr -n k8s.io images import pause.tar
```







