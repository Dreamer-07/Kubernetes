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

### Kubectl 命令自动补全

```shel
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo “source <(kubectl completion bash)” >> ~/.bashrc
```

## 资源管理

> 在线 yaml to json：https://www.json2yaml.com/

### 简介

- 在 K8S 中，所有的内容都抽象为资源，用户需要**通过操作资源来管理K8S**
  - K8S 本质是一个集群系统，用户可以在集群中部署各种服务(就是在 K8S 集群中运行容器，并将指定的程序跑在容器中)
  - K8S 的最小管理单元**是 Pod 而不是容器**，所以只能将**容器放在 Pod 中**，而 K8S 一般也不会直接管理 Pod，而是通过 **Pod 控制器** 来管理 Pod
  - K8S 提供了 **Service** 来实现对 Pod 中的服务进行访问
  - 当然，如果 Pod 中程序的数据需要持久化，K8S 还提供了相关**存储系统**

![资源管理介绍.png](README.assets/1609132417114-28600d3c-185f-4ded-9f5e-c0d77fb86d80.png)

学习 K8S 的核心，就是学习如何对集群中的 **Pod/Pod控制器/Service/存储** 等各种资源进行操作

### YAML

#### 介绍

- YAML是一个类似于 XML、JSON 的标记性语言。它强调的是以“数据”为中心，并不是以标记语言为重点。因而YAML本身的定义比较简单，号称是“一种人性化的数据格式语言”。
- YAML的语法比较简单，主要有下面的几个：
  - 大小写敏感。

  - 使用缩进表示层级关系。

  - 缩进不允许使用tab，只允许空格（低版本限制）。

  - 缩进的空格数不重要，只要相同层级的元素左对齐即可。

  - `# ` 表示注释。

- YAML支持以下几种数据类型：
  - 常量：单个的、不能再分的值。

  - 对象：键值对的集合，又称为映射/哈希/字典。
  - 数组：一组按次序排列的值，又称为序列/列表


#### 语法示例

常量

```yaml
#常量，就是指的是一个简单的值，字符串、布尔值、整数、浮点数、NUll、时间、日期
# 布尔类型
c1: true
# 整型
c2: 123456
# 浮点类型
c3: 3.14
# null类型
c4: ~ # 使用~表示null
# 日期类型
c5: 2019-11-11 # 日期类型必须使用ISO 8601格式，即yyyy-MM-dd
# 时间类型
c6: 2019-11-11T15:02:31+08.00 # 时间类型使用ISO 8601格式，时间和日期之间使用T连接，最后使用+代表时区
# 字符串类型
c7: haha # 简单写法，直接写值，如果字符串中间有特殊符号，必须使用双引号或单引号包裹
c8: line1
    line2 # 字符串过多的情况可以折成多行，每一行都会转换成一个空格
```

对象

```yaml
# 对象
# 形式一（推荐）：
xudaxian:	
	name: 许大仙
	age: 16
# 形式二（了解）：
xuxian: { name: 许仙, age: 18 }
```

数组

```yaml
# 数组
# 形式一（推荐）：
address:
	- 江苏
	- 北京
# 形式二（了解）：
address: [江苏,上海]
```

### 资源管理方式

#### 分类

1. 命令式对象管理：直接使用 **命令** 操作 K8S 的资源

   ```shell
   kubectl run nginx-pod --image=nginx:1.17.1 --port=80
   ```

2. 命令式对象配置：通过 **命令+配置文件** 的方式操作 K8S 的资源

   ```shell
   kubectl create -f nginx-pod.yaml
   ```

3. 声明式对象配置：通过 **apply命令+配置文件** 的方式操作 K8S 的资源(只能创建/更新pod)

   ```shell
   kubectl apply -f nginx-pod.yaml
   ```

| 类型           | 操作 | 适用场景 | 优点           | 缺点                               |
| -------------- | ---- | -------- | -------------- | ---------------------------------- |
| 命令式对象管理 | 对象 | 测试     | 简单           | 只能操作活动对象，无法审计、跟踪   |
| 命令式对象配置 | 文件 | 开发     | 可以审计、跟踪 | 项目大的时候，配置文件多，操作麻烦 |
| 声明式对象配置 | 目录 | 开发     | 支持目录操作   | 意外情况下难以调试                 |

#### 命令式对象管理 

##### Kubectl 命令

- Kubectl 是 K8S 集群的命令行工具，通过它能对集群本身进行管理，也能在集群上进行容器化应用的安装和部署

  ```shell
  kubectl [command] [type] [name] [flags]
  ```

  - `command`: 要对资源执行的操作(craete/get/delete等)
  - `type`: 指定资源的类型(deployment/pod/service等)
  - `name`: 指定资源的名称(大小写敏感)
  - `flags`: 额外的可选参数

- 示例：查看所有的 pod

  ```shell
  kubectl get pods
  ```

- 示例：查看一个具体的 pod

  ```shell
  kubectl get pod nginx-55f8fd7cfc-czf64
  ```

- 示例：查看某个 pod，以 yaml 格式显示其信息

  ```shell
  kubectl get pod nginx-55f8fd7cfc-czf64 -o yaml
  ```

##### 操作类型

K8S 允许你对资源进行多种操作，可以通过 `--help` 查看

经常使用的有：

- 基本命令

  | 命令    | 翻译 | 命令作用     |
  | ------- | ---- | ------------ |
  | create  | 创建 | 创建一个资源 |
  | edit    | 编辑 | 编辑一个资源 |
  | get     | 获取 | 获取一个资源 |
  | patch   | 更新 | 更新一个资源 |
  | delete  | 删除 | 删除一个资源 |
  | explain | 解释 | 展示资源文档 |

- 运行和调试

  | 命令      | 翻译     | 命令作用                   |
  | --------- | -------- | -------------------------- |
  | run       | 运行     | 在集群中运行一个指定的镜像 |
  | expose    | 暴露     | 暴露资源为Service          |
  | describe  | 描述     | 显示资源内部信息           |
  | logs      | 日志     | 输出容器在Pod中的日志      |
  | attach    | 缠绕     | 进入运行中的容器           |
  | exec      | 执行     | 执行容器中的一个命令       |
  | cp        | 复制     | 在Pod内外复制文件          |
  | rollout   | 首次展示 | 管理资源的发布             |
  | scale     | 规模     | 扩（缩）容Pod的数量        |
  | autoscale | 自动调整 | 自动调整Pod的数量          |

