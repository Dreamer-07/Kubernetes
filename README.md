# Kubernetes 

> 学习目标：对于开发来说会使用 K8S 部署项目即可，对于运维，这篇文章并不适合

## 介绍



### 项目部署不同阶段的演变

![image-20220105193750657](README.assets/image-20220105193750657.png)

1. 传统部署：多个应用一起部署在一个服务器上，由于没有做隔离，可能就会导致一个崩其他几个也崩
2. 虚拟化部署
   - 在一个物理机上，通过虚拟化技术开多个虚拟机，不同应用部署在不同虚拟机上
   - 但每个虚拟机之间都是一个**操作系统**，浪费了部分资源

3. 容器化：
   - 与虚拟化类似，但是**共享了操作系统**
   - 可以保证每个容器拥有自己的文件系统，CPU，内存，进程空间等
   - 运行应用程序需要的资源被容器包装，并和底层基础结构解耦
   - ?容器化应用程序可以跨云服务商，跨 Linux 系统发行版进行部署

容器化存在的问题：

- 一个容器故障停机后，如何自动让另外一个容器启动实现替补停机的容器
- 当并发访问量大的时候，怎么样做到横向扩展容器数量

为了解决上述问题 -> **容器编排系统：** 

- Swarm: Docker 官方的编排工具
- Mesos: Apache 提供的，需要和 Marathon 使用
- K8S: Google 开源

Swarm / Kubernetes  前者适合在服务器不多(例如就五六台)的情况下使用，而后者就适合服务器超过10台以上的环境中使用

- 编：将多个相同微服务容器集中管理
- 排：可以动态的扩缩容(例如使用一个命令就可以多启动一个相同的微服务模块容器)

### 简介

![image-20220517232943381](README.assets/image-20220517232943381.png)

- Kubernets 是本质是**一组服务器集群**，可以在集群的每个节点上运行时特定的程序，来对节点中的容器进行管理
- 目的就是实现**资源管理的自动化**
- 主要功能
  1. 自动修复：一旦某一个程序崩溃，能够迅速的启动新的容器
  2. 弹性伸缩：可以根据需求，自动对集群中正在运行的容器数量进行调整
  3. 服务发现：服务可以通过自动发现的形式找到它需要的依赖
  4. 负载均衡：如果一个服务启动了多个容器，能够自动实现请求的负载均衡
  5. 版本回退：如果新发布的程序版本有问题，可以立即回退到原来的版本
  6. 存储编排：根据容器自身的需求自动创建存储卷

![image-20220517234228913](README.assets/image-20220517234228913.png)

总结：Kubernetes 提供了一个**可弹性运行**的分布式系统框架，Kubernetes 会满足你扩展的要求 / 故障转移 / 部署模式 等。例如 Kuberntes 可以轻松管理系统的 Canary(金丝雀，也成为灰度) 部署

### 组件

> 介绍

- 一个 Kubernetes 集群主要是由 **控制节点(master)** & **工作节点(worker)** 构成，每个节点上都会安装不同组件


- **master：集群的控制平面，负责集群的决策**

  - ApiServer：资源操作的唯一入口，接受用户输入的命令，提供认证，授权，API注册和发现等机制

  - Scheduler: 负责集群资源调度，按照预定的调度策略将 **Pod** 调度到相应的 node 节点上

  - ControllerManager: 负责维护集群的状态，比如：程序部署安排，故障检测，自动扩展和滚动更新等

  - Etcd：负责存储集群中各种资源对象的信息

- worker：集群的数据平面，负责为容器提供运行环境

  - Kubelet:  负责维护容器的生命周期，即通过 Docker，来创建，更新，销毁容器
  - KubeProxy：负责提供集群内部的服务发现和负载均衡
  - Docker：容器运行时环境

![k8s组件.jpg](README.assets/1608888638188-86fad61e-547c-43d9-8424-1d17a0c6d5a2.jpeg)



> 举个栗子：K8S 部署一个 Nginx

1. 首先需要明确，一旦 Kubernetes 环境启动之后，master 和 node 都会将自身的信息存储到 **etcd数据库** 中。
2. 一个 Nginx 服务的安装请求首先会被发送到 **master 节点上的 API Server** 组件。
3. API Server 组件会调用 **Scheduler** 组件来决定到底应该把这个服务安装到那个 node 节点上。此时，它会从 etcd 中读取各个 node 节点的信息，然后按照一定的算法进行选择，并将结果告知 API Server 。
4. API Server **调用 Controller-Manager 去调用 Node 节点安装 Nginx 服务**。
5. **kubelet 接收到指令**后，会通知 Docker ，然后由 **Docker 来启动一个 Nginx 的 Pod**（Pod 是 Kubernetes 的最小操作单元，容器必须跑在 Pod 中)
6. 一个 Nginx 服务就运行了，**如果需要访问 Nginx ，就需要通过 kube-proxy 来对 Pod 产生访问的代理**，这样，外界用户就可以访问集群中的 Nginx 服务了。

> 补充概念

