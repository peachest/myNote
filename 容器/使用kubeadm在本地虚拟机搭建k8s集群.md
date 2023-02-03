# kubernets集群搭建

[toc]

## 环境说明

本次集群搭建使用的环境如下表：

| 软件        | 版本                   | 版本                     |
| ----------- | ---------------------- | ------------------------ |
| Linux       | CentOS-7               | CentOS-7-x86_64-DVD-2009 |
| 虚拟机软件  | VMware Workstation pro | 17                       |
| 容器运行时  | containerd             | 1.6.16                   |
| k8s部署工具 | kubeadm                | 1.26.1                   |



:warning: 补充说明：

* 节点（虚拟机）最低配置为2核2G

* 为了节省节点基础环境配置与镜像拉取等操作反复进行，可以在单个节点上完成，并使用VMware的虚拟机**克隆**操作生成多个节点

* 节点要求不可以有重复主机名、MAC地址与product_uuid

  * VMware克隆时**不会**重新生成MAC地址，需要手动重新生成
    * 手动生成MAC地址：关闭虚拟机并选中——虚拟机设置——硬件——网络适配器——高级——MAC地址——生成
  * VMware克隆时**会**重新生成product_uuid
    * 使用命令 `sudo cat /sys/class/dmi/id/product_uuid`，查看`product_uuid`