- 高级命令

  | 命令  | 翻译 | 命令作用               |
  | ----- | ---- | ---------------------- |
  | apply | 应用 | 通过文件对资源进行配置 |
  | label | 标签 | 更新资源上的标签       |

- 其他命令

  | 命令         | 翻译     | 命令作用                     |
  | ------------ | -------- | ---------------------------- |
  | cluster-info | 集群信息 | 显示集群信息                 |
  | version      | 版本     | 显示当前Client和Server的版本 |

##### 资源类型

- K8S 中的所有内容都抽象为资源，可以通过 `api-resources` 查看

- 经常使用的资源：

  - 集群级别资源

    | 资源名称   | 缩写 | 资源作用     |
    | ---------- | ---- | ------------ |
    | nodes      | no   | 集群组成部分 |
    | namespaces | ns   | 隔离Pod      |

  - Pod 资源

    | 资源名称 | 缩写 | 资源作用 |
    | -------- | ---- | -------- |
    | Pods     | po   | 装载容器 |

  - Pod 资源控制器

    | 资源名称                 | 缩写   | 资源作用    |
    | ------------------------ | ------ | ----------- |
    | replicationcontrollers   | rc     | 控制Pod资源 |
    | replicasets              | rs     | 控制Pod资源 |
    | deployments              | deploy | 控制Pod资源 |
    | daemonsets               | ds     | 控制Pod资源 |
    | jobs                     |        | 控制Pod资源 |
    | cronjobs                 | cj     | 控制Pod资源 |
    | horizontalpodautoscalers | hpa    | 控制Pod资源 |
    | statefulsets             | sts    | 控制Pod资源 |

  - 服务发现资源

    | 资源名称 | 缩写 | 资源作用        |
    | -------- | ---- | --------------- |
    | services | svc  | 统一Pod对外接口 |
    | ingress  | ing  | 统一Pod对外接口 |

  - 存储资源

    | 资源名称   | 缩写 | 资源作用 |
    | ---------- | ---- | -------- |
    | configmaps | cm   | 配置     |
    | secrets    |      | 配置     |

##### 应用实例

> 创建&删除 namespace&pod

1. 创建一个 `namespace`

   ```shell
   kubectl create ns dev
   ```

2. 查看一个 `ns(namesapce)` 

   ```shell
   kubectl get ns dev
   ```

3. 在刚刚创建的 `ns` 中创建一个 `pod`

   ```shell
   kubectl run nginx --image=nginx:1.14-alpine -n dev
   ```

4. 查看 `ns` 中 `pod` 的信息

   ```shell
   kubectl get pod nginx -n dev
   ```

5. 删除 `ns` 中的 `pod`

   ```shell
   kubectl delete pod nginx -n dev
   ```

6. 删除 `ns`

   ```shell
   kubectl delete ns dev
   ```

#### 命令式对象配置

- 命令式对象配置：通过 **命令+配置文件** 的方式操作 K8S 的资源

> 应用示例：创建&查看&删除 ns&pod

1. 创建一个 `nginx-pod.yaml`(先用着，里面的配置后面都会学)

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: dev
   ---
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginxpod
     namespace: dev
   spec:
     containers:
       - name: nginx-containers
         image: nginx:1.14-alpine
   ```

2. 执行 `craete` 命令，创建资源(命令+配置文件)

   ```shell
   kubectl create -f nginx-pod.yaml
   ```

3. 执行 `get` 命令，查看配置文件中声明的资源

   ```shell
   kubectl get -f nginx-pod.yaml
   ```

4. 执行 `delete` 命令。删除配置文件中声明的资源

   ```shell
   kubectl delete -f nginx-pod.yaml
   ```

配置文件中就是声明命令需要的各种参数

#### 声明式对象配置

##### 概述

- 通过 `apply`命令+配置文件去操作 K8S 的资源
- 和命令式对象配置类似，只不过它只有一个 `apply` 命令
- `apply` = `create`/`patch`

##### 应用示例

````shell
kubectl apply -f nginx-pod.yaml
````

![image-20220522193416178](README.assets/image-20220522193416178.png)

#### 使用方式推荐

- 创建/更新资源 - 声明式对象配置 - `kubectl apply -f xxx.yaml`
- 删除资源 - 命令式对象管理 - `kubectl delete -f xxx.yaml`
- 查询资源 - 命令式对象管理 - `kubectl get(describe) 资源 `

#### 扩展：在 Node 节点上运行 Kubectl

kubectl的运行需要进行配置，它的配置文件是$HOME/.kube，如果想要在Node节点上运行此命令，需要将Master节点的.kube文件夹复制到Node节点上，即在Master节点上执行下面的操作：

```shell
scp -r $HOME/.kube node节点ip:$HOME
```

## 实战入门

### NameSpace

#### 概述

- 主要作用：实现 `多套系统的资源隔离` / `多租户的资源隔离`
- 默认情况下，K8S 的集群中所有的 Pod 都是可以相互访问的，但我们可以通过将不同的 Pod(资源) 划分到不同的 NameSpace 中，形成逻辑上的"组"，以方便不同的组的资源进行隔离使用和管理
- 可以通过 K8S 的授权机制，将不同的 NS 交给不同的租户进行管理，实现多租户的资源隔离，此时还能结合 **K8S 的资源配额机制**，限定不同租户能占用的资源(CPU,内容使用量等)，来实现对租户可用资源的管理

![Namespace概述.png](README.assets/1609137920951-be890509-fe17-4be4-935b-f4f687745a1b.png)

- K8S 集群在启动之后，会默认创建几个 namesapce

  ```shell
  kubectl get ns
  ```

  ![image-20220523084701705](README.assets/image-20220523084701705.png)

  > default： 所有未指定 NS 的资源都会被分配到该 NS 中
  >
  > kube-node-lease: 集群节点之间的心跳维护(1.13中引入)
  >
  > kube-public: 此命名空间的资源可以被所有用户访问(包括未认证)
  >
  > kube-system: 所有由 K8S系统 创建的资源都处于这个命名空间

#### 使用

- 查看所有的命名空间

  ```shell
  kubectl get namesapce
  ```

   ![image-20220523100210699](README.assets/image-20220523100210699.png)

- 查看指定的命名空间

  ```shell
  kubectl get namespace default
  ```

   ![image-20220523100253744](README.assets/image-20220523100253744.png)

- 指定命名空间的输出格式

  ```shell
  kubectl get namespace default -o yaml/json/wide
  ```

  ![image-20220523100420682](README.assets/image-20220523100420682.png)

- 查看命名空间的详情

  ```shell
  kubectl describe namespace default
  ```

  ```shell
  [root@k8s-master ~]# kubectl describe namespace default
  Name:         default
  Labels:       <none>
  Annotations:  <none>
  Status:       Active  #Active: 命名空间正在使用; Terminating: 正在删除命名空间
  
  No resource quota.	  #针对命名空间做的资源限制
  
  No LimitRange resource. #针对命名空间的每个组件做的资源限制
  ```

- 创建命名空间

  ```shell
  kubectl create namesapce dev
  ```

- 删除命名空间

  ```shell
  kubectl delete ns dev
  ```

- 使用命令式对象配置

  - 新建一个 `ns-dev.yaml`

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: dev
    ```

  - 创建命名空间

    ```shell
    kubectl craete -f ns-dev.yaml
    ```

  - 删除命名空间

    ```shell
    kubectl delete -f ns-dev.yaml
    ```