- Master：集群控制节点，每个集群至少有一个 Master 节点来负责集群的管理
- Node：工作负载节点，由 Master 分配容器到这些 Node 工作节点上，然后 Node 节点上的 Docker 负责容器的运行
- Pod：**K8S 的最小控制单元，容器都是运行在 Pod 中的，一个 Pod 中可以有一个或多个容器**
- Controller：控制器，通过它来**实现对 Pod 的管理**，比如启动 Pod 、停止 Pod 、伸缩 Pod 的数量等等
- Service：**Pod 对外服务的统一入口**，其下面可以**维护同一类的多个 Pod**
- Label：标签，用于**对 Pod 进行分类**，同一类 P CCCod 会拥有相同的标签
- NameSpace：命名空间，用来**隔离 Pod 的运行环境。**

## 环境搭建

### 规划

#### 集群类型

Kubernetes 的集群一般分为两种：**一主多从** 和 **多主多从**

- 一主多从：一台 Master + N 台 Node 节点，搭建简单，但是有单机故障风险，适合用于测试环境
- 多主多从：多态 Master + 多态 Node 节点，搭建麻烦，安全性高，适合用于生产环境

![集群搭建类型.png](README.assets/1609119870944-20c07dd5-9a76-42c6-b1df-6cc206e516e6.png)

tips：这里主要是面向开发人员，学习如何使用 K8S 部署项目，搭建的就选择一主多从就好了(运维的同学就需要自己找找其他的文档咯)

#### 安装方式

1. minikube：快速搭建单节点的 K8S
2. **kubeadm**：快速搭建集群的 K8S
3. 二进制包：从官网上下载每个组件的二进制包，依次去安装，此方式对于理解kubernetes组件更加有效(开发同学不需要管，运维同学可以在学完之后回来搞)

#### 主机规划

| 角色   | IP地址          | 操作系统                   | 配置                    |
| ------ | --------------- | -------------------------- | ----------------------- |
| Master | 192.168.102.100 | CentOS7.8+，基础设施服务器 | 2核CPU，2G内存，50G硬盘 |
| Node1  | 192.168.102.101 | CentOS7.8+，基础设施服务器 | 2核CPU，2G内存，50G硬盘 |
| Node2  | 192.168.102.102 | CentOS7.8+，基础设施服务器 | 2核CPU，2G内存，50G硬盘 |

需要在每台服务器上都安装 Docker（18.06.3）、kubeadm（1.18.0）、kubectl（1.18.0）和kubelet（1.18.0）。

### 环境初始化

> **三台机器都要**，先跟着操作，具体细节之后就懂了

1. 关闭防火墙(生产环境禁止)

   ```shell
   systemctl stop firewalld
   systemctl disable firewalld
   ```

2. 设置主机名

   ```shel
   hostnamectl set-hostname 主机名
   ```

3. 设置主机名解析(企业中推荐使用内部的 DNS 服务器)

   ```shell
   cat >> /etc/hosts << EOF
   192.168.18.100 k8s-master
   192.168.18.101 k8s-node1
   192.168.18.102 k8s-node2
   EOF
   ```

   ip 地址记得改

4. 时间同步

   ```shel
   yum install ntpdate -y
   ntpdate time.windows.com
   ```

5. 关闭 selinux

   ```shel
   setenforce 0
   sed -i 's/enforcing/disabled/' /etc/selinux/config
   ```

6. 关闭 Swap 分区(也可以不关闭，但需要额外配置，要自己查咯)

   ```shel
   sed -ri 's/.*swap.*/#&/' /etc/fstab
   ```

7. 将桥接的IPv4流量传递到iptables的链

   ```shel
   cat > /etc/sysctl.d/k8s.conf << EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   vm.swappiness = 0
   EOF
   ```

   ```she
   # 加载br_netfilter模块
   modprobe br_netfilter
   ```

   ```shel
   # 查看是否加载
   lsmod | grep br_netfilter
   ```

   ```shel
   # 生效
   sysctl --system
   ```

8. 开启 ipvs：

   在 kubernetes 中 service 有两种代理模型，一种是基于 iptables，另一种是基于 ipvs 的。ipvs的性能要高于iptables的，但是如果要使用它，需要手动载入ipvs模块。

   ```shel
   yum -y install ipset ipvsadm
   ```

   ```shel
   cat > /etc/sysconfig/modules/ipvs.modules <<EOF
   #!/bin/bash
   modprobe -- ip_vs
   modprobe -- ip_vs_rr
   modprobe -- ip_vs_wrr
   modprobe -- ip_vs_sh
   modprobe -- nf_conntrack_ipv4
   EOF
   ```

   ```shel
   chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules
   ```

   ```shel
   lsmod | grep -e ip_vs -e nf_conntrack_ipv4
   ```

   ![image-20220521200839951](README.assets/image-20220521200839951.png)

9. 重启

   ```shell
   reboot
   ```

### 准备基础环境

> 三台服务器都要

#### Docker

1. 使用镜像源下载 docker

   ```shel
   wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
   ```

2. 指定版本安装 Docker

   ```shel
   yum -y install docker-ce-18.06.3.ce-3.el7
   ```