* 确保启用下列所有端口，使k8s集群中的各组件能相互通信。了解下列端口是连接到哪个目的组件，也有助于帮助debug。详情参考官方文档：[端口和协议](https://kubernetes.io/zh-cn/docs/reference/networking/ports-and-protocols/)

  * Master节点：

    | 端口      | 目的                    |
    | --------- | ----------------------- |
    | 6443      | Kubernetes API server   |
    | 2379-2380 | etcd server client API  |
    | 10250     | Kubelet API             |
    | 10259     | kube-scheduler          |
    | 10257     | kube-controller-manager |

  * Worker节点

    | 端口        | 目的               |
    | ----------- | ------------------ |
    | 10250       | Kubelet API        |
    | 30000-32767 | NodePort Services† |

* 关闭防火墙

* 关闭SELinux

* 禁用交换分区

​	

## 搭建基础环境

### 创建虚拟机

######







CentOS 7的镜像文件可以从阿里云的镜像仓库上获得，链接如下[centos-7-isos-x86_64安装包下载_开源镜像站-阿里云 (aliyun.com)](https://mirrors.aliyun.com/centos/7/isos/x86_64/?spm=a2c6h.25603864.0.0.78ba4511ukVWTX)

仓库中有四种镜像可以选择：

1. DVD：普通安装
2. Everything：完整安装
3. Minimal：最小安装
4. NetInstall：网络安装



本次集群搭建使用**标准安装**，即`CentOS-7-x86_64-DVD-2009`。如果直接下载`.iso`文件慢，也可以选择选择使用`.torrent`种子文件进行安装，速度挺快的。

标准安装集成了绝大部分的常用命令，本次集群搭建过程中基本不需要安装新的命令。**如果使用最小安装，可能后续过程缺乏一些必要的命令，需要用`yum`安装**



省略VMware创建虚拟机的过程。



### 添加sudoer

VMWare使用CentOS 7镜像文件配置虚拟机时，会创建root用户以及一个普通用户。但是不会将该普通用户加入到sudoer中，因此为了使用sudo命令，需要手动将该普通用户加入到sudoer配置中。





`sudoer`的配置文件在`/etc/sudoers`。

可以使用`visudo`来访问该文件，但`visudo`只能在root模式下才能执行。保存时，visudo会进行语法检查，以确保文件语法正确。

visudo默认使用vim来打开文件，可以添加行内环境变量`EDITOR`，以更改使用的编辑器。

```bash
su
EDITOR=nano visudo	# 使用nano进行编辑

## 在文件中添加一行，并将<yourUserName>替换为你的用户名 #
<yourUserName> ALL=(ALL) NOPASSWD:ALL
###################################################

exit
```



为了避免破坏`/etc/sudoers`文件，**最佳实践**是添加一个新的sudoer配置文件到`/etc/sudoers.d/`目录下，且文件名与用户名一致。系统会将查看该目录下的所有配置文件。

```bash
su
# 直接使用$USER获得su之前的用户名，而非root
echo "$USER ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/$USER
exit
```



### 关闭蜂鸣器

CentOS 7会在命令补全失败、对空行进行退格等情况下，发出蜂鸣声进行提示。但是该声音非常烦人，尤其在命令行模式下。

实现该功能由内核模块`pcspkr`实现，模块全称为`pc speaker`，可以通过卸载该功能实现关闭蜂鸣器

```bash
# 立即卸载
rmmod pcspkr

# 由于模块位于内核中，每次开机都会重新加载
# 将模块加入黑名单，避免开机时自动加载
echo "blacklist pcspkr" > /etc/modprobe.d/nobeep.conf
```



### 同步时间

集群搭建过程中经常失败，debug时经常需要查看日志。

而VMware使用CentOS 7镜像创建虚拟机时，使用的是默认的时区（美国洛杉矶），造成日志中的时间与真实时间难以对应，增加了debug的难度。因此需要设置正确的系统时间。



```bash
# 设置时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 若ntp指令不存在
# yum install -y ntp

# 使用ntp进行时间同步。并设置开机自启
systemctl enable --now ntpd
```



### 禁用swap分区

为了保证 kubelet 正常工作，你**必须**禁用交换分区。参考官方文档：[安装 kubeadm——准备开始](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#准备开始)

```bash
# 立刻卸载
swapoff -a

# 禁用开机自动挂载swap分区。注释了/etc/fstab中挂载swap的行
sed -ri 's/.*swap.*/#&/' /etc/fstab
```



在虚拟机环境中也可以修改内核参数`vm.swappiness`，该参数表示虚拟机开始使用swap分区时**内存未使用率**，CentOS7中默认`vm.swappiness=30`（可以使用指令`sysctl vm.swappiness`查看），表示当内存使用率达到$70\%$时，开始使用swap分区。

但是即使将该参数设置未0，但由定义可知只是尽可能不适用swap分区，**而非完全禁用swap分区**，swap分区仍然被挂载到系统中。因此**也许会导致集群崩溃**。

```bash
# 设置虚拟机内核参数swappiness为0并写入配置文件，确保开机自动加载
cat <<EOF >> /etc/sysctl.d/k8s.conf
vm.swappiness = 0
EOF
# 立即加载参数
sysctl -p
```



检查是否成功卸载swap分区。若成功，则swap分区的总量total应为0

```bash
free -m
```



### 禁用防火墙

```bash
# 立即停止防火墙
systemctl stop firewalld.service
# 禁止开机自启
systemctl disable firewalld.service
```



检查防火墙服务是否被禁止开机自启。若成功，状态应为**disabled**

```bash
systemctl list-unit-files | grep firewalld.service
```



### 禁用SELinux

禁用SELinux是允许容器访问主机文件系统所必需的，而这些操作是为了例如 Pod 网络工作正常。详情参考官方文档：[安装kubeadm——安装kubeadm、kubelet和kubectl——基于Red Hat的发行版](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

下列命令可以有效禁用SELinux

```bash
setenforce 0

# 将SELinux的状态从enforcing修改为permissive
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```



### 设置IP路由规则

参考官方文档：[容器运行时——转发IPv4并让iptables看到桥接流量](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#转发-ipv4-并让-iptables-看到桥接流量)

```bash
# 安装依赖模块
modprobe overlay
modprobe br_netfilter

# 设置模块开机自启
cat <<EOF >> /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# sysctl params required by setup, params persist across reboots
cat <<EOF >> /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sysctl --system
```



验证模块是否安装成功。若成功，则输出非空

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```



验证参数是否设置成功。若成功则显示参数

```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables
net.ipv4.ip_forward
```



### 安装容器运行时

k8s可以使用满足CRI标准的容器运行时。可选的容器运行时有：

* dockerd
* containerd
* cri-o

本次集群搭建使用 containerd作为K8s集群的容器运行时。



#### 安装containerd

containerd原本是docker后端使用的运行时，因此可以到docker-ce的仓库中下载。

使用阿里云的镜像源下载并安装containerd

```bash
# 使用yum-tuils中的yum-config-manager编辑yum源的配置。若命令不存在，则需要下载
# yum install -y yum-utils

# 添加docker-ce仓库的阿里云镜像源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 换源后必须更新yum的缓存，否则安装失败
yum makecache fast

# 注意.io后缀。
yum install -y contianerd.io
```



#### 配置containerd

直接启动containerd会使用其默认配置，但是我们需要修改其默认配置，因此首先需要获得containerd的默认配置并写成配置文件，再在配置文件的基础上修改

```bash
# 获得默认配置，并写入配置文件
containerd config default > /etc/containerd/config.toml
```





当前最新版containerd默认依赖pause:3.6镜像以创建沙箱环境，而kubeadm在初始化过程中依赖pause:3.9镜像。为了确保环境一致，以及避免重复拉取镜像，可以修改containerd依赖的pause版本。

```bash
# 将默认registry.k8s.io下的pause:3.6镜像
# 修改为registry.aliyuncs.com/google_containers仓库下的pause:3.9镜像
sed -ri 's/sandbox_image = "registry.k8s.io\/pause:3.6"$/sandbox_image =
"registry.aliyuncs.com\/google_containers\/pause:3.9"/'
/etc/containerd/config.toml
```



同时还需要将containerd默认配置中的`SystemdCgroup`字段，由`false`修改为`true`，以确保使用`systemd`的cgroup驱动。参考官方文档：

* [容器运行时——systemd cgroup驱动](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#systemd-cgroup-driver)

  >关键的一点是 kubelet 和容器运行时需使用相同的 cgroup 驱动并且采用相同的配置。
  >
  >
  >
  >如果你将 `systemd` 配置为 kubelet 的 cgroup 驱动，你也必须将 `systemd` 配置为容器运行时的 cgroup 驱动。

* [容器运行时——containerd——配置systemd cgroup](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd-systemd)

  >结合 `runc` 使用 `systemd` cgroup 驱动，在 `/etc/containerd/config.toml` 中设置

```bash
# 启用systemd cgroup配置
sed -ri 's/SystemdCgroup = false$/SystemdCgroup = true/'
/etc/containerd/config.toml
```



#### 启动containerd

```bash
# 设置开机自启，并立即启动containerd服务
systemctl enable --now containerd
```



启动成功后，可以查看是否存在文件`/var/run/containerd/containerd.sock`或`/run/containerd/containerd.sock`

即containerd**服务端点**为：

* `unix:///var/run/containerd/containerd.sock`
* 或者`unix:///run/containerd/containerd.sock`

这两个端点本质一样，前者是后者的一个软链接而已。



### 安装kubeadm，kubectl，kubelet



#### 安装

```bash
yum install -y kubelet kubectl kubeadm
```



#### 配置kubectl

参考官方文档：[在 Linux 系统中安装并设置 kubectl](https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/)

Kubernetes 提供的 [kubectl](https://kubernetes.io/zh-cn/docs/reference/kubectl/kubectl/)是使用 Kubernetes API 与 Kubernetes 集群的控制面（master节点）进行通信的命令行工具。







#### 配置kubelet

启用kubelet的自动补全功能。参考官方文档：[在 Linux 系统中安装并设置 kubectl——启用shell自动补全功能](https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/#introduction)

先安装依赖bash-completion

```bash
yum install -y bash-completion
```





#### 配置kubeadm







参考官方文档：

* [安装kubeadm——安装kubeadm、kubelet和kubectl——基于Red Hat的发行版
* 

[配置 cgroup 驱动——配置kubelet的cgroup驱动](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/#配置-kubelet-的-cgroup-驱动)

> 在版本 1.22 中，如果用户没有在 `KubeletConfiguration` 中设置 `cgroupDriver` 字段， `kubeadm` 会将它设置为默认值 `systemd`。
>
> 
>
> 这是一个最小化的示例，其中显式的配置了此字段。
>
> 这样一个配置文件就可以传递给 kubeadm 命令了。





```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```





## 部署Master节点



？kubelet会在`kubeadm init`执行成功后，正常运行





### 清理Master节点

如果`kubeadm init`失败了，需要清理节点后才能再次执行`init`命令，否则`init`过程会因为存在上一次`init`时创建的某些文件而报错。

`kubeadm reset`命令会尽可能清理节点。

* 添加`-f`选项以省略中间的确认步骤。
* 如果`init`时使用了配置文件，可以使用`--kubeconfig`选项指定



完整的命令实例如下：

```bash
sudo kubeadm reset -f --kubeconfig /path/to/kubeadm.conf
```



## 部署Worker节点





## 常见故障