### Pod

#### 概述

- Pod 是 K8S 集群管理的**最小单元**，程序必须部署到容器中，而容器必须部署在 Pod 中

- Pod 可以认为对容器的封装，一个 Pod 中可以存在一个或多个容器

   ![Pod概述.png](README.assets/1609137223448-715c6ece-0158-4ee2-9efa-fcff15e143ed.png)

- K8S 在集群启动之后，集群中的各个组件也是以 Pod 方式运行的

  ```shell
  kubectl get pods -n kube-system
  ```

   ![image-20220523102734068](README.assets/image-20220523102734068.png)

#### 使用

- 语法：创建并运行 Pod

  ```shell
  kubectl run (Pod的名称) [参数]
  # --image 指定Pod的镜像
  # --port 指定端口
  # --namespace 指定namespace
  ```

- 在名称为 `dev` 的 namesapce 下创建一个 Nginx 的 Pod

  ```shell
  kubectl run nginx --image=nginx:1.14-alpine --port=80 --namespace=dev
  ```

- 查看指定命名空间下的所有的 Pod

  ```shell
  kubectl get pod -n dev
  ```

   ![image-20220523103411426](README.assets/image-20220523103411426.png)

- 查看 Pod 的详细信息

  ```shell
  kubectl describe pod pod的名称 [-n 命名空间名称]
  ```

  ```shell
  kubectl describe pod nginx -n dev
  ```

  ![image-20220523103651958](README.assets/image-20220523103651958.png)

- 访问 Pod 中的容器

  ```shell
  # 获取 pod 的 ip(现在只能对内范围且不是固定的)
  kubectl get pod -n dev -o wide
  ```

  ![image-20220523103803808](README.assets/image-20220523103803808.png)

  ```shell
  # 访问 pod 中的容器
  curl 10.244.1.9:80
  ```

  ![image-20220523103843403](README.assets/image-20220523103843403.png)

- 删除指定的 Pod

  ```shell
  kubectl detele pod nginx -n dev
  ```

- 命令式对象配置

  - 新建一个 `pod-nginx.yaml`

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
      namespace: dev
    spec:
      containers:
      - image: nginx:1.14-alpine
        name: pod
        ports: 
        - name: nginx-port
          containerPort: 80
          protocol: TCP
    ```

  - 创建一个 Pod

    ```shell
    kubectl create -f pod-nginx.yaml
    ```

  - 删除一个 Pod

    ```shell
    kubectl delete -f pod-nginx.yaml
    ```

### Label

#### 概述

- 作用：为资源添加标识，用来对它们进行区分和选择

- 特点：

  1. 一个 Label 会以 key/value 键值的形式附加到各种对象上，如 Node/Pod/Service 等
  2. 一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上去
  3. Label 通常在资源对象定义时确定，当然也可以在对象创建后动态的添加和删除

- 可以通过Label实现资源的多纬度分组，以便灵活、方便地进行资源分配、调度、配置和部署等管理工作

  > 一些常用的Label标签示例如下：
  >
  > - 版本标签：“version”:”release”,”version”:”stable”
  >
  > - 环境标签：“environment”:”dev”,“environment”:”test”,“environment”:”pro”
  >
  > - 架构标签：“tier”:”frontend”,”tier”:”backend”

- 需要通过 Label Selector 对标签进行选择：
  - Label 用于对某个资源定义标识
  - Label Selector 用于查询和筛选拥有某些标签的资源对象
- 当前有两种Label Selector

- - 基于**等式**的 Label Selector。
    - `name=slave`：选择所有包含 Label 中的 key=“name” 并且 value=“slave” 的对象。

- - - `env!=production`：选择所有包含 Label 中的 key=“env” 并且 value!=“production” 的对象。

- - 基于**集合**的 Label Selector。

- - - `name in (master,slave)`：选择所有包含 Label 中的 key=“name” 并且 value=“master” 或 value=“slave” 的对象。

- - - `name not in (master,slave)`：选择所有包含 Label 中的 key=“name” 并且 value!=“master” 和 value!=“slave” 的对象。

- 标签的选择条件可以使用多个，此时将多个Label Selector进行组合，使用**逗号（,）**进行分隔即可。

- - name=salve,env!=production。

- - name not in (master,slave),env!=production。

#### 使用

- 为资源动态添加标签

  ```shell
  kubectl label 资源类型 具体资源 [-n 命名空间] label.key=lavbel.value
  ```

  ```shell
  kubectl label pod nginx -n dev version=1.0
  ```

- 查看资源的标签

  ```shell
  kubectl get pod nginxpod -n dev --show-labels
  ```

   ![image-20220523110744139](README.assets/image-20220523110744139.png)

- 更新资源的标签(一个资源不允许拥有两个 key 相同的 label)

  ```shell
  kubectl label pod nginxpod -n dev version=2.0 --overwrite
  ```

  ![image-20220523110759507](README.assets/image-20220523110759507.png)

- 筛选标签

  ```shell
  kubectl get 资源类型 -l 筛选条件 [-n 命名空间] --show-labels
  ```

  ```shell
  kubectl get pod nginxpod -n dev --show-labels
  ```

  ![image-20220523111118433](README.assets/image-20220523111118433.png)

- 删除标签

  ```shell
  kubectl label 资源类型 具体资源 [-n 命名空间] label.key-
  ```

  ```shell
  kubectl label pod nginx -n dev version-
  ```

   ![image-20220523124559091](README.assets/image-20220523124559091.png)

- xl使用命令式对象配置

  - 修改 `pod-nginx.yaml`

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
      namespace: dev
      # 设置标签
      labels:
        version: "3.0"
        env: "test"        
    spec:
      containers:
      - image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        name: pod
        ports: 
        - name: nginx-port
          containerPort: 80
          protocol: TCP
    ```

  - 执行创建

    ```shell
    kubectl apply -f pod-nginx.yaml
    ```

  - 查看标签

    ![image-20220523125205078](README.assets/image-20220523125205078.png)