3. 启动并设计开机自启

   ```shel
   systemctl enable docker && systemctl start docker
   ```

4. 设置 Docker 镜像加速器

   ```shel
   sudo mkdir -p /etc/docker
   sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "exec-opts": ["native.cgroupdriver=systemd"],	
     "registry-mirrors": ["https://du3ia00u.mirror.aliyuncs.com"],	
     "live-restore": true,
     "log-driver":"json-file",
     "log-opts": {"max-size":"500m", "max-file":"3"},
     "storage-driver": "overlay2"
   }
   EOF
   ```

5. 重启

   ```shel
   sudo systemctl daemon-reload
   ```

   ```she
   sudo systemctl restart docker
   ```

#### Kubeadm, Kubelet, Kubectl

1. 切换为阿里云的 YUM 软件源

   ```shell
   cat > /etc/yum.repos.d/kubernetes.repo << EOF
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   ```

2. 指定版本号安装

   ```shel
   yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
   ```

3. 为了实现 Docker 使用的 cgroup drvier 和 kubelet 使用的 cgroup drver 一致，建议修改"/etc/sysconfig/kubelet"文件的内容

   ```shel
   vim /etc/sysconfig/kubelet
   ```

   ```she
   KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
   KUBE_PROXY_MODE="ipvs"
   ```

4. 设置开启自启(先不用启动，由于没有生成配置文件，集群初始化后自动启动)

   ```shel
   systemctl enable kubelet
   ```

   ```shel
   kubeadm init \
     --apiserver-advertise-address=192.168.102.100 \
     --image-repository registry.aliyuncs.com/google_containers \
     --kubernetes-version v1.18.0 \
     --service-cidr=10.96.0.0/12 \
     --pod-network-cidr=10.244.0.0/16
   ```

### 部署 K8S 的 Master 节点

> 只需要在 Master 节点上运行

```shel
# 由于默认拉取镜像地址 k8s.gcr.io 国内无法访问，这里需要指定阿里云镜像仓库地址
kubeadm init \
  --apiserver-advertise-address=192.168.102.100 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

其中 `--apiserver-advertise-address` 的值修改为当前 Master 服务器的 ip

![image-20220521210148213](README.assets/image-20220521210148213.png)

根据提示消息。在 Master 节点上添加 **kubectl** 工具的配置文件

```shel
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 部署 K8S 的 Node 节点

将在 Master 节点上获取到的命令拿到 Node 节点上运行

```shel
kubeadm join 192.168.102.100:6443 --token mmeohm.z3l655i4ee1ufh6w \
    --discovery-token-ca-cert-hash sha256:9a2bf7f423194b9ddaf47c8b587af9b2abc654d808f3775ceb7ffef1fe7ae876
```

如果令牌过期了

```shel
kubeadm token create --print-join-command
```

在主节点上可以看到 K8S 集群信息

```shel
kubectl get nodes
```

![image-20220522145542274](README.assets/image-20220522145542274.png)

### 网络环境搭建

> 只用在 Master 节点上操作

- K8S 支持多种网络插件(flannel/calico/canal等)，任选一种即可，这里使用 fiannel -> 获取[flannel配置文件](https://lark-assets-prod-aliyun.oss-cn-hangzhou.aliyuncs.com/yuque/0/2021/yml/513185/1609860138490-0ef90b45-9b0e-47e2-acfa-0c041f083bf9.yml?OSSAccessKeyId=LTAI4GGhPJmQ4HWCmhDAn4F5&Expires=1653204747&Signature=hrm8NMt2ECqBcVUv7C1xpf%2FWy6k%3D&response-content-disposition=attachment%3Bfilename*%3DUTF-8%27%27kube-flannel.yml)

- 使用配置文件启动 flannel

  ```shel
  kubectl apply -f kube-flannel.yml
  ```

- 查看部署 CNI 网络进度

  ```shel
  kubectl get pods -n kube-system
  ```

  ![image-20220522151252286](README.assets/image-20220522151252286.png)

- 查看集群状态

  ```shell
  kubectl get nodes
  ```

   ![image-20220522151406366](README.assets/image-20220522151406366.png)

- 查看 Master 节点组件健康状态

  ```shell
  kubectl get cs
  ```

  ![image-20220522151454394](README.assets/image-20220522151454394.png)

- 查看集群健康状态

  ```shel
  kubectl cluster-info
  ```

  ![image-20220522151527700](README.assets/image-20220522151527700.png)

### 环境测试

> 启动一个 Nginx

1. 部署 Nginx

   ```shell
   kubectl create deployment nginx --image=nginx:1.14-alpine
   ```

2. 暴漏端口

   ```shell
   kubectl expose deployment nginx --port=80 --type=NodePort
   ```

3. 查看服务状态

   ```shell
   kubectl get pods,service
   ```

   ![image-20220522151957825](README.assets/image-20220522151957825.png)

4. 访问任意节点的 **32282** 端口

   ![image-20220522152026511](README.assets/image-20220522152026511.png)