### Deployment

#### 概述

- 在 K8S 中，Pod 是最小的控制单元，但一般 K8S 都是通过 **Pod控制器** 完成对 Pod 的控制

- Pod 控制器用于 Pod的管理，确保Pod资源符合预期的状态，当Pod的资源出现故障时，会尝试进行重启/重建Pod

- 在 K8S 中的 Pod控制器 种类有很多，这里先介绍一种：Deloyment

   ![Deployment概述.png](README.assets/1609137432883-b2ec0213-7c10-4efa-9689-066a4e239a74.png)

#### 使用

- 创建指定名称的 `deployment`

  ```shell
  kubectl create deployment 名称 [-n 命名空间]
  ```

  ```shell
  # 需要通过 --image 指定 pod 中需要运行的容器
  kubectl create deployment nginx-deploy --image=nginx:1.17.1 -n dev
  ```

- 对指定的 `deployment` 创建指定数量的 pod

  ```shell
  kubectl scale deployment xxx [--replicas=正整数] [-n 命名空间]
  ```

  ```shell
  kubectl scale deployment nginx-deploy --replicas=3 -n dev
  ```

- 查看 `deploy` && `pods`

  ```shell
  kubectl get deploy,pod -n dev
  ```

   ![image-20220523131606035](README.assets/image-20220523131606035.png)

- 查看 `deploy` 的详细信息

  ```shell
  kubectl get deployment nginx-deplpy -n dev
  ```

   ![image-20220523131748624](README.assets/image-20220523131748624.png)

  ```shell
  kubectl describe deploy nginx-deploy -n dev
  ```

  ![image-20220523131853253](README.assets/image-20220523131853253.png)

- 删除 `deploy`

  ```shell
  kubectl delete deployment nginx-deploy -n dev
  ```

- 命令式对象配置

  - 创建一个 `deploy-nginx.yaml`

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx
      namespace: dev
    spec:
      # pod 的数量
      replicas: 3
      selector:
        # 需要匹配的 pod 的 label
        matchLabels:
          run: nginx
      template:
        metadata:
          # 为生成的 pod 添加一个标签
          labels:
            run: nginx
        spec:
          # 指定 pod 中运行的容器
          containers:
          - image: nginx:1.17.1
            name: nginx
            ports:
            - containerPort: 80
              protocol: TCP
    ```

### Service

#### 概述

- 通过 Deploy 可以一组 Pod 来提供具体高可用性的服务，虽然每个 Pod 都会分配一个单独的 IP 地址，但是存在以下问题

  - Pod 的 IP 会随着 Pod 的重建产生变化

    ![image-20220523134300857](README.assets/image-20220523134300857.png)

  - Pod 的 IP 仅仅是就请你内部可见的虚拟 IP，**外部无法访问**

    ![image-20220523134100093](README.assets/image-20220523134100093.png)

- Service 可以看作是**一组同类的Pod**对外的访问接口，借助 Service，应用可以方便的实现**服务发现和负载均衡**

   ![Service概述.png](README.assets/1609137571725-a4754fdb-d0a1-4a49-a7a4-169fc6cc9703.png)

#### 使用

- 创建集群内部访问的 Service

  ```shell
  kubectl expose deploy xxx --name=服务名 --type=ClusterIP --port=对外暴漏端口 --target-port=指向集群中的Pod的端口 [-n 命名空间]
  ```

  ```shell
  kubectl expose deploy nginx --name=svc-deploy-nginx --type=ClusterIp --port=80 --target-port=80 -n dev
  ```

- 查看 Service

  ```shell
  kubectl get svc -n dev
  ```

  ![image-20220523135142985](README.assets/image-20220523135142985.png)

- 创建集群外部访问的 Service

  ```shell
  kubectl expose deployment nginx --name=svc-deploy-nginx2 --type=NodePort --port=80 --target-port=80  -n dev
  ```

  ![image-20220523135414863](README.assets/image-20220523135414863.png)

  ![image-20220523135515353](README.assets/image-20220523135515353.png)

- 删除 Service

  ```shell
  kubectl delete service svc-deploy-nginx -n dev
  ```

- 命令式对象配置

  - 新建一个 `svc-nginx.yaml`

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: svc-nginx
      namespace: dev
    spec:
      clusterIP: 10.109.179.231
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: ClusterIP
    ```

  - 创建

    ```shell
    kubectl create -f svc-nginx.yaml
    ```

  - 删除

    ```shell
    kubectl delete -f svc-nginx.yaml
    ```

## Pod 详解

### Pod 的介绍

#### Pod 的结构

Pod 的架构图

 ![Pod的结构.png](README.assets/1609399357456-e5dc5f6d-7c2e-44bf-aae3-50d51ec951e9.png)

- 每个 Pod 中都包含一个/多个容器，这些容器可以分为两类

- **Pause** 容器：这是每个 Pod 都有会的一个**根容器**，主要作用：

  1. 可以以它为依赖，评估整个 Pod 的健康状况

  2. 可以在根容器上设置IP地址，**其它容器都共享此IP（Pod的IP）**，以实现**Pod内部的网络通信**

     (这里是Pod内部的通讯，**Pod之间的通讯采用虚拟二层网络技术来实现，我们当前环境使用的是Flannel**)。

#### Pod 的定义

- Pod 的配置(yaml)的资源清单

  ```yaml
  apiVersion: v1     #必选，版本号，例如v1
  kind: Pod       　 #必选，资源类型，例如 Pod
  metadata:       　 #必选，元数据
    name: string     #必选，Pod名称
    namespace: string  #Pod所属的命名空间,默认为"default"
    labels:       　　  #自定义标签列表
      - name: string      　          
  spec:  #必选，Pod中容器的详细定义
    containers:  #必选，Pod中容器列表
    - name: string   #必选，容器名称
      image: string  #必选，容器的镜像名称
      imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 
      command: [string]   #容器的启动命令列表，如不指定，使用打包时使用的启动命令
      args: [string]      #容器的启动命令参数列表
      workingDir: string  #容器的工作目录
      volumeMounts:       #挂载到容器内部的存储卷配置
      - name: string      #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
        mountPath: string #存储卷在容器内mount的绝对路径，应少于512字符
        readOnly: boolean #是否为只读模式
      ports: #需要暴露的端口库号列表
      - name: string        #端口的名称
        containerPort: int  #容器需要监听的端口号
        hostPort: int       #容器所在主机需要监听的端口号，默认与Container相同
        protocol: string    #端口协议，支持TCP和UDP，默认TCP
      env:   #容器运行前需设置的环境变量列表
      - name: string  #环境变量名称
        value: string #环境变量的值
      resources: #资源限制和请求的设置
        limits:  #资源限制的设置
          cpu: string     #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
          memory: string  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
        requests: #资源请求的设置
          cpu: string    #Cpu请求，容器启动的初始可用数量
          memory: string #内存请求,容器启动的初始可用数量
      lifecycle: #生命周期钩子
  		postStart: #容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
  		preStop: #容器终止前执行此钩子,无论结果如何,容器都会终止
      livenessProbe:  #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
        exec:       　 #对Pod容器内检查方式设置为exec方式
          command: [string]  #exec方式需要制定的命令或脚本
        httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
          path: string
          port: number
          host: string
          scheme: string
          HttpHeaders:
          - name: string
            value: string
        tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
           port: number
         initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
         timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
         periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
         successThreshold: 0
         failureThreshold: 0
         securityContext:
           privileged: false
    restartPolicy: [Always | Never | OnFailure]  #Pod的重启策略
    nodeName: <string> #设置NodeName表示将该Pod调度到指定到名称的node节点上
    nodeSelector: obeject #设置NodeSelector表示将该Pod调度到包含这个label的node上
    imagePullSecrets: #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork: false   #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:   #在该pod上定义共享存储卷列表
    - name: string    #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}       #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string   #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:       　　　#类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:         #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
  ```

- 查看各种资源的配置项

  ```shell
  kubectl explain 资源类型[.属性]
  ```

  ```shell
  kubectl explain pod
  ```

  ![image-20220523142053369](README.assets/image-20220523142053369.png)

  ![image-20220523142200727](README.assets/image-20220523142200727.png)

> 在kubernetes中基本所有资源的一级属性都是一样的，主要包含5个部分：
>
> - apiVersion  \<string>：版本，有 kubernetes 内部定义，版本号必须用`kubectl api-versions`查询。
>
> - kind \<string>：类型，有 kubernetes 内部定义，类型必须用`kubectl api-resources`查询。
>
> - metadata  \<Object>：元数据，主要是资源标识和说明，常用的有 `name、namespace、labels` 等。
>
> - **spec** \<Object>：描述，这是配置中最重要的一部分，里面是对各种资源配置的详细描述。
>
> - status  \<Object>：状态信息，里面的内容**不需要定义，由kubernetes自动生成。**
>
> 在上面的属性中，spec是接下来研究的重点，继续看下它的常见子属性：
>
> - containers  <[]Object>：容器列表，用于定义容器的详细信息。
>
> - nodeName \<String>：根据 nodeName 的值将Pod调度到**指定的Node节点**上(也可以不配置，让Schedule配置)。
>
> - nodeSelector  <map[]> ：根据 NodeSelector 中定义的信息选择该 Pod 调度到**包含这些 Label 的 Node 上**。
>
> - hostNetwork  \<boolean>：是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络(如果使用这个，注意端口冲突)
>
> - volumes    <[]Object> ：存储卷，用于定义Pod上面挂载的存储信息。
>
> - restartPolicy	\<string>：重启策略，表示Pod在遇到故障的时候的处理策略。

### Pod 的配置

#### 概述

- 这里主要先研究 `pod.spec.containers` 属性的配置，也是 Pod 配置中**最关键**的一项配置

- 查看 `pod.spec.containers` 的可选配置项

  ```shell
  kubectl explain pod.spec.containers
  ```

  ```shell
  # 比较常用的属性
  KIND:     Pod
  VERSION:  v1
  RESOURCE: containers <[]Object>   # 数组，代表可以有多个容器FIELDS:
    name  <string>     # 容器名称
    image <string>     # 容器需要的镜像地址
    imagePullPolicy  <string> # 镜像拉取策略 
    command  <[]string> # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args   <[]string> # 容器的启动命令需要的参数列表 
    env    <[]Object> # 容器环境变量的配置
    ports  <[]Object>  # 容器需要暴露的端口号列表
    resources <Object> # 资源限制和资源请求的设置
  ```

#### 基本配置

- 创建 `pod-base.yaml` 文件

  ```yaml
  apiVersion: v1		# 固定值
  kind: Pod			# 资源名称
  metadata:
    name: pod-base	# pod name
    namespace: dev	# pod 隶属的 namesapce
    labels:
      user: xudaxian  # 打上 label
  spec:
    containers:
      - name: nginx # 容器名称
        image: nginx:1.17.1 # 容器需要的镜像地址
      - name: busybox # 容器名称
        image: busybox:1.30 # 容器需要的镜像地址
  ```

- 使用声明式对象配置使用该文件

  ```shell
  kubectl apply -f pod-base.yaml
  ```

  ![image-20220523144938878](README.assets/image-20220523144938878.png)

- 通过 `describe` 查看一个 pod 的详情

  ```shell
  kubectl describe pod pod-base -n dev
  ```

  ![image-20220523145119575](README.assets/image-20220523145119575.png)

#### 镜像拉取

- 创建 `pod-imagepullpolicy.yaml` 

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-imagepullpolicy
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers:
      - name: nginx # 容器名称
        image: nginx:1.17.1 # 容器需要的镜像地址
        imagePullPolicy: Always # 用于设置镜像的拉取策略
      - name: busybox # 容器名称
        image: busybox:1.30 # 容器需要的镜像地址
  ```

  - **imagePullPolicy**：用于设置镜像**拉取的策略**，K8S 支持三种拉取策略：
    - Always：总是从远程仓库拉取镜像（一直远程下载）。
    - IfNotPresent：本地有则使用本地镜像，本地没有则从远程仓库拉取镜像（本地有就用本地，本地没有就使用远程下载）
    - Never：只使用本地镜像，从不去远程仓库拉取，本地没有就报错（一直使用本地，没有就报错）
  - 默认情况下：
    - 如果镜像的 `tag` 为具体的版本号，默认使用 **IfNotPresent**
    - 如果镜像的 `tag` 为 **latest**，默认使用 **Always**

- 创建 pod

  ```shell
  kubectl apply -f pod-imagepullpolicy.yaml
  ```

#### 启动命令

- 在前面的案例中由于 `busybox` 并不是一个程序，在被启动之后就会自动关闭，可以使用 `command` 配置解决

- `command`：在 Pod 中的容器初始化后执行的一个命令

- 创建一个 `pod-command.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-command
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers:
      - name: nginx # 容器名称
        image: nginx:1.17.1 # 容器需要的镜像地址
        imagePullPolicy: IfNotPresent # 设置镜像拉取策略
      - name: busybox # 容器名称
        image: busybox:1.30 # 容器需要的镜像地址
        command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt;sleep 3;done;"]
  ```

- 启动 pod

  ```shell
  kubectl apply -f pod-command.yaml
  ```

- 查看 pod 运行状态

   ![image-20220523152938589](README.assets/image-20220523152938589.png)

- 进入到容器中查看运行详情

  ```shell
  kubectl exec -it pod名称 [-n 命名空间] -c 容器名 /bin/sh
  ```

  ```shell
  kubectl exec -it pod-command -n dev -c busybox /bin/sh
  ```

  ![image-20220523153224831](README.assets/image-20220523153224831.png)

- 注意：和 **command** 配置相似的还有一个 **args** 配置
  - 如果 command & args 都没有配置：使用容器 Dockerfile 的配置(ENTRYPOINT)
  - 如果 command 有，args 没有：Dockerfile 的配置会被忽略，使用 command
  - 如果 command 没有, args 有，Dockerfile 配置的 ENTRYPOINT 会被执行，使用当前的 args 作为参数
  - 如果 command & args 都有，Dockerfile 的配置会被忽略，执行 command 并追加 args

#### 环境变量(不推荐)

- 创建 `pod-env.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-env
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers:
      - name: nginx # 容器名称
        image: nginx:1.17.1 # 容器需要的镜像地址
        imagePullPolicy: IfNotPresent # 设置镜像拉取策略
      - name: busybox # 容器名称
        image: busybox:1.30 # 容器需要的镜像地址
        command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt;sleep 3;done;"]
        env:
          - name: "username"
            value: "admin"
          - name: "password"
            value: "123456"
  ```

- 执行

  ```shell
  kubectl apply -f pod-env.yaml
  ```

- 进入到容器内部

  ```shell
  kubectl exec -it pod-env -n dev -c busybox /bin/sh
  ```

- 查看环境变量

  ```shell
  / # echo $username
  admin
  ```

> 此种方式不推荐，推荐将这些配置单独存储在配置文件中，后面介绍。

#### 端口设置

- 关于 `ports` 的子配置

  ```shell
  kubectl explain pod.spec.containers.ports
  ```

  ![image-20220523154255380](README.assets/image-20220523154255380.png)

  ```yaml
  KIND:     Pod
  VERSION:  v1
  RESOURCE: ports <[]Object>
  FIELDS:
    name <string> # 端口名称，如果指定，必须保证 name 在pod中是唯一的
    containerPort <integer> # 容器要监听的端口(0<x<65536)
    hostPort <integer> # 容器要在主机上公开的端口，如果设置，主机上只能运行容器的一个副本(一般省略）
    hostIP <string>  # 要将外部端口绑定到的主机IP(一般省略)
    protocol <string>  # 端口协议。必须是UDP、TCP或SCTP。默认为“TCP”
  ```

- 创建 `pod-ports.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-ports
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers:
      - name: nginx # 容器名称
        image: nginx:1.17.1 # 容器需要的镜像地址
        imagePullPolicy: IfNotPresent # 设置镜像拉取策略
        ports:
          - name: nginx-port # 端口名称，如果执行，必须保证name在Pod中是唯一的
            containerPort: 80 # 容器要监听的端口 （0~65536）
            protocol: TCP # 端口协议
  ```

- 执行

  ```yaml
  kubectl apply -f pod-ports.yaml
  ```

- 查看 pod 的详细信息

  ![image-20220523154729934](README.assets/image-20220523154729934.png)

  可以通过 **PodIP:ContainerPort** 访问对应的容器

#### 资源配额

- K8S 提供了容器对**内存和CPU**的限额机制，避免某个容器吃掉大量的资源，导致其他容器无法进行

  - `limits`: 用于限制运行的容器的最大占用资源，当容器占用资源超过limits时会被终止，并进行重启。
  - `requests`: 用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动。
  - 可以通过 limits + requests 设置资源的上下限

- 创建 `pod-resources.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-resoures
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers:
      - name: nginx # 容器名称
        image: nginx:1.17.1 # 容器需要的镜像地址
        imagePullPolicy: IfNotPresent # 设置镜像拉取策略
        ports: # 端口设置
          - name: nginx-port # 端口名称，如果执行，必须保证name在Pod中是唯一的
            containerPort: 80 # 容器要监听的端口 （0~65536）
            protocol: TCP # 端口协议
        resources: # 资源配额
          limits: # 限制资源的上限
            cpu: "2" # CPU限制，单位是core数
            memory: "10Gi" # 内存限制
          requests: # 限制资源的下限
            cpu: "1" # CPU限制，单位是core数 
            memory: "10G" # 内存限制
  ```

  cpu：core数，可以为整数或小数。

  memory：内存大小，可以使用Gi、Mi、G、M等形式。

- 执行

  ```shell
  kubectl apply -f pod-resources.yaml
  ```

- 查看 pod 状态

  ```shell
  kubectl describe pod pod-resoures -n dev
  ```

  ![image-20220523155335153](README.assets/image-20220523155335153.png)

### Pod 的生命周期

- 我们一般将 Pod 对象从**创建到终止**这段时间范围称为 Pod 的生命周期：
  1. Pod 创建过程
  2. 运行**初始化容器过程**
  3. 运行**主容器**
     - 容器启动后钩子，容器终止前钩子
     - 容器的存活性探测，就绪性探测
  4. Pod 的终止过程

![Pod的生命周期.png](README.assets/1609399647590-472c8628-8b69-42ab-8a50-929c27737926.png)

- 在整个生命周期中，Pod 会出现5中**状态(相位)**：
  - 挂起(Pending): Api Server 已经创建了 Pod 资源对象，但其尚未被调度/处于下载镜像的过程中
  - 运行中(Running): Pod 已经被调度到某节点上，并且所有的容器都已经创建完成
  - 成功(Succeeded): Pod 的所有容器都已经成功终止并且不会被重启
  - 失败(Failed): 所有容器都已经终止，但至少有一个容器终止失败(即容器返回了非 0 值的退出状态)
  - 未知(Unkown): Api Server 无法正常获取到 Pod 对象的状态信息(通常是由于网络通信失败所导致的)

#### 创建和终止

![Pod的创建过程.jpg](README.assets/1609399660203-ab0d9834-3b35-4119-b304-4394b00f0b9d.jpeg)

##### Pod 的创建过程

1. 用户通过 `kubectl` 或其他api客户端提交需要创建的 Pod 给 ApiServer
2. Api Server 开始生成 Pod 对象的信息，并将信息存入到 **etcd**，然后返回确认信息到客户端
3. Api Server 开始对外暴露 etch 中 Pod 对象的变化，其他组件使用 **watch机制** 跟踪检查 Api Server 上的变动
4. **Scheduler** 发现有新的 Pod 对象要创建，开始**为 Pod 分配主机并将结果信息更新到 Api Server**
5. Node 节点上的 kubelet 发现有 Pod 要调度过来，通过 Docker 启动容器并将结果信息更新到 Api Server
6. Api Server 将接收到的 Pod 状态信息存入到 etcd 中

##### Pod 的终止过程

1. 用户向 Api Server 发送删除 Pod 对象的命令

2. Api Server 中的 Pod 对象信息会随着的推移而更新，在宽限期内(默认 30s)，Pod 视为 dead

3. 将 Pod 标记为 “Terminating” 状态。

4. （与第3步同时运行）kubelet 在监控到 Pod 对象转为 “Terminating” 状态的同时**启动 Pod 关闭程序**。

5. （与第3步同时运行）**端点控制器**监控到 Pod 对象的关闭行为时将其从所有匹配到此端点的 **Service 资源的端点列表中移除。**(关闭对外访问)

6. 如果当前 Pod 对象定义了 preStop 钩子处理器，则在其标记为 “terminating” 后即会以同步的方式启动执行；

   如若宽限期(30s)结束后，preStop 仍未执行结束，则第2步会被重新执行并额外获取一个时长为2秒的小宽限期。

7. Pod 对象中的容器**进程收到停止信号**。

8. 宽限期结束后，若存在任何一个仍在运行的进程，那么 Pod 对象即会收到 SIGKILL 信号。

9. kubelet 请求 API Server 将此 Pod 资源的宽限期设置为0从而完成删除操作，它变得对用户不在可见。

> 默认情况下，所有删除操作的宽限期都是30秒，不过，kubectl delete 命令可以使用“--grace-period=”选项自定义其时长，若使用0值则表示直接强制删除指定的资源，不过，此时需要同时为命令使用 “--force” 选项。

#### 初始化容器

- 初始化容器是在 **Pod 的主容器启动之前要运行的容器**，主要是做一些主容器的前置工作，具体两大特征：

  1. 初始化容器必须运行完成直至结束，如果某个初始化容器运行失败，那么 K8S 需要重启它直至功能完成
  2. 初始化容器必须按照定义的顺序执行，当且仅当前一个成功之后，后面的一个才能运行

- 应用场景

  1. 提供主容器镜像中**不具备**的工具程序或自定义代码
  2. 应用容器的启动可能需要某些依赖的条件，可以在初始化容器中先于应用容器串行启动并运行完成，保证应用容器的运行依赖条件得到满足

- 模拟下场景2：假设主容器运行需要 Nginx，但是在 Nginx 运行之前需要连接上其他两台的服务器

  1. 创建 `pod-initcontainer.yaml` 文件

     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: pod-initcontainer
       namespace: dev
       labels:
         user: xudaxian
     spec:
       containers: # 容器配置
         - name: nginx
           image: nginx:1.17.1
           imagePullPolicy: IfNotPresent
           ports:
             - name: nginx-port
               containerPort: 80
               protocol: TCP
           resources:
             limits:
               cpu: "2"
               memory: "10Gi"
             requests:
               cpu: "1"
               memory: "10Mi"
       initContainers: # 初始化容器配置
         - name: test-mysql
           image: busybox:1.30
           command: ["sh","-c","until ping 192.168.102.104 -c 1;do echo waiting for mysql ...;sleep 2;done;"]
           securityContext:
             privileged: true # 使用特权模式运行容器
         - name: test-redis
           image: busybox:1.30
           command: ["sh","-c","until ping 192.168.102.105 -c 1;do echo waiting for redis ...;sleep 2;done;"]
     ```

     注意修改下 ip 地址为当前网段内的没有使用过的 ip

  2. 执行

     ```shell
     kubectl apply -f pod-initcontainer.yaml
     ```

  3. 查看容器状态

     ```shell
     kubectl get pods -n dev
     ```

      ![image-20220523194239270](README.assets/image-20220523194239270.png)

  4. 动态监视 pod

     ```shell
     kubectl get pod pod-initcontainer -n dev -w
     ```

  5. 打开一个新窗口，依次输入以下命令

     ```shell
     ifconfig ens33:1 192.168.102.104 netmask 255.255.255.0 up
     ```

     ```shell
     ifconfig ens33:1 192.168.102.105 netmask 255.255.255.0 up
     ```

  6. 观察原窗口的变化

     ![image-20220523194515392](README.assets/image-20220523194515392.png)

#### 钩子函数

- 钩子函数能在指定的时刻到来时运行用户指定的程序代码

- K8S 在主容器启动之后和停止之前提供了两个钩子函数

  - post start: 在容器创建后运行，如果失败会重启容器
  - pre  stop:  在容器终止前执行，执行完成之后容器将成功终止，在其完成之前会**阻塞删除容器的操作**

- 钩子处理器支持三种方式定义动作

  1. `exec`: 在容器内执行一次命令

     ```yaml
     ……
       lifecycle:
          postStart: 
             exec:
                command:
                  - cat
                  - /tmp/healthy
     ……
     ```

  2. `tcpSocket`: 在当前容器尝试访问指定的 socket

     ```yaml
     …… 
        lifecycle:
           postStart:
              tcpSocket:
                 port: 8080
     ……
     ```

  3. `httpGet`: 在当前容器中向某 url 发起 HTTP 请求

     ```yaml
     …… 
        lifecycle:
           postStart:
              httpGet:
                 path: / #URI地址
                 port: 80 #端口号
                 host: 192.168.109.100 #主机地址  
                 scheme: HTTP #支持的协议，http或者https
     ……
     ```

- 创建一个 `pod-hook-exec.yaml` 文件

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-hook-exec
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers: # 容器配置
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - name: nginx-port
            containerPort: 80
            protocol: TCP
        resources:
          limits:
            cpu: "2"
            memory: "10Gi"
          requests:
            cpu: "1"
            memory: "10Mi"
        lifecycle: # 生命周期配置
          postStart: # 容器创建之后执行，如果失败会重启容器
            exec: # 在容器启动的时候，执行一条命令，修改掉Nginx的首页内容
              command: ["/bin/sh","-c","echo postStart ... > /usr/share/nginx/html/index.html"]
          preStop: # 容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作
            exec: # 在容器停止之前停止Nginx的服务
              command: ["/usr/sbin/nginx","-s","quit"]
  ```

- 执行

  ```shell
  kubectl apply -f pod-hook-exec.yaml
  ```

- 查看 Pod Ip

  ```shell
  kubectl get pods pod-hook-exec -n dev -o wide
  ```

  ![image-20220523200143249](README.assets/image-20220523200143249.png)

- 访问对应的 80 端口查看其输出内容

  ```shell
  [root@k8s-master pod]# curl 10.244.2.13:80
  postStart ...
  ```

#### 容器探测

##### 概述

- 作用：检测容器中的应用示例是否正常工作，是保障业务可用性的一种传统机制；经过探测后，如果实例的状态不符合预期，那么 K8S 就会把该问题实例 **"摘除"，不承担业务流量**

- 分类

  1. liveness probes: 存活性探测，决定是否重启容器，用于检测应用实例当前是否处于正常运行状态，如果不是，**k8s 会重启容器**。
  2. readiness probes：就绪性探测，决定是否将请求转发个给容器，用于检测应用实例是否可以接受请求，如果不能，**k8s不会转发流量**。
  3. startup probes(v1.16+)：判断容器内的应用是否已经启动成功，如果配置了其就会**先禁止其他的探针**，直到其探针成功为止，一旦成功后不在进行探测

- 第1和第2中探针目前支持三种探测方式：

  1. exec: 在容器内执行一次命令，如果命令执行的退出码为 0，则认为程序正常，否则不正常

     ```yaml
     ……
       livenessProbe:
          exec:
             command:
               -	cat
               -	/tmp/healthy
     ……
     ```

  2. tcpSocket：将会禅师访问一个用户容器的端口，如果可以建议连接，则认为程序正常，否则不正常

     ```yaml
     ……
        livenessProbe:
           tcpSocket:
              port: 8080
     ……
     ```

  3. httpGet：调用 web url，如果返回的转发码为 200 和 399 之间，则认为程序正常，否则不正常

     ```yaml
     ……
        livenessProbe:
           httpGet:
              path: / #URI地址
              port: 80 #端口号
              host: 127.0.0.1 #主机地址
              scheme: HTTP #支持的协议，http或者https
     ……
     ```

##### exec 方式

- 创建 `pod-liveness-exec.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-liveness-exec
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers: # 容器配置
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - name: nginx-port
            containerPort: 80
            protocol: TCP
        livenessProbe: # 存活性探针
          exec:
            command: ["/bin/cat","/tmp/hello.txt"] # 执行一个查看文件的命令，必须失败，因为根本没有这个文件
  ```

- 执行

  ```shell
  kubectl apply -f pod-liveness-exec.yaml
  ```

- 等待一会查看 pod 的详情信息

  ```shell
  kubectl describe pod pod-liveness-exec -n dev
  ```

  ![image-20220523204547917](README.assets/image-20220523204547917.png)

##### tcpSocket 方式

- 创建 `pod-liveness-tcpsocket.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-liveness-tcpsocket
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers: # 容器配置
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - name: nginx-port
            containerPort: 80
            protocol: TCP
        livenessProbe: # 存活性探针
          tcpSocket:
            port: 8080 # 尝试访问8080端口，必须失败，因为Pod内部只有一个Nginx容器，而且只是监听了80端口
  ```

- 执行

  ```shell
  kubectl apply -f pod-liveness-tcpsocket.yaml
  ```

- 查看 pod 状态

  ![image-20220523205008387](README.assets/image-20220523205008387.png)

##### httpGet方式

- 创建 `pod-liveness-httpget.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-liveness-httpget
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers: # 容器配置
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - name: nginx-port
            containerPort: 80
            protocol: TCP
        livenessProbe: # 存活性探针
          httpGet: # 其实就是访问http://127.0.0.1:80/hello
            port: 80 # 端口号
            scheme: HTTP # 支持的协议，HTTP或HTTPS
            path: /hello # URI地址
            host: 127.0.0.1 # 主机地址
  ```

- 执行

  ```shell
  kubectl apply -f pod-liveness-httpget.yaml
  ```

- 等待一会，查看 pod 的运行详情

  ```shell
  kubectl describe pod pod-liveness-httpget -n dev
  ```

  ![image-20220524085127692](README.assets/image-20220524085127692.png)



##### 补充

`livenessProbe` 的其他子属性

```shell
FIELDS:
livenessProbe
    initialDelaySeconds    # 容器启动后等待多少秒执行第一次探测
    timeoutSeconds      # 探测超时时间。默认1秒，最小1秒
    periodSeconds       # 执行探测的频率。默认是10秒，最小1秒
    failureThreshold    # 连续探测失败多少次才被认定为失败。默认是3。最小值是1
    successThreshold    # 连续探测成功多少次才被认定为成功。默认是1
```

#### 重启策略

- 在容器探测的过程中，一旦发现了问题，K8S 就会对容器所在的 Pod 进行重启，重启的策略分为 3 种

  - Always：容器失效时，自动重启该容器，默认值
  - OnFailure：容器终止运行且**退出码不为0**时重启
  - Never：不论状态如何，都不重启该容器。

- 重启策略适用于 Pod 对象中的**所有容器**，首次需要重启的容器，将在其需要的时候立即进行重启，随后再次重启的操作将由 kubelet 延迟一段时间后进行，其反复的重启操作的延迟时间以此为 10s、20s、40s、80s、160s和300s(最大的延迟时长)

- 创建 `pod-restart-policy.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-restart-policy
    namespace: dev
    labels:
      user: xudaxian
  spec:
    containers: # 容器配置
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - name: nginx-port
            containerPort: 80
            protocol: TCP
        livenessProbe: # 存活性探测
          httpGet:
            port: 80
            path: /hello
            host: 127.0.0.1
            scheme: HTTP
    restartPolicy: Never # 重启策略
  ```

- 执行

  ```yaml
  kubectl apply -f pod-restart-policy.yaml
  ```

- 查看容器运行状态

  ```shell
  kubectl describe pod pod-restart-policy  -n dev
  ```

  ![image-20220524090622109](README.assets/image-20220524090622109.png)

  ```shell
  kubectl get pod -n dev
  ```

  ![image-20220524090717858](README.assets/image-20220524090717858.png)
