# Kubernetes 最佳实践

## 注意 ⚠️

- _斜体表示引用_
- **未经允许，禁止转载**

## 学习本课程前，你应该具备如下知识：

- Linux 系统的基本配置和命令
- Docker 命令
- K8S 基本配置和 kubectl 命令
- 对存储和网络有基本了解更佳

## 课程目录

| Date  | Time | Title                          | Content                                        |
| ----- | ---- | ------------------------------ | ---------------------------------------------- |
| 第 1 天 | 上午   | [1. 操作系统](#1-操作系统)             | [1.1 CentOS 7](#11-centos-7)                   |
|       |      |                                | [1.4 Rocky](#14-rocky) 和其它选择（1.2-1.8）          |
|       |      |                                | [1.9 最佳实践](#19-最佳实践)                           |
|       |      | [2. 容器运行时](#2-容器运行时)           | [2.1 LXC 和 Docker（网络和存储模型）](#21-docker)        |
|       | 下午   |                                | [2.2 Containerd 和命令行工具们](#22-containerd)       |
|       |      |                                | [2.3 CRI-O](#23-cri-o)                         |
| 第 2 天 | 上午   |                                | [2.4 ECI 和 Kata](#24-kata-和它的朋友们)              |
|       |      |                                | [2.5 GPU](#25-gpu)                             |
|       |      |                                | [2.6 最佳实践](#26-最佳实践)                           |
|       | 下午   | [3. K8S 生命周期管理](#3-k8s-生命周期管理) | [3.1 K8S 集群创建删除、扩缩容、备份恢复](#31-集群的创建删除扩缩容备份恢复)  |
|       |      |                                | [3.2 K8S 升级](#32-版本升级)                         |
| 第 3 天 | 上午   |                                | [3.1.2 Sealos](#312-sealos) 和其它工具（kubekey/k0s） |
|       |      |                                | [3.1.4 KubeClipper](#314-kubeclipper)          |
|       |      |                                | [3.1.5 K3S](#315-k3s)                          |
|       | 下午   |                                | [3.3 迁移和纳管：Rancher 和 K3S](#33-迁移和纳管)           |
|       |      |                                | [3.4 统一认证和鉴权：KubeSphere](#34-统一认证和鉴权) 和计量、日志   |
|       |      |                                | [3.7 最佳实践](#37-最佳实践)                           |
|       |      | [4. 存储管理](#4-存储管理)             | [4.2 对接 NAS/NFS](#42-对接-nfs-和-nas)             |
| 第 4 天 | 上午   |                                | [4.3 对接 CephRBD](#43-对接-ceph-rbd)              |
|       |      |                                | [4.4 跨命名空间复制 PVC](#44-跨命名空间的快照和备份)             |
|       |      |                                | [4.6 最佳实践](#46-最佳实践)                           |
|       |      | [5. 网络管理](#5-网络管理)             | [5.2 对接 Calico](#52-对接-calico)                 |
|       | 下午   |                                | [5.3 对接 OVN](#53-对接-ovn) 和 Multus              |
|       |      |                                | [5.6 最佳实践](#56-最佳实践)                           |
|       |      | [6. 安全相关](#6-安全相关)             | [6.2 权限控制](#62-权限限制)                           |
|       |      |                                | [6.3 扫描工具](#63-k8s-安全扫描工具)                     |
|       |      |                                | [6.5 审计日志](#65-审计日志)                           |
|       |      |                                | [6.7 最佳实践](#67-最佳实践)                           |

## 1. 操作系统

[返回目录](#课程目录)

### 1.1 CentOS 7

[返回目录](#课程目录)

参考 <https://www.centos.org/download/>，EOF（End-of-life）时间 2024/06/30。CentOS 7 目前仍是生产环境中使用最广泛的容器操作系统。

CentOS 最麻烦的地方是配套的 yum/rpm 包过于久远。

#### 1.1.1 内核升级

首先是 Kernel，3.10 的 Kernel 会遇到的问题

1. KVM 无法启用硬件加速（kvm_intel
   模块无法加载），告警：`QEMU: Checking if device /dev/kvm exists : FAIL (Check that the 'kvm-intel' or 'kvm-amd' modules are loaded & the BIOS has enabled virtualization)`
2. K8S 启用 CPU/Memory limitaion 时，3.10 内核会额外耗费较多的 CPU 资源
3. 不支持 Calico ebpf 特性
4. 不支持较新的 cephcsi，cephfs

升级步骤：

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install -y https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
yum -y --disablerepo="*" --enablerepo="elrepo-kernel" list available
yum -y --disablerepo=\* --enablerepo=elrepo-kernel install kernel-lt.x86_64
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

grub2-set-default 0
# 这里的 0 要根据实际情况来填写

reboot
```

重启之后，可以看到内核已经升级完成

```console
[root@lab-kubernetes ~]# uname -a
Linux lab-kubernetes 5.4.196-1.el7.elrepo.x86_64 #1 SMP Tue May 24 12:49:20 EDT 2022 x86_64 x86_64 x86_64 GNU/Linux
```

#### 1.1.2 Python 3.8

Python 2 于 2020.01.01 结束支持，Python 3.6 于 2021.12 结束支持。

如果已经安装了 python 3.6，先移除

```bash
yum remove python3
```

然后安装 python 3.8

```bash
yum update -y
yum install centos-release-scl-rh -y
yum install rh-python38-python -y
yum install rh-python38-python-pip -y
yum install rh-python38-python-devel -y

echo "export PATH=$PATH:/opt/rh/rh-python38/root/usr/bin" >> /etc/profile && source /etc/profile

mkdir ~/.pip
vi ~/.pip/pip.conf
```

设置如下代理：

```ini
[global]
index-url=https://pypi.tuna.tsinghua.edu.cn/simple/
```

然后升级 pip

```bash
python3 -m pip install --upgrade pip
```

验证版本

```console
[root@lab-c2009 ~]# python3 --version
Python 3.8.11

[root@lab-c2009 ~]# python3 -m pip --version
pip 22.0.4 from /opt/rh/rh-python38/root/usr/local/lib/python3.8/site-packages/pip (python 3.8)
```

#### 1.1.3 git 升级

git 1.x 版本有一些命令与 2 不兼容，一般推荐升级到 2

```bash
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel asciidoc -y
yum install gcc perl-ExtUtils-MakeMaker -y

yum remove git -y

cd /usr/local/src/
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.32.0.tar.xz
tar -xvf git-2.32.0.tar.xz
cd git-2.32.0

make prefix=/usr/local/git all
make prefix=/usr/local/git install

echo "export PATH=$PATH:/usr/local/git/bin" >> /etc/profile && source /etc/profile

python3 -m pip install git-review
echo "export PATH=$PATH:/opt/rh/rh-python38/root/usr/local/bin" >> /etc/profile && source /etc/profile
```

完成后，可以确认 git 升级到 2.x 版本

```console
[root@lab-kubernetes git-2.32.0]# git --version
git version 2.32.0
```

### 1.2 Ubuntu LTS

[返回目录](#课程目录)

参考：<https://endoflife.software/operating-systems/linux/ubuntu>

| LTS(Long Term Support) | Release    | EOL(End Of Life) | HWE(Hardware Enablement) |
| ---------------------- | ---------- | ---------------- | ------------------------ |
| 22.04 Jammy Jellyfish  | 2022/04/21 | 2027/04          |                          |
| 20.04 Focal Fossa      | 2020/04    | 2025/04          | 2030/04                  |
| 18.04 Bionic Beaver    | 2018/04/26 | 2023/04          | 2028/04                  |

Ubuntu 的策略比较简单，保持循 LTS 升级即可。

### 1.3 CentOS 8 Stream

[返回目录](#课程目录)

[Comparing CentOS Stream and CentOS Linux](https://www.centos.org/cl-vs-cs/):

_End of Life_

- CentOS Linux 8 EOL: 2021-12-31
- CentOS Stream 8 EOL: 2024-05-31
- CentOS Stream 9 EOL: estimated 2027, dependent on RHEL9 end of “Full Support Phase”

_Upstream vs downstream_

- CentOS Linux 是 Red Hat Enterprise Linux (RHEL) 的 rebuild，是 RHEL 的 Downstream。CentOS Linux 的版本号是
  RHEL 的发布日期，比如 CentOS 8.2105 就是 RHEL 8.3（发布于 2021/05） 的 rebuild
- CentOS Stream 是 RHEL 的 upstream, RHEL 的 public development 分支。简单说，就是：**不稳定**。

综上，**CentOS 8 已经停止支持，CentOS 8 Stream 不稳定，两者都不要用于生产了**。

参考：<https://www.ispsystem.com/news/what-would-be-the-alternative-to-centos>

![](/image/CentOS-alternative-survey.png)

注：Oracle Linux 以 RHEL 为起始，与 RHEL 完全兼容，移除了 Red Hat 商标，加入了 Linux 错误修正。其可取之处是 7*24 服务…… 像另一个 RHEL。

### 1.4 Rocky

[返回目录](#课程目录)

参考：<https://www.logicweb.com/what-is-rocky-linux/>

2020 年 12 月（IBM 收购 Red Hat 之后），收购 CentOS 的 Red Hat 宣布，CentOS Linux 8 将于 2021 年底结束支持，比早些时候承诺的 10
年计划要短得多。CentOS Stream 是 development 版本，将按计划在2024年结束。从此以后 CentOS 将位于 RHEL 的上游，而不是下游，CentOS 用户实际上将是
RHEL 的测试人员。

这引发了广大 CentOS 用户的极大不满，CentOS 创始人 Gregory Kurtzer 随即宣称领导创建新的 “CentOS” 发行版的工作。Rocky 这个名字是为了纪念已故 CentOS
联合创始人 Rocky McGaugh。Kurtzer 曾在加州大学伯克利分校从事高性能计算工作很长时间。鉴于CentOS 在 CERN 等机构的粒子物理学中得到了广泛应用，这可能会是Rocky
Linux 的主要关注点之一。

参考：<https://rockylinux.org/>

_Enterprise Linux, the community way. Rocky Linux is an open-source enterprise operating system
designed to be 100% bug-for-bug compatible with Red Hat Enterprise Linux®. It is under intensive
development by the community._

参考：<https://rockylinux.org/download/>，Planned EOL: 2029/05/31

综上，**Rocky 是另起炉灶的 CentOS Linux**。当下进展不错。参考 <https://github.com/rocky-linux/rocky>，2021/06，Rocky
Linux 已经生产环境 ready。建议在生产环境中使用（CentOS 7 结束支持前半年，也就是 2023 年应该转向 Rocky Linux）。

### 1.5 AlmaLinux

AlmaLinux 由 CloudLinux 的开发人员构建和维护，CloudLinux 是一家提供服务器托管和 Linux 软件的公司。这是一家在 RHEL
分支方面经验丰富的公司，十多年来一直构建和维护其内部发行版 CloudLinux OS，它本身就是一个分支。

你**应该选 Alma 还是 Rocky**？它俩都致力于提供社区版的 RHEL，这有点难回答：

从所有权来说（AlmaLinux 开放，Rocky 独裁）：

- Rocky Linux 由 Kurtzer 创立的 Rocky Enterprise Software Foundation (RESF) 控制和管理。同时，他还是为 Rocky Linux
  提供保护伞的 Public Benefit Corporation (PBC) 所有者。所以，Kurtzer 基本上拥有 Rocky。是的，RESF
  有一个管理委员会，但无论你怎么看，Kurtzer 都是公司持有人，并且可能是 Rocky Linux 的决策者。“独裁”可能是好事，也可能是坏事。理论上讲，他可能像卖 CentOS 一样，再卖一次
  Rocky。我们只需要相信他，他会阻止之前发生的事情再次发生。
- AlmaLinux OS 基金会是一个真正的非营利组织，拥有独立的董事会和公共所有权，贡献者在项目治理中拥有直接投票权和直接发言权。

从贡献者来说（AlmaLinux 集中，Rocky 分散）：

- AlmaLinux 的贡献者相对集中（CloudLinux 雇员居多）
- Rocky 则分散得多

从经验来说（两者差不多）：

- Rocky 是 CentOS 创始人的一个倡议，这意味着在这方面有很多经验。
- AlmaLinux 主要由 CloudLinux 团队开发，他们对 CentOS 也有丰富的经验，所以实际上核心开发团队有超过十年的重建 RHEL 的经验。

从响应速度来说（AlmaLinux 更快）：

- AlmaLinux 更早推出（2021/03/30）稳定版本，Rocky 是 2021/06
- AlmaLinux 更快响应社区问题

综上：**目前 Rocky 的呼声更高，但 AlmaLinux 也不差，可以再看看**。

### 1.6 Openeuler

[返回目录](#课程目录)

参考：<https://developer.huaweicloud.com/ict/en/site-euleros/euleros>，EulerOS 是华为推出的企业 Linux
系统（诞生于华为科学实验室），Openeuler 是 EulerOS 的社区版本，2019 年开源，现贡献给开放原子基金会，代码托管在 <https://gitee.com/openeuler>。

OpenEuler 兼容 CentOS（但是它并不是蹭 CentOS 8 结束支持热点才开源的，开源在那之前），相较 CentOS 对核内关键功能如进程调度、内存管理、IO
读写进行了深度优化；同时在核外构筑了容器 iSula、虚拟化 StraitVirt、机密计算 SecGear、毕昇 JDK（Huawei JDK 的开源版本）。

综上，**OpenEuler 的优势是国产、信创，能较好地适配国产 ARM 服务器（特别是华为鲲鹏、泰山等）。缺点是兼容性和稳定性验证稍显不足**。

### 1.7 龙蜥

[返回目录](#课程目录)

龙蜥由阿里巴巴在 2021/10/20 孵化出来，诞生背景是 CentOS 8 结束支持（CentOS 8 结束支持这事，堪称“一鲸落，万物生”，可惜 Alma 和 Rocky Linux
起得太快）。

参考：<https://openanolis.cn/anolisos>，100% 兼容 CentOS 8 软件生态。

参考：<https://www.zhihu.com/question/502615238/answer/2408765289>，一言难尽。

### 1.8 麒麟

这个就说来话长了，我也查了半天，诸君姑且听之。

- **中标麒麟**：2010/12/16，中标 Linux 和国防科大的“银河麒麟”在上海宣布合并，开发方中标软件有限公司和国防科大同日缔结了战略合作协议，双方今后将共同以“中标麒麟”的新品牌出现。
- **银河麒麟** Kylin Operating System 是天津麒麟信息技术有限公司旗下的国产 Linux 操作系统，源自国防科大“麒麟”、“银河麒麟”操作系统，支持主流 X86 架构
  CPU 以及国产飞腾 CPU 平台。国防科大继续了老“麒麟”的开发。
- **优麒麟** UbuntuKylin 是 Ubuntu 社区中面向中文用户的 Ubuntu 衍生版本，中文名称优麒麟，与麒麟系统没有关系。优麒麟有两个身份，首先它是 Ubuntu 的一个官方
  Flavor 版本。其次，它背后也有国防科大和天津麒麟的支持，可以看做银河麒麟的社区版。优麒麟最初的目标是像 Ubuntu 一样占领中国市场，可是很多人直接选择了
  Ubuntu，一些选择了更接地气的 Deepin，所以优麒麟并不算非常成功。
- **湖南麒麟**，湖南麒麟信息工程技术有限公司（简称湖南麒麟）是 2007
  年成立的一家民营企业，公司成立之初依托国防科大计算机学院，长期致力于信息安全的研发，在集中管控和机要密码等领域有一定的影响力。2014
  年天津麒麟成立时，国防科大将“麒麟”、“银河麒麟”等无形资产注入了天津麒麟，湖南麒麟原有的操作系统研发团队整体转入天津麒麟。现在的湖南麒麟，只是一家单纯的民营企业了，可以视为一个新的系统。

综上：

- 中标麒麟像 CentOS，原来兼容性很好，2021 版本改了很多 rpm 依赖
- 银河麒麟像 Ubuntu
- 优麒麟基本不会在生产环境中用到
- 2022/05/16，科创板上市委公告，湖南麒麟信安科技股份有限公司首发获通过

emmmmm，得承认，麒麟在信创方面有一定优势。

### 1.9 最佳实践

1. 如选用 CentOS 系列，**2023/06/30 之前 CentOS 7 没问题**（需要升级 linux kernel 和工具库，比如 python 3.8，git v2）；但之后要选择
   AlmaLinux 或者 Rocky Linux，二选一，**2024/06/30 之前要完成生产环境 OS 升级**。不要使用 CentOS Stream。
2. 如选用 Ubuntu 系列，Ubuntu LTS（或者 Debian）都可以，持续升级就行
3. 如考虑信创，OpenEuler 对国产 ARM 服务器兼容性良好；如果政策有倾向性，那么麒麟系列、龙蜥对应考虑

## 2. 容器运行时

[返回目录](#课程目录)

### 2.1 Docker

[返回目录](#课程目录)

#### 2.1.1 Linux 容器和 Docker

Docker 的历史

- 2013 年 Docker 诞生，因对容器采用开创性方法而备受关注。虽然 Docker 并未发明容器，但它让开发人员极容易使用容器，因而在云行业掀起了一股延续至今的浪潮
- 2016 年，Docker 拒绝了微软斥资 40 亿美元的收购要约
- 2018 年估值达到最高峰：13.2 亿美元
- Kubernetes 的横空出世给这家初创公司施加了巨大的压力，Docker 未找到清晰的商业模式，Docker-Compose 和 Docker-Swarm 等编排工具被废弃。
- 2019 年，Docker 拆分公司，将企业业务出售给了 Mirantis。剩余的家底筹集了 3500 万美元的 A 轮融资，实际上重组为一家全新的初创公司
- 如今（2022年），Docker 致力于为开发人员提供自动化工具，最近宣布年度经常性收入（ARR）已超过 5000 万美元

我们需要知道的：

- Google 最早开始大规模使用容器技术（Borg 系统，后来简化重构，以 K8S 开源）

  ![](/image/borg-arch.png)

- Linux 容器（LXC）技术主要两个内核特性组成：namespace & cgroup。namespace 最早是 2002 年在 2.4.19 内核中引入（mount
  单元），用于实现**资源隔离**。cgroup 2000 年以前就在 google 使用，2006 年以后贡献到 Linux Kernel，用于实现**资源限制**，2008 年 LXC
  技术基本完成。
- Docker 在 2013 年成立之后，对 LXC 进行封装，提供了极简的容器使用方案，几乎成为容器的代名词

Docker Overview，参考：<https://docs.docker.com/engine/docker-overview>

![](/image/docker-arch.png)

![](/image/docker-undertech.png)

UFS（union file system）用来支持对文件系统的修改分层

Docker QuickStart，参考：<https://docs.docker.com/get-started>

#### 2.1.2 Docker 和 K8S

**K8S 1.24 开始放弃对 Docker 的支持，放弃的是什么？**

![](/image/k8s-CRI-OCI-docker.png)

接口解释：

1. `unix:///var/run/dockershim.sock`
2. `unix:///var/run/docker.sock`
3. `unix:///run/containerd/containerd.sock`

Docker 18.x 以后的版本，会有 3 个组件：runC、containerd、dockerd。

- Dockerd 是个守护进程，直接面向用户，用户使用的命令 docker 直接调用的后端就是 dockerd
- Dockerd 不会直接使用 runc，而是去调用 containerd
- containerd 会 fork 出一个单独子进程的 containerd-shim，使用 runc 将目标容器跑起来。
- Kubelet 则是通过内置的 docker-shim 去调用 dockerd，或者直接通过 CRI 接口调用 containerd

通过 ps -ef 可以看到几个进程之间的关系：

```console
[root@lab-kubernetes ~]# docker run -d nginx
614b35ef3e2842cdc57bcb08c89df93533802d443065f8887c3951a601b738c2

[root@lab-kubernetes ~]# docker run -d nginx
62970cd2a51798b07c4f8ad6b42a19e72410aba7a2e6841c1c1606a35feedbee

[root@lab-kubernetes ~]# ps -ef
root     18442     1  0 00:44 ?        00:00:27 /usr/bin/containerd
root     18452     1  0 00:44 ?        00:00:02 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root     20595     1  0 11:18 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 614b35ef3e2842cdc57bcb08c89df93533802d443065f8887c3951a601b738c2 -address /run/containerd/containerd.sock
root     20615 20595  0 11:18 ?        00:00:00 nginx: master process nginx -g daemon off;
root     20708     1  0 11:22 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 62970cd2a51798b07c4f8ad6b42a19e72410aba7a2e6841c1c1606a35feedbee -address /run/containerd/containerd.sock
root     20728 20708  0 11:22 ?        00:00:00 nginx: master process nginx -g daemon off;
```

K8S 和 Docker
有什么关系？参考：[Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

Docker 和 Rocket 之争

![](/image/k8s-cri-rocket.png)

- Orchestration API -> Container API (cri-runtime) -> Kernel API(oci-runtime)

**OCI 标准是什么**？：runC / Kata（ 及其前身 runV 和 Clear Containers ），gVisor，Rust railcar

- 容器镜像的制作标准，即 ImageSpec。大致规定：容器镜像这个压缩了的文件夹里以 xxx 结构放 xxx 文件
- 容器要需要能接收哪些指令，以及这些指令的行为，即 RuntimeSpec。这里面的大致内容就是“容器”要能够执行 "create"，"start"，"stop"，"delete"
  这些命令，并且行为要规范。
- Docker 则把 libcontainer 封装了一下，变成 runC 捐献出来作为 OCI 的参考实现。

**CRI 标准是什么**？：Docker（ 借助 dockershim ），containerd（ 借助 CRI-containerd ），CRI-O，Frakti。是一组 gRPC
接口，[cri-api/pkg/apis/services.go](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/services.go)：

- 一套针对容器操作的接口，包括创建，启停容器等等
- 一套针对镜像操作的接口，包括拉取镜像删除镜像等
- 一套针对 PodSandbox（容器沙箱环境）的操作接口

containerd-1.0，对 CRI 的适配通过一个单独的进程 CRI-containerd 来完成

![](/image/k8s-containerd-1-0.png)

containerd-1.1，把适配逻辑作为插件放进了 containerd 主进程中

![](/image/k8s-containerd-1-1.png)

综上：

1. 容器运行时是管理容器和容器镜像的程序。有两个标准，一个是 CRI，抽象了 kubelet 如何启动和管理容器；另一个是 OCI，抽象了怎么调用内核 API
   来管理容器。标准实际上是定义了一系列接口，让上层应用与底层实现接耦。
2. 实现 CRI 的 runtime 有 CRI-O、CRI-containred 等，CRI 的命令行客户端是 crictl。containerd 的客户端是 ctr。dockerd 的客户端是
   docker。它们通过 unix sock 与对应的 daemon 交互。
3. OCI 的默认实现是 runc。runc 是一个命令行工具，而不是一个 daemon。通过 runc 我们可以手动启动一个容器，也可以查看其他进程启动的容器。

#### 2.1.3 Docker 部署和基本使用

Docker shim 接口虽然在 K8S 1.24 中被弃用，但我们可以通过 docker 来理解 Linux 容器的一般操作和使用方法。这部分知识在 K8S 的其它 CRI 中是通用的。

在 Ubuntu 18.04 / Ubuntu 20.04 / CentOS 7 上配置 Docker

```bash
# 更新依赖仓库
apt-get update -y || yum update -y

# 安装 Docker
apt-get install docker.io -y || yum install docker -y
systemctl enable docker --now

# 检查 docker 服务状态
systemctl status docker

# ubuntu 中需要把 docker 的 cgroup driver 改成 systemd
# !! centos 默认就是 systemd，不要修改这个文件，改了 docker 会起不来，保持 {} 就好
vi /etc/docker/daemon.json

{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

systemctl restart docker

# run hello-world
docker run hello-world
```

> Note：2021 年 7 月之后，ubuntu 环境 kubeadmin 默认都是 1.22+ 版本，因此需要将 docker 的 cgroup driver 改成 systemd（原来是
> cgroup）。如果不改，后续 kubeadm init
> 时，会报错：`[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.`

> 检查 journalctl -x -u
> kubelet，可以看到：`Aug 07 15:10:45 ckalab2 kubelet[11394]: E0807 15:10:45.179485   11394 server.go:294] "Failed to run kubelet" err="failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"cgroupfs\""`

> 看官方文档：<https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/>：`In v1.22, if the user is not setting the cgroupDriver field under KubeletConfiguration, kubeadm will default it to systemd.`

> 所以我们需要把 docker 的 cgroup driver 改成 systemd

> 修改步骤参考：<https://stackoverflow.com/questions/43794169/docker-change-cgroup-driver-to-systemd>

> 修改完成后，检查一下 docker cgroup，确保 docker cgroup 是 systemd 了：`sudo docker info | grep -i cgroup`

如何创建一个镜像？如何启动和调试容器？[Github](https://github.com/99cloud/lab-openstack/tree/master/src/docker-quickstart)
或 [Gitee](https://gitee.com/dev-99cloud/lab-openstack/tree/master/src/docker-quickstart)

```bash
mkdir ~/test
cd ~/test

wget https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/docker-quickstart/app.py
wget https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/docker-quickstart/requirements.txt
wget https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/docker-quickstart/Dockerfile

# 如果是 CentOS 7.x 需要安装下 python3
# yum install python3 python3-pip

# python3 -m pip install -r requirements.txt
python3 -m pip install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements.txt

python3 app.py
# * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
# 此时可以从浏览器访问 http://<ip>/

docker build --tag=friendlyhello .

# 可以看一下本地镜像列表
docker images

docker rm testFlask
docker run --rm -p 4000:80 --name=testFlask friendlyhello

# 也可以从 dockerhub 下载
docker run --rm -p 4000:80 --name=testFlask 99cloud/friendlyhello:3.9.6
# * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
# 此时可以从浏览器访问 http://<ip>:4000

# 如果要跑在后台，可以加 -d 参数
docker run -d --rm -p 4000:80 --name=testNew 99cloud/friendlyhello:3.9.6
```

```console
# 进入容器调试
$ docker exec -it testFlask /bin/bash
root@4224b69e7ee3:/app# env
HOSTNAME=4224b69e7ee3
PWD=/app
HOME=/root
NAME=World
...

root@4224b69e7ee3:/app# ps -ef
```

```bash
# 查看容器日志
docker logs -f testFlask

# 结束容器
docker stop testFlask

# 启动容器
docker start testFlask

# 从容器生成新的镜像
docker stop testFlask
docker commit testFlask test-new

# 删除容器
docker rm testFlask

# 删除镜像
docker rmi maodouzi/get-started:part2
```

#### 2.1.4 Docker 网络模型和 network namespace

参考：<https://docs.docker.com/network/#network-drivers>，Docker 网络模型分为若干种，在生产环境中会被用到的主要是 bridge 和 host
模式。

host 模式是容器和宿主机共享同一个 TCP/IP 协议栈（network namespace），容器的网络和宿主机网络不做隔离（只是网络不隔离，PID / IPC / MNT / UTS 还是
namespace 隔离的）

bridge 模式下，容器网络和宿主机网络是通过 network namespace 隔离开的，然后再通过类似 NAT 或者 port-mapping 技术完成转发。

因此，我们通过 bridge 模式来研究 network namespace 的底层逻辑。

![](/image/bridge_network.jpeg)

docker 容器实现没有把 network namespaces 放到标准路径 `/var/run/netns` 下，所以 `ip netns list` 命令看不到。但是可以看
`ll /proc/<pid>/ns`，两个进程的 namespaces id 相同说明在同一个 namespaces

```console
[root@cloud025 ns]# ll /proc/2179/ns/
total 0
lrwxrwxrwx 1 root root 0 Aug 10 11:58 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Aug 10 11:58 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Aug 10 11:58 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0 Aug 10 11:58 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Aug 10 11:58 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Aug 10 11:58 uts -> uts:[4026531838]
```

做个软链接，就可以看到 netns 了

```console
[root@cloud025 ns]# docker ps
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS                  NAMES
07297a3ac7ea        nginx:latest                  "/docker-entrypoin..."   29 minutes ago      Up 29 minutes       80/tcp                 devtest
0935b08509a4        test-new                      "python app.py"          35 minutes ago      Up 35 minutes       0.0.0.0:5000->80/tcp   testNew
c23c76dd779c        99cloud/friendlyhello:3.9.6   "python app.py"          37 minutes ago      Up 36 minutes       0.0.0.0:4000->80/tcp   testFlask

[root@cloud025 ns]# docker inspect testFlask | grep -i pid
            "Pid": 1159,
            "PidMode": "",
            "PidsLimit": 0,

[root@cloud025 ns]# mkdir -p /var/run/netns

[root@cloud025 ns]# ln -s /proc/1159/ns/net /var/run/netns/testFlask

[root@cloud025 ns]# ip netns list
testFlask (id: 0)
devtest (id: 2)
testNew (id: 1)
```

进入对应的 namespaces，看 ip，pod namespace 里虚拟网卡的 `link-netnsid` 始终等于 0

```console
[root@cloud025 ns]# ip netns exec testNew ip a
...
44: eth0@if45: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 scope global eth0
      valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:3/64 scope link
      valid_lft forever preferred_lft forever
```

在 root namespaces 中 `ip a`，可以看到 `link-netnsid = 1`，1 是前面 `ip netns list` 里的 namespaces id

```console
[root@cloud025 ns]# ip a
45: vethb6d08be@if44: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 0e:d9:14:d1:86:98 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::cd9:14ff:fed1:8698/64 scope link
      valid_lft forever preferred_lft forever
```

这是一对 veth pair，看他们的序号和 if 可以发现

看网桥，可以看到这个 root namespaces 的虚拟网卡绑在 docker0 网桥上。在 CentOS 上，需要安装一下：`yum install bridge-utils`

```console
[root@cloud025 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.02428c25c112	no		vethb6d08be
virbr0		8000.525400d583b9	yes		virbr0-nic
```

#### 2.1.5 尝试对虚拟网卡抓包，看进出容器的流量

怎么对 pod 的网口抓包？

```bash
# 查看指定 pod 运行在那个 node 上
kubectl describe pod <pod> -n <namespace>

# 获得容器的 pid
docker inspect -f {{.State.Pid}} <container>

# 进入该容器的 network namespace
nsenter --target <PID> -n

# 使用 tcpdump 抓包，指定 eth0 网卡
tcpdump -i eth0 tcp and port 80 -vvv

# 或者抓包并导出到文件
tcpdump -i eth0 -w ./out.cap

# 退出 nsenter，要记得退出！！
exit
```

然后用 wireshark 或者 netmon 打开，即可查看。

### 2.2 Containerd

[返回目录](#课程目录)

#### 2.2.1 Containerd 和 Docker

Containerd 是从 Docker 中分离出来的一个项目，是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性。

![](/image/k8s-cri-docker.png)

![](/image/k8s-cri-containerd.png)

![](/image/docker-and-kubernetes-use-containerd-2000-opt.png)

生产环境中 Containerd 作为容器运行时应用最为广泛

![](/image/containerd-eco-system.jpeg)

[Containerd 的优势](https://icloudnative.io/posts/getting-started-with-containerd/)：

1. 兼容 Docker

   Docker 直接带 Containerd，Containerd 可以单独装，也可以装 Docker，再用 Docker 中的 Containerd 对接 K8S。这在某些必须使用到
   docker 的场景中比较受欢迎，比如超融合场景，需要依赖 docker 部署 ceph 等

2. 直接兼容 K8S CRI
   - 不再需要 docker-shim 适配器
   - 可直接对接 K8S CRI 接口

3. 性能优良

   使用 bucketbench 对 Docker、crio 和 Containerd 的性能测试结果，包括启动、停止和删除容器，以比较它们所耗的时间：

   ![](/image/cri-stress-testing.png)

   可以看到 Containerd 在各个方面都表现良好，总体性能优于 Docker 和 crio

#### 2.2.2 命令行对比

![](/image/k8s-cri-tools.png)

![](/image/k8s-containerd-tools.png)

从以上两张图，可以大致了解 crictl / ctr / nerdctl / podman 的关系。

##### 2.2.2.1 crictl

参考：<https://kubernetes.io/docs/reference/tools/map-crictl-dockercli/>

crictl 命令和 docker 命令用法基本一样，是同一家开发的东西。

相同的命令包括：

- attach
- exec
- images
- info
- inspect
- logs
- ps
- stats
- version
- create
- kill
- pull
- rm
- rmi
- run
- start
- stop
- update

以下命令只有 crictl 有

| crictl       | Description                            |
| ------------ | -------------------------------------- |
| imagefsinfo  | Return image filesystem info           |
| inspectp     | Display the status of one or more pods |
| port-forward | Forward local port to a pod            |
| pods         | List pods                              |
| runp         | Run a new pod                          |
| rmp          | Remove one or more pods                |
| stopp        | Stop one or more running pods          |

参考：<https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/>

可以看到 crictl 对 pod / container / image 的操作

参考：<https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md>，安装 crictl

```bash
VERSION="v1.24.1"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

```console
[root@lab-kubernetes tmp]# crictl --version
crictl version v1.24.1

[root@lab-kubernetes tmp]# crictl images
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock]. As the default settings are now deprecated, you should set the endpoint instead.
ERRO[0002] connect endpoint 'unix:///var/run/dockershim.sock', make sure you are running as root and the endpoint has been started: context deadline exceeded
ERRO[0004] connect endpoint 'unix:///run/containerd/containerd.sock', make sure you are running as root and the endpoint has been started: context deadline exceeded
FATA[0006] connect: connect endpoint 'unix:///run/crio/crio.sock', make sure you are running as root and the endpoint has been started: context deadline exceeded
```

这里 crictl 不可用是因为没有为 crictl 命令配置 endpoint。在 centos 7 上部署的 docker 版本是 1.13，版本太低了，使用的不是 containerd 而是
libcontainerd，略有不同。可以卸载原有的 docker，重新安装高版本 docker。

参考：<https://docs.docker.com/engine/install/centos/>

```bash
yum remove -y docker \
            docker-client \
            docker-client-latest \
            docker-common \
            docker-latest \
            docker-latest-logrotate \
            docker-logrotate \
            docker-engine

yum install -y yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# yum list docker-ce --showduplicates | sort -r
# yum install docker-ce-20.10.16 docker-ce-cli-20.10.16 containerd.io docker-compose-plugin

systemctl enable docker --now

echo "alias docker-compose='docker compose'" >> /etc/profile && source /etc/profile

docker version
docker-compose version
```

然后可以检查 systemctl 的配置文件：

```ini
[root@lab-kubernetes ~]# cat /usr/lib/systemd/system/docker.service

[Unit]
...
Requires=docker.socket containerd.service

[Service]
...
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

启动时使用了 `/run/containerd/containerd.sock`

此时 crictl 应该可以对接 containerd 了，我们再测试一下：

```console
[root@lab-kubernetes ~]# cat /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock

[root@lab-kubernetes ~]# crictl images
FATA[0000] listing images: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.ImageService
```

这个报错是因为 containerd 的配置文件中，默认关闭了 cri plugin，所以无法进行 cri 对接。

修改配置文件，启用 cri plugin：

```console
[root@lab-kubernetes ~]# vi /etc/containerd/config.toml

[root@lab-kubernetes ~]# cat /etc/containerd/config.toml | grep cri
# disabled_plugins = ["cri"]

[root@lab-kubernetes ~]# systemctl restart containerd

[root@lab-kubernetes ~]# crictl images
IMAGE               TAG                 IMAGE ID            SIZE

[root@lab-kubernetes ~]# crictl ps -a
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
```

综上：

1. docker 构建在 containerd 之上，所以在生产环境中，我们可以同时拥有 containerd 和 docker，不干扰。
2. crictl 命令的优点是和 docker 命令非常像，几乎一样。差异是 image 相关的处理逻辑（load / save / tag）缺失，这些不是 cri 考虑的范畴。这个可以由 ctr
   或 nerdctl 补齐。
3. crictl 兼容 cri API，这就使得它不仅可以用于 containerd，而且适用于 CRIO 等所有支持 CRI 接口的容器运行时。

##### 2.2.2.2 ctr

ctr 命令会随着 containerd 服务一起安装。

```bash
ctr version

# pull 镜像
# 注意 ⚠️，这里要按 registry/orgnization/image:tag 写全，没有默认了
ctr images pull docker.io/library/redis:alpine3.13

# 查看镜像
ctr i ls

# 启动镜像
ctr run -d docker.io/library/redis:alpine3.13 redis
```

查看容器，注意 ⚠️：

- ctr 概念空间中，container 和 task 分离
- container 是容器分配和附加资源的元数据对象，是静态内容
- task 是系统上一个活动的、正在运行的进程
- task 在每次运行后删除，而 container 可以被多次使用、更新和查询

这和 docker 中 container 定义是不一样

```console
ctr container ls
CONTAINER    IMAGE                                 RUNTIME
redis        docker.io/library/redis:alpine3.13    io.containerd.runc.v2

ctr task ls
TASK     PID      STATUS
redis    20808    RUNNING
```

进入到容器中执行 redis 命令

```console
ctr task exec -t --exec-id redis-sh redis sh
/data # redis-cli
127.0.0.1:6379> set k1 v1
OK
127.0.0.1:6379> get k1
"v1"
```

注意 ⚠️：containerd 中存在 namespace 概念，这样可以将不同业务和应用进行隔离。例如 k8s 使用 containerd 和直接使用 ctr 创建的容器可以隔离开。

```console
$ ps -ef | grep runc | grep redis
/usr/local/containerd/bin/containerd-shim-runc-v2 -namespace default -id redis -address /run/containerd/containerd.sock
```

查看一下当前的 namespace:

```console
$ ctr ns ls
NAME    LABELS
default
k8s.io
```

不同 namespace 下 pull 的镜像也是隔离显示的，可以使用 -n 指定具体的 namespace：

```bash
ctr -n default i ls
ctr -n k8s.io i ls
```

`crictl pull` 的镜像是在 `k8s.io` namespace 下，`docker pull` 的镜像不在 moby namespaces（容器在）。

```console
$ crictl pull docker.io/library/redis:alpine3.13

$ crictl images
IMAGE                                       TAG                 IMAGE ID            SIZE
docker.io/library/redis                     alpine3.13          554d20f203657       10.9MB
```

crictl 不能像 ctr 那样通过参数给定用户名和密码的方式从开启认证的私有仓库中 pull 镜像。需要对 containerd 进行配置。 containerd
提供的各种功能在其内部都是通过插件实现的，可以使用 `ctr plugins ls` 查看 containerd 的插件。

```console
[root@lab-kubernetes ~]# ctr plugin ls
TYPE                                  ID                       PLATFORMS      STATUS
io.containerd.content.v1              content                  -              ok
io.containerd.snapshotter.v1          aufs                     linux/amd64    skip
io.containerd.snapshotter.v1          btrfs                    linux/amd64    skip
io.containerd.snapshotter.v1          devmapper                linux/amd64    error
io.containerd.snapshotter.v1          native                   linux/amd64    ok
io.containerd.snapshotter.v1          overlayfs                linux/amd64    ok
io.containerd.snapshotter.v1          zfs                      linux/amd64    skip
io.containerd.metadata.v1             bolt                     -              ok
io.containerd.differ.v1               walking                  linux/amd64    ok
io.containerd.event.v1                exchange                 -              ok
io.containerd.gc.v1                   scheduler                -              ok
io.containerd.service.v1              introspection-service    -              ok
io.containerd.service.v1              containers-service       -              ok
io.containerd.service.v1              content-service          -              ok
io.containerd.service.v1              diff-service             -              ok
io.containerd.service.v1              images-service           -              ok
io.containerd.service.v1              leases-service           -              ok
io.containerd.service.v1              namespaces-service       -              ok
io.containerd.service.v1              snapshots-service        -              ok
io.containerd.runtime.v1              linux                    linux/amd64    ok
io.containerd.runtime.v2              task                     linux/amd64    ok
io.containerd.monitor.v1              cgroups                  linux/amd64    ok
io.containerd.service.v1              tasks-service            -              ok
io.containerd.grpc.v1                 introspection            -              ok
io.containerd.internal.v1             restart                  -              ok
io.containerd.grpc.v1                 containers               -              ok
io.containerd.grpc.v1                 content                  -              ok
io.containerd.grpc.v1                 diff                     -              ok
io.containerd.grpc.v1                 events                   -              ok
io.containerd.grpc.v1                 healthcheck              -              ok
io.containerd.grpc.v1                 images                   -              ok
io.containerd.grpc.v1                 leases                   -              ok
io.containerd.grpc.v1                 namespaces               -              ok
io.containerd.internal.v1             opt                      -              ok
io.containerd.grpc.v1                 snapshots                -              ok
io.containerd.grpc.v1                 tasks                    -              ok
io.containerd.grpc.v1                 version                  -              ok
io.containerd.tracing.processor.v1    otlp                     -              skip
io.containerd.internal.v1             tracing                  -              ok
io.containerd.grpc.v1                 cri                      linux/amd64    ok
```

devmapper plugin error 是因为没有配置，可以暂时忽略，devmapper
概念可以参考：<https://pkg.go.dev/github.com/containerd/containerd/snapshots/devmapper#section-readme>

私有镜像仓库相关的配置在 cri 插件中，文档 Configure Image Registry中包含了镜像仓库的配置。
关于私有仓库和认证信息配置示例如下，修改/etc/containerd/config.toml：

```ini
...
[plugins]
...
  [plugins."io.containerd.grpc.v1.cri"]
  ...
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.my.org"]
          endpoint = ["https://harbor.my.org"]
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.my.org".tls]
          insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.my.org".auth]
          username = "username"
          password = "passwd"
          # auth = "base64(username:password)"
...
```

配置完成后重启 containerd，就可以使用 crictl pull 配置的私有仓库的镜像了：

```bash
crictl pull harbor.my.org/library/nginx:1.1
```

##### 2.2.2.3 nerdctl

参考：<https://github.com/containerd/nerdctl>

nerdctl 是一个非常棒的客户端：

- 保持了和 Docker 一致的使用习惯
- 甚至兼容 Docker-Compose
- 与 docker / ctr / crictl 相比，它又有独到之处

参考：<https://github.com/containerd/nerdctl#features-present-in-nerdctl-but-not-present-in-docker>

- [Lazy-pulling is a technique to running containers before completion of pulling the
  images.](https://github.com/containerd/nerdctl/blob/master/docs/stargz.md)
- [Image encryption and decryption](https://github.com/containerd/nerdctl/blob/master/docs/ocicrypt.md)：This
  command only encrypts image layers, but does NOT encrypt container configuration such as Env and
  Cmd
- [Distribute Container Images on IPFS](https://github.com/containerd/nerdctl/blob/master/docs/ipfs.md)

安装使用：

```bash
wget https://github.com/containerd/nerdctl/releases/download/v0.20.0/nerdctl-0.20.0-linux-amd64.tar.gz

tar zxvf nerdctl-0.20.0-linux-amd64.tar.gz -C /usr/bin/
nerdctl run -d --name nginx -p 80:80 nginx:alpine
ctr -n default i ls
```

可以看到：

1. 启动容器带 -p 需要 CNI 支持

   ```console
   [root@lab-kubernetes tmp]# nerdctl run -d --name nginx -p 80:80 nginx:alpine
   docker.io/library/nginx:alpine:
   FATA[0013] needs CNI plugin &{"bridge" "nerdctl0" %!q(bool=true) %!q(bool=false) %!q(bool=false) %!q(bool=true) '\x00' %!q(bool=true) %!q(bool=false) '\x00' map["ranges":[[map["gateway":"10.4.0.1" "subnet":"10.4.0.0/24"]]] "routes":[map["dst":"0.0.0.0/0"]] "type":"host-local"]} to be installed in CNI_PATH ("/opt/cni/bin"), see https://github.com/containernetworking/plugins/releases: exec: "/opt/cni/bin/bridge": stat /opt/cni/bin/bridge: no such file or directory
   ```

   安装一下 cni 就好

   ```bash
   mkdir -p /opt/cni/bin/
   cd /opt/cni/bin/
   wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
   tar zxvf cni-plugins-linux-amd64-v1.1.1.tgz
   ```

   然后再加 -p 运行命令就不会报错了

   ```console
   [root@lab-kubernetes tmp]# nerdctl run -d --name nginx -p 80:80 nginx:alpine
   5bc1f7fcae4e4891c7e57626286fbed1465739f2b28270ec90c88a82e1106a64

   [root@lab-kubernetes tmp]# docker ps
   CONTAINER ID   IMAGE     COMMAND                  CREATED       STATUS       PORTS     NAMES
   62970cd2a517   nginx     "/docker-entrypoint.…"   5 hours ago   Up 5 hours   80/tcp    admiring_yalow
   614b35ef3e28   nginx     "/docker-entrypoint.…"   5 hours ago   Up 5 hours   80/tcp    fervent_ardinghelli

   [root@lab-kubernetes tmp]# docker rm -f 62970cd2a517 614b35ef3e28
   62970cd2a517
   614b35ef3e28

   [root@lab-kubernetes tmp]# docker ps
   CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

   [root@lab-kubernetes tmp]# nerdctl ps -a
   CONTAINER ID    IMAGE                             COMMAND                   CREATED           STATUS    PORTS                 NAMES
   5bc1f7fcae4e    docker.io/library/nginx:alpine    "/docker-entrypoint.…"    38 seconds ago    Up        0.0.0.0:80->80/tcp    nginx
   ```

2. nerdctl pull 的镜像在 default namespace 下

   ```
   [root@lab-kubernetes tmp]# ctr -n default i ls | grep nginx
   docker.io/library/nginx:alpine       application/vnd.docker.distribution.manifest.list.v2+json sha256:a74534e76ee1121d418fa7394ca930eb67440deda413848bc67c68138535b989 9.7 MiB  linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x
   ```

综上：

1. containerd 可以有多个 namespaces，default/k8s.io/moby。这里的 namespace 不是 k8s 层面的，而是 containerd 自己的概念空间。参考
   <https://github.com/containerd/containerd/blob/main/docs/namespaces.md>

   _containerd offers a fully namespaced API so multiple consumers can all use a single containerd
   instance without conflicting with one another. Namespaces allow multi-tenancy within a single
   daemon. This removes the need for the common pattern of using nested containers to achieve this
   separation. Consumers are able to have containers with the same names but with settings and/or
   configurations that vary drastically. For example, system or infrastructure level containers can
   be hidden in one namespace while user level containers are kept in another. Underlying image
   content is still shared via content addresses but image names and metadata are separate per
   namespace._

   _It is important to note that namespaces, as implemented, is an administrative construct that is
   not meant to be used as a security feature. It is trivial for clients to switch namespaces._

2. 通过 kubelet/crictl 启动的容器，ns 就是 k8s.io，通过 docker 启动的就是 moby。docker ps 是看不到 k8s.io 下的容器的。对于
   containerd 而言， docker 和 kubelet 是两个不同的客户端。
3. nerdctl 和 ctr 默认都是 default namespaces
4. docker 启动的容器进程在 moby namespace，拉取的镜像不在 namespaces。
5. podman 不和 containerd 打交道，拉取的镜像和启动的容器都与 containerd namespace 无关。

##### 2.2.2.4 podman

podman 是 CRI-O 项目分裂出来的，由 redhat 发起。

参考：<https://github.com/containers/podman>，podman 致力于创造一个用户体验类似 docker，但没有 dockerd daemon 服务的工具。

参考：<https://podman.io/getting-started/>

```bash
yum install -y podman

podman pull docker.io/library/httpd
podman images

podman run -dt -p 8080:80/tcp docker.io/library/httpd
```

```console
[root@lab-kubernetes ~]# podman run -dt -p 8080:80/tcp docker.io/library/httpd
Trying to pull docker.io/library/httpd...
439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04

[root@lab-kubernetes ~]# ps -ef | grep httpd
root     22267 22256  0 17:01 pts/0    00:00:00 httpd -DFOREGROUND
33       22278 22267  0 17:01 pts/0    00:00:00 httpd -DFOREGROUND
33       22279 22267  0 17:01 pts/0    00:00:00 httpd -DFOREGROUND
33       22280 22267  0 17:01 pts/0    00:00:00 httpd -DFOREGROUND
root     22256     1  0 17:01 ?        00:00:00 /usr/bin/conmon --api-version 1 -s -c 439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04 -u 439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04 -r /usr/bin/runc -b /var/lib/containers/storage/overlay-containers/439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04/userdata -p /var/run/containers/storage/overlay-containers/439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04/userdata/pidfile -l k8s-file:/var/lib/containers/storage/overlay-containers/439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04/userdata/ctr.log --exit-dir /var/run/libpod/exits --socket-dir-path /var/run/libpod/socket --log-level error --runtime-arg --log-format=json --runtime-arg --log --runtime-arg=/var/run/containers/storage/overlay-containers/439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04/userdata/oci-log -t --conmon-pidfile /var/run/containers/storage/overlay-containers/439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04/userdata/conmon.pid --exit-command /usr/bin/podman --exit-command-arg --root --exit-command-arg /var/lib/containers/storage --exit-command-arg --runroot --exit-command-arg /var/run/containers/storage --exit-command-arg --log-level --exit-command-arg error --exit-command-arg --cgroup-manager --exit-command-arg systemd --exit-command-arg --tmpdir --exit-command-arg /var/run/libpod --exit-command-arg --runtime --exit-command-arg runc --exit-command-arg --storage-driver --exit-command-arg overlay --exit-command-arg --events-backend --exit-command-arg journald --exit-command-arg container --exit-command-arg cleanup --exit-command-arg 439e9d1532d0578af0d3d80e9b06f728ed06389a6d878430b8bee71aa74f5d04
```

可以看到：

- podman 有个称之为 conmon 的守护进程，它是各个容器进程的父进程，每个容器各有一个，conmon 的父进程是 1 号进程。podman 中的 conmon 相当于
  docker/containerd 中的 containerd-shim。

podman 相比较 docker 的优势：

- docker 在实现 CRI 时，需要一个守护进程（dockerd daemon），该进程需要以 root 运行（有安全隐患）。而 podman 不需要守护程序，因此也不需要 root 用户运行。
- 在 docker 的运行体系中，需要多个 daemon 才能调用到 OCI 的实现 RunC（dockerd 调用 containerd，containerd
  调用containerd-shim，然后才能调用 runC）podman 直接调用 OCI,runtime（runC），通过 conmon 作为容器进程的管理工具，不需要dockerd 这种以
  root 身份运行的守护进程。

#### 2.2.3 Containerd 手动部署

先移除之前安装的 containerd / runc / docker

```bash
yum remove runc -y
```

部署 Containerd

```bash
# 配置 containerd 所需 kernel module
cat << EOF > /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# 加载 kernel module
modprobe overlay
modprobe br_netfilter

# 修改内核参数
cat << EOF > /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
user.max_user_namespaces=28633
EOF
sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf

# 低内核版本 nf_conntrack_ipv4 / 高内核版本（4.x+） nf_conntrack
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack

yum install -y ipset ipvsadm

# 下载和安装 containerd
wget https://github.com/containerd/containerd/releases/download/v1.6.5/cri-containerd-cni-1.6.5-linux-amd64.tar.gz
tar -zxvf cri-containerd-cni-1.6.5-linux-amd64.tar.gz -C /
```

containerd 压缩包中包含了 crictl/ctr 命令行，cni，runc 和 containerd 自身服务。

```console
[root@lab-kubernetes tmp]# tar -zxvf cri-containerd-cni-1.6.5-linux-amd64.tar.gz -C /
etc/
etc/cni/
etc/cni/net.d/
etc/cni/net.d/10-containerd-net.conflist
etc/crictl.yaml
etc/systemd/
etc/systemd/system/
etc/systemd/system/containerd.service
usr/
usr/local/
usr/local/sbin/
usr/local/sbin/runc
usr/local/bin/
usr/local/bin/containerd-shim
usr/local/bin/containerd
usr/local/bin/critest
usr/local/bin/crictl
usr/local/bin/containerd-shim-runc-v1
usr/local/bin/containerd-stress
usr/local/bin/containerd-shim-runc-v2
usr/local/bin/ctd-decoder
usr/local/bin/ctr
opt/
opt/cni/
opt/cni/bin/
opt/cni/bin/static
opt/cni/bin/vlan
opt/cni/bin/bandwidth
opt/cni/bin/portmap
opt/cni/bin/ipvlan
opt/cni/bin/firewall
opt/cni/bin/macvlan
opt/cni/bin/dhcp
opt/cni/bin/ptp
opt/cni/bin/sbr
opt/cni/bin/vrf
opt/cni/bin/loopback
opt/cni/bin/host-device
opt/cni/bin/tuning
opt/cni/bin/bridge
opt/cni/bin/host-local
opt/containerd/
opt/containerd/cluster/
opt/containerd/cluster/gce/
opt/containerd/cluster/gce/cloud-init/
opt/containerd/cluster/gce/cloud-init/node.yaml
opt/containerd/cluster/gce/cloud-init/master.yaml
opt/containerd/cluster/gce/configure.sh
opt/containerd/cluster/gce/cni.template
opt/containerd/cluster/gce/env
opt/containerd/cluster/version
```

`cri-containerd-cni-1.6.5-linux-amd64.tar.gz` 包含的 runc 动态链接库与 CentOS 7 有兼容性问题。运行 runc
的时候，会遇到报错：`runc: undefined symbol: seccomp_notify_respond`

类似这样的 bug 我们可以用 hotfix 来修复（直接下载更新版本的 runc，或者自己编译 hotfix 的 runc）。

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.0-rc.1/runc.amd64
chmod +x runc.amd64
cp runc.amd64 /usr/local/sbin/runc

# 生成 containerd 配置文件
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

修改 /etc/containerd/config.toml

```ini
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
   ...
  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"
   ...
```

```bash
# 然后启动 containerd
systemctl enable containerd --now

# 检查 containerd 是否运行正常
ctr version
ctr plugin ls
crictl version
```

然后按需部署 docker

```bash
wget https://download.docker.com/linux/static/stable/x86_64/docker-20.10.9.tgz

tar -xvf docker-20.10.9.tgz
```

docker 包括如下二进制文件

```console
[root@lab-kubernetes tmp]# tar -xvf docker-20.10.9.tgz
docker/
docker/containerd-shim-runc-v2
docker/dockerd
docker/docker-proxy
docker/ctr
docker/docker
docker/runc
docker/containerd-shim
docker/docker-init
docker/containerd
```

安装和部署 docker

```bash
mv docker/docker* /usr/bin

cat <<EOF > /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=containerd.service

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd --containerd /run/containerd/containerd.sock --cri-containerd
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process
OOMScoreAdjust=-500

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl start docker
```

检查安装情况

```console
[root@lab-kubernetes tmp]# docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

[root@lab-kubernetes tmp]# docker version
Client:
 Version:           20.10.9
 API version:       1.41
 Go version:        go1.16.8
 Git commit:        c2ea9bc
 Built:             Mon Oct  4 16:03:22 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.9
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.8
  Git commit:       79ea9d3
  Built:            Mon Oct  4 16:07:30 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.6.5
  GitCommit:        96df0994faabc1944fc614e52b0b3c6feb609a57
 runc:
  Version:          1.1.0-rc.1
  GitCommit:        v1.1.0-rc.1-0-g55df1fc4
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

此时有个问题，用 nerdctl 运行带 -p 参数的容器会失败，报错 cni 版本不匹配。

检查 nerdctl 参数，发现默认 cni path 是 `/usr/libexec/cni/`，有三种方法：

1. 更新这个目录下的 cni（更新到 1.1.1 版本）
2. 或者该目录软链接到 `/opt/cni/bin/` 目录，此目录下的 cni 是 1.1.1 版本的
3. 再或者删除 `/usr/libexec/cni/` 目录（删除后，nerdctl 默认的 cni path 变成了 `/opt/cni/bin/`）

都可以解决此问题。

```console
[root@kubernetes004 ~]# nerdctl run -d --name nginx -p 80:80 nginx:alpine
FATA[0000] failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error running hook #0: error running hook: exit status 1, stdout: , stderr: time="2022-06-07T21:21:59+08:00" level=fatal msg="failed to call cni.Setup: plugin type=\"bridge\" failed (add): incompatible CNI versions; config is \"1.0.0\", plugin supports [\"0.1.0\" \"0.2.0\" \"0.3.0\" \"0.3.1\" \"0.4.0\"]"
Failed to write to log, write /var/lib/nerdctl/1935db59/containers/default/954e3d09f83bdce90305b2e8f58dad59a58ac5d9f350711a4025c7854067470c/oci-hook.createRuntime.log: file already closed: unknown

[root@kubernetes004 ~]# nerdctl -h | grep cni
      --cni-netconfpath string   cni config directory [$NETCONFPATH] (default "/etc/cni/net.d")
      --cni-path string          cni plugins binary directory [$CNI_PATH] (default "/usr/libexec/cni")

[root@kubernetes004 ~]# ls /usr/libexec/cni/
bandwidth    dhcp         flannel      host-local   loopback     portmap      sample       static       vlan
bridge       firewall     host-device  ipvlan       macvlan      ptp          sbr          tuning

[root@kubernetes004 ~]# rm -rf /usr/libexec/cni/

[root@kubernetes004 ~]# nerdctl -h | grep cni
      --cni-netconfpath string   cni config directory [$NETCONFPATH] (default "/etc/cni/net.d")
      --cni-path string          cni plugins binary directory [$CNI_PATH] (default "/opt/cni/bin")

[root@kubernetes004 ~]# nerdctl rm -f nginx
[root@kubernetes004 ~]# nerdctl run -d --name nginx -p 80:80 nginx:alpine
4958c5b1ad1fd2db2d8d1c4ccd0d796cd9115d4f07f17aa7a1fa615b89831c7a
```

### 2.3 CRI-O

[返回目录](#课程目录)

CRI-O 是 RedHat 发布的容器运行时，旨在同时满足 CRI 标准和 OCI 标准。kubelet 通过 CRI 与 CRI-O 交互，CRI-O 通过 OCI 与 runC
交互，追求简单明了。

![](/image/k8s-cri-o-flow.png)

CRI 接口是：`unix:///var/run/crio/crio.sock`

![](/image/k8s-cri-o-arch-2.jpg)

在同一个集群中，混用不同的 CRI 实验，参考：<https://gobomb.github.io/post/container-runtime-note/>

安装 cri-o，参考：<https://cri-o.io/>

```bash
OS=CentOS_7
VERSION=1.23
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
yum install -y cri-o
```

修改 kubelet 启动参数（也可以写在 EnvironmentFile 指定的文件里）：

```ini
vim /lib/systemd/system/kubelet.service

[Unit]
......
[Service]
......
ExecStart=/k8s/kubernetes/bin/kubelet $KUBELET_OPTS --container-runtime=remote --container-runtime-endpoint=/var/run/crio/crio.sock --cgroup-driver=systemd
......

[Install]
.......
```

启动 crio：

```bash
systemctl start crio

# 使用 crictl 查看版本
crictl version

# 重启 kubelet
systemctl restart kubelet
```

测试：

```console
$ crictl -r /var/run/crio/crio.sock version
Version:  0.1.0
RuntimeName:  cri-o
RuntimeVersion:  1.11.11-1.rhaos3.11.git474f73d.el7
RuntimeApiVersion:  v1alpha1

$ crictl -r /var/run/crio/crio.sock ps
CONTAINER           IMAGE                                                                                               CREATED             STATE               NAME                ATTEMPT             POD ID
fbe5b37ad3c47       docker.io/library/busybox@sha256:895ab622e92e18d6b461d671081757af7dbaa3b00e3e28e12505af7817f73649   About an hour ago   Running             busybox             0                   f9f42b5ae54c7

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

[root@node02 runc]# runc list
ID                                                                 PID         STATUS      BUNDLE                                                                                                                 CREATED                          OWNER
f9f42b5ae54c71180ab7eb1706205bba79597d019f1c3e22f5f68e0b0ae055a8   31252       running     /run/containers/storage/overlay-containers/f9f42b5ae54c71180ab7eb1706205bba79597d019f1c3e22f5f68e0b0ae055a8/userdata   2019-07-30T07:31:54.768990081Z   root
fbe5b37ad3c472ea970af75afc4c58481c2dd4d89a93b1a7ca37ddda823b201c   31785       running     /run/containers/storage/overlay-containers/fbe5b37ad3c472ea970af75afc4c58481c2dd4d89a93b1a7ca37ddda823b201c/userdata   2019-07-30T07:34:15.524324699Z   root
```

综上：

1. docker ps / ctr / nerdctl 看不到 crio 创建的容器
2. crictl / podman / runc 可以看到 pause 容器和主容器

### 2.4 Kata 和它的朋友们

[返回目录](#课程目录)

容器毕竟还是共享内核的（容器的天然不安全与天然安全），安全性和隔离型对于想要实现多租户是不够。所以又出现了许多基于虚拟机隔离的方案出来。

- [Kata Container](https://katacontainers.io/learn/)
- [Kata PDF](https://katacontainers.io/collateral/kata-containers-1pager.pdf)

![](/image/katacontainers-traditionalvskata-diagram.jpg)

![](/image/katacontainers-architecture-diagram.jpg)

实现 ECI 不只可以通过 K8S，也可以通过 OpenStack zun 来直接编排 Kata，参考
<https://github.com/cloudmaster2010/openstack/blob/main/devstack/3.zun-kata-kuryr.md>

Kata
安装步骤，参考：<https://github.com/kata-containers/kata-containers/blob/main/docs/install/README.md#kata-deploy-installation>

第一种安装方案：参考
<https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md>

1. **安装 snap**，参考：<https://snapcraft.io/docs/installing-snap-on-centos>
2. **通过 snap 安装
   kata-container**，参考：<https://github.com/kata-containers/kata-containers/blob/main/docs/install/snap-installation-guide.md>。如果报错：`too early for operation, device not yet seeded or device model not acknowledged`，可以参考：<https://blog.csdn.net/u010620626/article/details/117259178>，`sudo setenforce 0`
3. **安装
   containerd**，完成适配，参考：<https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md>，和
   <https://github.com/kata-containers/kata-containers/blob/main/docs/install/container-manager/containerd/containerd-install.md#install-containerd>

第二种安装方案，先安装好 containerd + k8s，再通过 kata-deploy
部署。参考：<https://github.com/kata-containers/kata-containers/blob/main/tools/packaging/kata-deploy/README.md>

除 Kata 以外，还有 Frakti / gVisor 等

- Frakti 提供了 hypervisor 级别的隔离性，提供的是内核级别的而非 Linux 命名空间级别的隔离：

  _Frakti lefts Kubernetes run pods and containers directly inside hypervisors via runV. It is light
  weighted and portable, but can provide much stronger isolation with independent kernel than
  linux-namespace-based container runtimes._
- gVisor 是拦截了系统调用，用自己实现用户态的进程而非内核来处理系统调用

  运行沙箱容器时，给出了支持安全沙箱容器运行时 handler runsc，我们需要创建一个 RuntimeClass 并在 pod spec 里指定是用该 RuntimeClass

      ```yaml
      apiVersion: node.k8s.io/v1beta1
      kind: RuntimeClass
      metadata:
      name: untrusted
      handler: runsc
      ```

  这里 runsc 时 gvisor 的 "runc"

  还需要修改 deployment 的 pod template

      ```yaml
      spec:
          runtimeClassName: untrusted
          containers:
          - image: vicuu/nginx:host
              imagePullPolicy: IfNotPresent
              name: nginx-host
      ```

### 2.5 GPU

[返回目录](#课程目录)

参考：<https://developer.nvidia.com/blog/announcing-containerd-support-for-the-nvidia-gpu-operator/>

Nvidia 官方提供的 containerd 支持步骤如下：

1. 安装 NVIDIA 容器运行时（需要先安装 Nvidia 和 CUDA 驱动）
2. 更新 containerd 配置文件.
3. 让 containerd 配置文件生效（发送 SIGHUP）
4. 跑 helm chart

也可以参考这里的步骤：<https://icloudnative.io/posts/add-nvidia-gpu-support-to-k8s-with-containerd/>

关于 GPU 的虚拟化：

1. Nvidia 官方推荐 MIG：<https://docs.nvidia.com/datacenter/cloud-native/kubernetes/mig-k8s.html>
2. 第四范式有一种基于 CUDA 的切分 GPU 方案，比 MIG 灵活，但不被 Nvidia
   官方支持。参考：<https://github.com/4paradigm/k8s-device-plugin>

### 2.6 最佳实践

1. 如果需要 debug 容器网络，可以先找到该容器进程的 pid，然后进入其 network namespace，就可以方便地调试（包括 ip / route / iptables / ping
   / traceroute / tcpdump 等命令皆可操作），可以从宿主机上抓容器虚拟网卡的网络包，然后进行分析。
2. 生产环境建议用 Containerd，稳定性、性能和生态支持都有明显优势
3. 如果生产环境中其它依赖服务需要 docker 支持，可以安装，与 Containerd 不冲突
4. 关于命令行，首选 crictl 代替，用法和 docker 命令一致，并且能兼容所有支持 CRI 接口的容器运行时，比如 containerd / CRI-O 等。
5. Image 相关的操作，containerd 可以用自带的 ctr 命令（容器相关的也可以用 ctr，但命令格式和 docker CLI 不一致）。如果是 CRI-O，用 podman。
6. 如果需要延迟加载、P2P 镜像服务、镜像加密存取等功能，可以使用 nerdctl
7. 关于用 RPM 包还是二进制安装，it depends。如果我们可以预见不会频繁 apply patch，比如基础服务/工具：python/git，甚至
   runc/containerd/docker，那么推荐 RPM。如果我们对安全或者 SLA 有较高的要求（等不及 rpm 提供方出 hotfix），自身技术实力也能充分保证，
   那建议二进制部署，这样方便及时升级（和 bug 修复）。
8. 关于 GPU 支持，Nvidia 之前提供了 Docker 运行时支持，后续提供了 Containerd 支持。建议用 containerd + K8S 搭配。
9. 注意，不同的 GPU 型号对应不同的应用场景，训练和推理不混用。
10. 关于切分 GPU 支持，建议用 Nvidia 官方提供的 MIG 方案，优点是官方支持，缺点是切分数量不够灵活。第四范式的方案切分数量灵活，但不推荐上生产，因为出问题后（比如 TF 或者
    Pytorch 版本兼容性问题，改方案只支持特定版本） Nvidia 会说不支持。
11. ECI 解决方案中，Kata 相对成熟，可以应用于 OpenStack Zun 或者 K8S。

## 3. K8S 生命周期管理

[返回目录](#课程目录)

K8S 的操作要记得参考：<https://kubernetes.io/>

[K8S 有哪些组件](https://kubernetes.io/zh/docs/concepts/architecture/#)？api-server、kube-scheduler、kube-controller、etcd、coredns、kubelete、kubeproxy

组件结构图

![](/image/k8s-architecture.png)

### 3.1 集群的创建删除、扩缩容、备份恢复

[返回目录](#课程目录)

生产环境配置

| Host-Node  | CPU  | RAM  | root  | etcd | kubelet | CRI   |
| ---------- | ---- | ---- | ----- | ---- | ------- | ----- |
| Master * 3 | 16C+ | 32G+ | 100G+ | 40G+ | 250G+   | 250G+ |
| Worker * N | 16C+ | 32G+ | 100G+ | N/A  | 250G+   | 250G+ |

1. ETCD 需要 SSD
2. 确保 kubelet 使用的磁盘容量大于容器运行时使用的磁盘！
3. 考虑如何限制容器内 inode 泄露或磁盘写满：CRI rootfs 限制方案
4. 考虑是否禁止 hostpath / local 类型的 storageclass
5. 备份需要对接额外的 NFS 或对象存储

参考部署架构

![](/image/openshift-ha-deployment.png)

1. Infra 节点能避免 Worker Node 直接对外暴露
2. ELB 提供 VIP，或者 LB 类型 service 对外提供 VIP

#### 3.1.1 KubeAdmin

##### 3.1.1.1 单节点集群部署

基于 Containerd 部署 K8S 1.23.3 集群

1. CentOS 7.9，升级 kernel，参考 [1.1.1 内核升级](#111-内核升级)

   ```bash
   rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
   yum install -y https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
   yum -y --disablerepo="*" --enablerepo="elrepo-kernel" list available
   yum -y --disablerepo=\* --enablerepo=elrepo-kernel install kernel-lt.x86_64
   awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

   grub2-set-default 0
   # 这里的 0 要根据实际情况来填写

   reboot
   ```

   重启后，`uname -a` 检查内核版本，可以看到

   ```console
   [root@kubernetes001 ~]# uname -a
   Linux kubernetes001 5.4.197-1.el7.elrepo.x86_64 #1 SMP Sat Jun 4 08:43:19 EDT 2022 x86_64 x86_64 x86_64 GNU/Linux
   ```

2. 部署 Containerd 和 ctictl 等工具，参考 [2.2.2.1 Crictl](#2221-crictl)

   ```bash
   # 如果之前安装 centos 默认源的 docker，要先删除掉
   yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

   yum install -y yum-utils
   yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
   yum install -y containerd.io
   # yum install -y docker-ce docker-ce-cli docker-compose-plugin

   systemctl enable containerd --now
   # systemctl enable docker --now
   ```

   然后用 ctr 检查 containerd 是否正常运行

   ```console
   [root@kubernetes001 ~]# ctr version
   Client:
   Version:  1.6.4
   Revision: 212e8b6fa2f44b9c21b2798135fc6fb7c53efc16
   Go version: go1.17.9

   Server:
   Version:  1.6.4
   Revision: 212e8b6fa2f44b9c21b2798135fc6fb7c53efc16
   UUID: 1cd1f73e-28ae-4f7c-9fa3-5ccdcdb2a23c
   ```

   _[可选]：然后部署 crictl_

   ```bash
   VERSION="v1.24.1"
   wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz

   tar zxvf crictl-$VERSION-linux-amd64.tar.gz
   mv crictl /usr/bin/
   ```

   接下来修改 containerd 配置，避开 gcr

   ```bash
   cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
   containerd config default > /etc/containerd/config.toml
   vi /etc/containerd/config.toml
   ```

   修改 `SystemdCgroup` 和 `sandbox_image`

   ```ini
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
   ...
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
       SystemdCgroup = true
   ...
   [plugins."io.containerd.grpc.v1.cri"]
       sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"
   ...
   ```

   然后，重启服务

   ```bash
   systemctl restart containerd
   # 检查版本，验证可用
   crictl version
   ```

3. 部署 Kubernetes

   ```bash
   # 3. 关闭防火墙（默认就是关闭的，不用做）
   # systemctl stop firewalld.service
   # systemctl disable firewalld.service

   # 4. 关闭 selinux（默认就是关闭的，不用做）
   # vi /etc/selinux/config
   # 将 SELINUX=enforcing 改为 SELINUX=disabled

   # 5. 关闭 swap（默认就是关闭的，不用做）
   # swapoff /dev/sda2
   # vi /etc/fstab
   # 在 swap 分区这行前加 # 禁用掉，保存退出
   # reboot

   # 6. 配置系统相关属性
   cat <<EOF > /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF

   sysctl -p
   sysctl --system

   modprobe br_netfilter
   echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
   echo 1 > /proc/sys/net/ipv4/ip_forward

   # 7. 配置yum源
   cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
           http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF

   # cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   # [kubernetes]
   # name=Kubernetes
   # baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
   # enabled=1
   # gpgcheck=0
   # repo_gpgcheck=0
   # gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
   # https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
   # EOF

   # 8. 安装 CRI

   # 10. 下载 kubernetes https://kubernetes.io/releases/
   export k8s_version="1.23.3"

   yum install -y kubelet-${k8s_version}-0 kubeadm-${k8s_version}-0 kubectl-${k8s_version}-0  --disableexcludes=kubernetes

   # 11. 启动 kubelet
   systemctl enable kubelet --now

   # 12. 用 kubeadm 初始化创建 K8S 集群 https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#options
   kubeadm init --cri-socket unix:///run/containerd/containerd.sock --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v${k8s_version} --pod-network-cidr=10.244.0.0/16

   # 13. 配置 .kube/config 用于使用 kubectl
   mkdir -p $HOME/.kube
   cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   chown $(id -u):$(id -g) $HOME/.kube/config

   # 15. 安装 calico
   kubectl apply -f https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/ansible-cloudlab-centos/playbooks/roles/init04-prek8s/files/calico-${k8s_version}.yml
   # kubectl apply -f https://raw.githubusercontent.com/99cloud/lab-openstack/master/src/ansible-cloudlab-centos/playbooks/roles/init04-prek8s/files/calico-${k8s_version}.yml

   # 看到 node Ready 就 OK
   kubectl get nodes

   # 16. [可选]：移除 taint
   kubectl taint nodes $(hostname) node-role.kubernetes.io/master-
   ```

##### 3.1.1.2 增加 CRI-O Worker 节点

拓扑结构：1 Master（Containerd） + 1 Worker（CRI-O）

此处采用 Containerd + CRI-O 混合 CRI 部署。生产环境中不会这么用（生产环境中会尽量用较少、较成熟的模块完成搭建，减少依赖，减少技术栈复杂度），这里这么实验是用于同时展示
Kubelet 对接不同 CRI 时的情况。

[CRIO 部署](#23-cri-o) 时注意，sandbox 要配置成 `registry.aliyuncs.com/google_containers`，`gcr.io` 的 pause
容器拉取不了，或者可以 podman 手动 load

##### 3.1.1.3 移除 Node 节点

参考：<https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/>

```bash
# 首先确认节点情况
kubectl get nodes

# 然后告诉 API Server 暂停该节点的调度
kubectl drain <node name>

# 等节点恢复正常后，需要加回，可以用
kubectl uncordon <node name>
```

##### 3.1.1.4 ETCD 操作和备份恢复

ETCD 命令行工具安装

```bash
# Ubuntu 环境上用 apt-get 安装
apt install etcd-client
```

其它环境直接下载二进制文件，参考：<https://github.com/etcd-io/etcd/releases>

```bash
ETCD_VER=v3.5.4

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
/tmp/etcd-download-test/etcdutl version
# start a local etcd server
/tmp/etcd-download-test/etcd

# write,read to etcd
/tmp/etcd-download-test/etcdctl --endpoints=localhost:2379 put foo bar
/tmp/etcd-download-test/etcdctl --endpoints=localhost:2379 get foo
```

通过 API Server 理解 ETCD 使用情况，测试 etcdctl 存取内容

```console
root@ckalab001:~# ps -ef | grep api | grep -i etcd
root       24761   24743  3 10:17 ?        00:06:53 kube-apiserver --advertise-address=172.31.43.206 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --insecure-port=0 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key

# 设置环境变量，ETCDCTL_API=3
root@ckalab001:~# export ETCDCTL_API=3

root@ckalab001:~# etcdctl --cert="/etc/kubernetes/pki/apiserver-etcd-client.crt" --key="/etc/kubernetes/pki/apiserver-etcd-client.key" --cacert="/etc/kubernetes/pki/etcd/ca.crt" --endpoints=https://127.0.0.1:2379 put /firstkey trystack
OK

root@ckalab001:~# etcdctl --cert="/etc/kubernetes/pki/apiserver-etcd-client.crt" --key="/etc/kubernetes/pki/apiserver-etcd-client.key" --cacert="/etc/kubernetes/pki/etcd/ca.crt" --endpoints=https://127.0.0.1:2379 get /firstkey
/firstkey
trystack
```

备份恢复，参考：<https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster>

```bash
# list 所有的 key
etcdctl --cert="/etc/kubernetes/pki/apiserver-etcd-client.crt" --key="/etc/kubernetes/pki/apiserver-etcd-client.key" --cacert="/etc/kubernetes/pki/etcd/ca.crt" --endpoints=https://127.0.0.1:2379 get --prefix --keys-only ""

# list 所有的 key & value
etcdctl --cert="/etc/kubernetes/pki/apiserver-etcd-client.crt" --key="/etc/kubernetes/pki/apiserver-etcd-client.key" --cacert="/etc/kubernetes/pki/etcd/ca.crt" --endpoints=https://127.0.0.1:2379 get --prefix ""

# backup & restore
etcdctl --cert="/etc/kubernetes/pki/apiserver-etcd-client.crt" --key="/etc/kubernetes/pki/apiserver-etcd-client.key" --cacert="/etc/kubernetes/pki/etcd/ca.crt" --endpoints=https://127.0.0.1:2379 snapshot save a.txt
# etcdctl --cert="/etc/kubernetes/pki/apiserver-etcd-client.crt" --key="/etc/kubernetes/pki/apiserver-etcd-client.key" --cacert="/etc/kubernetes/pki/etcd/ca.crt" --endpoints=https://127.0.0.1:2379 snapshot restore a.txt
```

##### 3.1.1.5 Master0 节点的备份恢复

Kubeadm 部署 k8s 集群时，master0 尤其重要，需要通过该节点来把证书推送给新加入的节点。当 master0 节点出现故障，可以利用备份恢复 master0 节点。通过 kubeadm
安装的 k8s，主节点包括两类的灾备恢复：

- etcd 数据存储恢复
- 主节点控制组件恢复

场景描述：

- 当前环境时 master0/master1/master2，三个 master 节点 HA
- 会尝试备份 master0 节点，再用 master3 替换 master0

**Etcd 数据备份**：定义 etcd-backup.yaml 文件，运行在 master0 上

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: backup
  namespace: kube-system
spec:
  schedule: "0 0 * * *" # 指定时间
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            # Same image as in /etc/kubernetes/manifests/etcd.yaml
            image: k8s.gcr.io/etcd-amd64:3.2.18
            env:
            - name: ETCDCTL_API
              value: "3"
            command: ["/bin/sh"]
            args: ["-c", "etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S_%Z).db"]
            volumeMounts:
            - mountPath: /etc/kubernetes/pki/etcd
              name: etcd-certs
              readOnly: true
            - mountPath: /backup
              name: backup
          restartPolicy: OnFailure
          # 指定master0节点执行
          nodeName: master0
          hostNetwork: true
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
              type: DirectoryOrCreate
          - name: backup
            hostPath:
              path: /tmp/etcd_backup/
              type: DirectoryOrCreate
```

实现思路：

1. 定义 CronJob，这个 pod 每天凌晨自动运行 (schedule: "0 0 * * *")，用户可以自定义运行时间
2. 运行在 master0 节点上，通过 nodeName 实现
3. 挂载了 master0 机器上的 /tmp/etcd_backup/ 作为备份
4. Args 参数中的
   `etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S_%Z).db`
   即为备份命令，按照时间的格式命名 etcd 的备份数据

**Etcd 数据恢复**：

```bash
# 1. 将 /etc/kubernetes/manifests/ kube-apiserver.yaml 文件移动到别处，停止 kube-apiserver 服务
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. 将 /etc/kubernetes/manifests/ etcd.yaml 文件移动到别处，停止 etcd server 服务
mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 3. 运行如下命令，将损坏的数据文件移至其他地方
mv /var/lib/etcd/* /tmp/

# 4. 运行以下命令，临时运行 docker 容器，将数据从备份里恢复到 /var/lib/etcd/
docker run --rm \
   -v '/tmp/etcd_backup/:/backup' \
   -v '/var/lib/etcd:/var/lib/etcd' \
   --env ETCDCTL_API=3 \
   'k8s.gcr.io/etcd-amd64:3.2.18' \
  /bin/sh -c "etcdctl snapshot restore '/backup/etcd-snapshot-xxx_UTC.db' ; mv /default.etcd/member/ /var/lib/etcd/"

# 5. 将/etc/kubernetes/manifests/kube-apiserver.yaml文件放回原来路径，恢复kube-api server服务
# 6. 将/etc/kubernetes/manifests/etcd.yaml文件放回原来路径，恢复etcd server服务
```

**主节点数据备份**主要包括三个部分：

- `/etc/kubernetes/` 目录下的所有文件(证书，manifest 文件)
- 用户主目录下 .kube/config 文件(kubectl 连接认证)
- /var/lib/kubelet/ 目录下所有文件(plugins 容器连接认证)

定义 k8s-master-backup.yaml 文件，运行在 master0 上。

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: k8s-master-backup
  namespace: kube-system
spec:
  # activeDeadlineSeconds: 100
  schedule: "5 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: k8s-master-backup
            image: docker.io/alpine:latest
            command: ["/bin/sh"]
            args: ["-c", "tar -zcvf /backup/k8s-master-$(ifconfig eth0 | grep 'inet addr:' | awk '{print $2}' | cut -c 6-)-$(date +%Y-%m-%d_%H:%M:%S_%Z).tar.gz /kubernetes /kubelet"]
            volumeMounts:
            - mountPath: /backup
              name: backup
            - mountPath: /kubernetes
              name: kubernetes
            - mountPath: /kubelet
              name: kubelet
          restartPolicy: OnFailure
          # 指定节点master0执行
          nodeName: master0
          hostNetwork: true
          volumes:
          - name: backup
            hostPath:
              path: /tmp/k8s_master_backup/
              type: DirectoryOrCreate
          - name: kubernetes
            hostPath:
              path: /etc/kubernetes/
              type: DirectoryOrCreate
          - name: kubelet
            hostPath:
              path: /var/lib/kubelet/
              type: DirectoryOrCreate
```

实现思路：

1. 通过 hostPath 方式挂载了 /etc/kubernetes 目录
2. 以 hostPath 方式挂载了 /var/lib/kubelet 目录
3. 以 hostNetwork: true 方式运行，能读取主机IP地址
4. 以 nodeName 方式，运行于 master0 节点
5. Backup 目录默认挂载于宿主机 /tmp/k8s_master_backup/

注意：为防止宿主机本身故障，建议备份文件应定期转移到备份机器上，编写脚本定时将备份文件传到备份机器上

创建脚本文件 filescp.sh，filescp.sh 文件内容

```bash
#!/bin/bash
scp -i .ssh/id_rsa /tmp/etcd_backup/ root@10.0.0.76:/tmp
scp -i .ssh/id_rsa /tmp/k8s_master_backup/ root@10.0.0.76:/tmp
```

```bash
# 加入脚本执行到计划任务
crontab -e

# 添加计划任务(计划每日0时30分执行，时间用户可自指定)
30 0 * * * /tmp/filescp.sh
```

**新建节点替换 master0**

1. 准备一个待加入的节点 master3

- 自行关闭防火墙 selinux 等
- 自行安装 docker 或者 containerd（未测试）
- 自行安装 kubeadm/kubelet/kubectl
- 若待加入集群使用了非 k8s.gcr.io 的仓库，请自行修改 /etc/docker/daemon.json 或者 /etc/containerd/config.toml

2. 生成 certificate-key(在master0节点执行)

   该过程遇上 warning 可以忽略。主要目的是让 kubeadm 自己处理证书，不用我们 scp

   ```console
   $ kubeadm init phase upload-certs --upload-certs
   # 或者
   $ kubeadm init phase upload-certs --upload-certs --config /tmp/.k8s/kubeadm.yaml

   b019bb8de1cbe3de4327a7a238e60faf6f865bc815cdcdeb86f1f6f116bad3df
   ```

3. 查看加入集群命令(master0节点执行)

   ```bash
   kubeadm token create --print-join-command
   kubeadm join 192.168.40.199:16443 --token e5wrs0.lqcem5us4a04tp5x --discovery-token-ca-cert-hash sha256:61c6754582a1ca7668770594acd1efa36a9c5c71a897517d8fb6f6c9db8ee314
   ```

4. 将 master3 节点加入 k8s 集群，充当控制节点(master3 节点上执行)

   ```bash
   kubeadm join apiserver.cluster.local:6443 --token 6g5enb.ub1pap31ty0zoym4     --discovery-token-ca-cert-hash sha256:a057ab2330c6dba1de3816a291ea3786a5a558d5737d94ef64c5c045bf1e5e1c --control-plane --certificate-key b019bb8de1cbe3de4327a7a238e60faf6f865bc815cdcdeb86f1f6f116bad3df
   ```

5. 查看 master3 是否加入成功

   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config

   kubectl get pod -A
   kubectl get node
   ```

6. 将 master0 从 k8s 集群删除(任意 master 节点执行)

   ```bash
   kubectl delete nodes master0

   # 在 master0 上运行 reset
   kubeadm reset
   ```

7. 获取 master0 节点的 hash 值(任意 master 节点执行)

   ```bash
   # 集群本身安装了 etcdctl 工具，可省略这几步
   tar -zxvf etcd-v3.3.4-linux-amd64.tar.gz
   cd etcd-v3.3.4-linux-amd64
   cp etcdctl /usr/local/sbin/

   # 获取 master0 节点 hash 值
   ETCDCTL_API=3 etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key member list
   ```

   输出：

   ```
   1203cdd3ad75e761, started, master0, https://192.168.40.180:2380, https://192.168.40.180:2379
   dda71d9d52b97028, started, master1, https://192.168.40.181:2380, https://192.168.40.181:2379
   dkeijf23cjd3445k, started, master2, https://192.168.40.182:2380, https://192.168.40.182:2379

   找到 master0 对应的 hash 值是：1203cdd3ad75e761
   ```

8. 根据 hash 删除 etcd 信息，执行如下命令:

   ```bash
   ETCDCTL_API=3 etcdctl --endpoints 127.0.0.1:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key member remove 1203cdd3ad75e761
   ```

9. 查看节点是否加入成功

   ```console
   $ kubectl get nodes

   NAME STATUS ROLES AGE VERSION
   master3 Ready control-plane,master 50s v1.20.6
   master1 Ready control-plane,master 35m v1.20.6
   master2 Ready control-plane,master 35m v1.20.6
   ```

##### 3.1.1.6 删除 K8S

参考：<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/>

`kubectl reset -f`

##### 3.1.1.7 常见的集群排错思路

参考：<https://kubernetes.io/docs/tasks/debug/debug-cluster/#cluster-failure-modes>

#### 3.1.2 Sealos

参考：<https://github.com/labring/sealos#quickstart>

先确定能 ssh 172.19.30.106（本机 IP） 成功，然后用下面的命令就可以一行搞定 AIO 的 K8S 部署了。ctr / crictl / nerdctl 都会一起装上。

```bash
wget -c https://sealyun-home.oss-cn-beijing.aliyuncs.com/sealos-4.0/latest/sealos-amd64 -O sealos && chmod +x sealos && mv sealos /usr/bin

sealos run labring/kubernetes:v1.24.0 labring/calico:v3.22.1 --masters 172.19.30.106
```

本地起了镜像仓库，可以看配置文件怎么配置的。

```console
[root@lab-k8s001 ~]# crictl images
IMAGE                                       TAG                 IMAGE ID            SIZE
sealos.hub:5000/calico/cni                  v3.22.1             2a8ef6985a3e5       80.5MB
sealos.hub:5000/calico/node                 v3.22.1             7a71aca7b60fc       69.6MB
sealos.hub:5000/calico/pod2daemon-flexvol   v3.22.1             17300d20daf93       8.46MB
sealos.hub:5000/calico/typha                v3.22.1             f822f80398b9a       52.7MB
sealos.hub:5000/coredns/coredns             v1.8.6              a4ca41631cc7a       13.6MB
sealos.hub:5000/etcd                        3.5.3-0             aebe758cef4cd       102MB
sealos.hub:5000/kube-apiserver              v1.24.0             529072250ccc6       33.8MB
sealos.hub:5000/kube-controller-manager     v1.24.0             88784fb4ac2f6       31MB
sealos.hub:5000/kube-proxy                  v1.24.0             77b49675beae1       39.5MB
sealos.hub:5000/kube-scheduler              v1.24.0             e3ed7dee73e93       15.5MB
sealos.hub:5000/pause                       3.7                 221177c6082a8       311kB
sealos.hub:5000/tigera/operator             v1.25.3             648350e58702c       44.8MB
```

优点：

1. 管理 K8S 集群生命周期，HA 集群，扩缩容，清空集群，自动恢复（_If any master is down, lvscare will remove the ipvs
   realserver, when master recover it will add it back_）
2. 可以从 sealos hub 下载和使用 OCI-compatible 的 openebs, minio, ingress, pgsql, mysql, redis 等插件
3. 证书直接签 99 年（这个是优点还是安全漏洞待考…… 而且为了这个还直接改了 kubeadm 源码），依赖 k8s API 证书的 kubefed 之类就不要每年更新证书了。
   - 参考：<https://blog.51cto.com/heyong/5149534>，修改 `cmd/kubeadm/app/constants/constants.go` 里的
     CertificateValidity
   - 参考：<https://www.infinisign.com/news/one-year-certs>，kubeadm 强制只签一年，不让传参签 99 年还是有安全方面考虑的
   - 参考：<https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/>，这里介绍了证书在
     upgrade 时或者手动执行时可以 renew

   ```console
   [root@lab-k8s001 ~]# kubeadm certs check-expiration
   [check-expiration] Reading configuration from the cluster...
   [check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

   CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
   admin.conf                 May 15, 2122 13:08 UTC   99y             ca                      no
   apiserver                  May 15, 2122 13:08 UTC   99y             ca                      no
   apiserver-etcd-client      May 15, 2122 13:08 UTC   99y             etcd-ca                 no
   apiserver-kubelet-client   May 15, 2122 13:08 UTC   99y             ca                      no
   controller-manager.conf    May 15, 2122 13:08 UTC   99y             ca                      no
   etcd-healthcheck-client    May 15, 2122 13:08 UTC   99y             etcd-ca                 no
   etcd-peer                  May 15, 2122 13:08 UTC   99y             etcd-ca                 no
   etcd-server                May 15, 2122 13:08 UTC   99y             etcd-ca                 no
   front-proxy-client         May 15, 2122 13:08 UTC   99y             front-proxy-ca          no
   scheduler.conf             May 15, 2122 13:08 UTC   99y             ca                      no

   CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
   ca                      May 15, 2122 13:08 UTC   99y             no
   etcd-ca                 May 15, 2122 13:08 UTC   99y             no
   front-proxy-ca          May 15, 2122 13:08 UTC   99y             no
   ```

缺点：

1. 不受支持的离线包要自己制作
2. 没有 Web UI 界面
3. 不支持集中管理多个 K8S 集群的生命周期
4. 没考虑升级、备份、恢复自动化（4.0 估计还没重构到这）
5. 修改了 kubeadm 的业务逻辑（ipvs 和 99 年），不 100% 兼容社区 kubeadm 了

#### 3.1.3 KubeKey

参考：<https://kubesphere.com.cn/docs/installing-on-linux/introduction/kubekey/>

安装步骤：<https://kubesphere.com.cn/docs/installing-on-linux/on-premises/install-kubesphere-on-bare-metal/>

```
# 配置好密钥，确定可以 ssh 免密登录需要安装 kubelet 的 node

yum install -y openssl openssl-devel
yum install -y socat
yum install -y epel-release
yum install -y conntrack-tools

export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v2.0.0 sh -

./kk create config --with-kubernetes v1.21.5

vi config-sample.yaml

# {name: node1, address: 172.19.30.105, internalAddress: 172.19.30.105, user: root}
# 这里 172.19.30.105 是 node IP
# node1 要写你期望的节点名称，安装时会根据这个设置 hostname 的

./kk create cluster -f config-sample.yaml

# 部署完成后，检查 pod
kubectl get pod -A
docker ps
```

优点：

1. 对 kubeadm 进行了封装，略改善了 HA 部署体验

缺点：

1. 支持的版本比较久远（2022.06.08，CentOS 7.9 kubeadm 官方 1.24.X 都出来了，kk 只支持到 1.23.0）
2. 最新支持 kubesphere 3.2.1 的 k8s 版本还是 1.21.5（1.22.x 实验性支持，1.23 未涉及）实测 1.23.3 没啥问题，但官方为宣称，总担心有坑……
3. 1.21.5 安装时还是默认用 docker 作为容器运行时，不是 containerd。可以 describe
   node：`Container Runtime Version:  docker://20.10.8`
4. 命令行工具 crictl / nerdctl 之类都没有装

槽点：

1. 没啥高级功能…… 为啥不直接用 kubeadm，封装干嘛呢……
2. 跑了一遍 kk
   之后，收到短信：`【阿里云】尊敬的 XXXX：云盾云安全中心检测到您的服务器：XXXX 出现了紧急安全事件：恶意脚本代码执行，建议您立即登录云安全中心控制台-安全告警处理 http://a.aliyun.com/XXXX 进行处理。`

   ![](/image/kubekey-aliyun-alert.png)

#### 3.1.4 KubeClipper

参考：<https://kubeclipper.io/>

源码：<https://github.com/kubeclipper-labs>，喜欢的话请帮忙 star

QuickStart：<https://github.com/kubeclipper-labs/kubeclipper/blob/master/README_zh.md#quick-start>

|                     | KubeClipper | Sealos | KubeKey | Kubeasz | KubeOperator | K0S |
| ------------------- | ----------- | ------ | ------- | ------- | ------------ | --- |
| 图形化页面               | ✅           | ❌      | ❌       | ❌       | ✅            | ❌   |
| 轻依赖 - 不依赖 ansible 等 | ✅           | ✅      | ✅       | ❌       | ❌            | ✅   |
| 多区域、多集群管理           | ✅           | ❌      | ❌       | ❌       | ✅            | ❌   |
| 支持多版本 K8S、CRI       | ✅           | ✅      | ✅       | ✅       | ✅            | ✅   |
| 离线安装                | ✅           | ✅      | ✅       | ✅       | ✅            | ✅   |
| 基于 kubeadm 封装       | ✅           | ✅      | ✅       | ❌       | ❌            | ❌   |

#### 3.1.5 K3S

##### 3.1.5.1 什么 K3S

K3S 参考文档

- [Kubernetes 官网](https://kubernetes.io/zh)
- [Kubernetes 官方文档](https://kubernetes.io/zh/docs/home)
- [k3s 搭建步骤](https://github.com/k3s-io/k3s#quick-start---install-script)

Kubernetes 集群在管理大规模服务上有极佳的运维自动化优势（自愈 / 扩缩容 / 负载均衡 / 服务注册和服务发现 /
部署），但也需要耗费相对较多的资源，许多功能对特定对特定场景来说是冗余的，K3S 作为一个轻量级的 Kubernetes 发行版应运而生，它针对边缘计算、物联网等场景进行了高度优化。

K3s 有以下增强功能：

- 打包为单个二进制文件。
- 使用基于 sqlite3 的轻量级存储后端作为默认存储机制。同时支持使用 etcd3、MySQL 和 PostgreSQL 作为存储机制。
- 封装在简单的启动程序中，通过该启动程序处理很多复杂的 TLS 和选项。
- 默认情况下是安全的，对轻量级环境有合理的默认值。
- 添加了简单但功能强大的 batteries-included 功能，例如：本地存储提供程序，服务负载均衡器，Helm controller 和 Traefik Ingress
  controller。
- 所有 Kubernetes control-plane 组件的操作都封装在单个二进制文件和进程中，使 K3s 具有自动化和管理包括证书分发在内的复杂集群操作的能力。
- 最大程度减轻了外部依赖性，K3s 仅需要 kernel 和 cgroup 挂载。 K3s 软件包需要的依赖项包括：
  - containerd
  - Flannel
  - CoreDNS
  - CNI
  - 主机实用程序（iptables、socat 等）
  - Ingress controller（Traefik）
  - 嵌入式服务负载均衡器（service load balancer）
  - 嵌入式网络策略控制器（network policy controller）

![](/image/how-it-works-k3s.svg)

K3S 适用于：

- 边缘计算-Edge / 物联网-IoT / 嵌入式 K8S
- CI / DevOps

##### 3.1.5.2 单节点架构部署

![](/image/k3s-architecture-single-server.png)

K3s 单节点集群的架构如下图所示，该集群有一个**内嵌 SQLite** 数据库的单节点 **K3s server**。

在这种配置中，每个 **agent** 节点都注册到**同一个** server 节点。K3s 用户可以通过调用 server 节点上的 K3s API 来操作 Kubernetes
资源。外部流量则通过 Traeffic 导入，且经过 loadBalance 进行负载均衡。

部署准备：参阅 [K3S 官网配置改动](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/_index)

| 节点           | CPU    | 内存  | 磁盘   | 数量 | os        |
| ------------ | ------ | --- | ---- | -- | --------- |
| server/agent | >2Core | >1G | >10G | 1  | centos7.x |

k3s 默认使用 containerd 作为 cri，也使用更为熟悉的 docker 作为集群 cri，先安装 docker

```bash
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum install docker-ce-19.03.9 -y
mkdir -p /etc/docker
echo '{"registry-mirrors": ["http://hub-mirror.c.163.com"]}'>/etc/docker/daemon.json
systemctl enable --now docker
```

部署脚本安装（近期 k3s 和 autok3s 都在调整中，rancher-mirror.cnrancher.com
在归档，功能受影响，rancher-mirror.oss-cn-beijing.aliyuncs.com 尚未能全功代替）

```bash
# curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--docker" sh -
# 受限国内网络，大多数时候上述脚本无法安装，建议采用下述国内加速安装脚本
curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--docker" sh -
```

如果不用 docker 的话，改成

```bash
curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

输出下述日志，即部署完成

```log
...
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s
```

查看 pod 是否正常即可

```console
$ kubectl get po -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   helm-install-traefik-crd-j4bwn            0/1     Completed   0          2m10s
kube-system   helm-install-traefik-sxnmw                0/1     Completed   0          2m10s
kube-system   metrics-server-86cbb8457f-47kzz           1/1     Running     0          2m10s
kube-system   local-path-provisioner-5ff76fc89d-7bzjd   1/1     Running     0          2m10s
kube-system   coredns-7448499f4d-cjph9                  1/1     Running     0          2m10s
kube-system   svclb-traefik-8q77r                       2/2     Running     0          83s
kube-system   traefik-97b44b794-gb72h                   1/1     Running     0          83s

# 集群基础使用
# 我们来创建一个 deployment 的 pod
$ kubectl create deployment nginx --image=nginx
$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-pc4wz   1/1     Running   0          101s
```

网络选项

```console
# 修改 CNI 网络模式
# 默认情况下，K3S 将以 flannel 作为 CNI 运行，使用 VXLAN 作为默认后端。如需修改 CNI 的模式，可以参考下述命令安装
# --flannel-backend=vxlan   (默认) 使用 VXLAN 后端。
# --flannel-backend=ipsec	使用 IPSEC 后端，对网络流量进行加密。
# --flannel-backend=host-gw	使用 host-gw 后端。
# --flannel-backend=wireguard	使用 WireGuard 后端，对网络流量进行加密。可能需要额外的内核模块和配置。
$ curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--docker --flannel-backend=vxlan" sh -

# 查看对应的 linux 网络设备是否生成
$ ip a | grep flannel

31: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    inet 10.42.0.0/32 scope global flannel.1

# 替换 CNI
# K3S 除了支持 flannel，还支持 calico\canel 等 CNI 组件
# 我们使用 calico 作为网络组件

# 注：下述 10.96.0.0/16 不能跟实际网络冲突
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--flannel-backend=none --cluster-cidr=10.96.0.0/16 --disable-network-policy --disable=traefik" sh -

# 下载 calico.yaml
$ wget https://docs.projectcalico.org/manifests/calico.yaml
# 修改 calico.yaml 中被注释掉的参数 CALICO_IPV4POOL_CIDR
- name: CALICO_IPV4POOL_CIDR
  value: "10.96.0.0/16"

# apply calico
kubectl apply -f calico.yaml

# 查看集群 pod
$ kubectl get po -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-node-6cv7v                          1/1     Running   0          3m30s
kube-system   local-path-provisioner-5ff76fc89d-sqdts    1/1     Running   0          33m
kube-system   calico-kube-controllers-74b8fbdb46-mdwjm   1/1     Running   0          3m30s
kube-system   metrics-server-86cbb8457f-5xgrs            1/1     Running   0          33m
kube-system   coredns-7448499f4d-mwtmw                   1/1     Running   0          33m
```

K3S 卸载

```console
# 要从 server 节点卸载 K3s
$ /usr/local/bin/k3s-uninstall.sh

# 要从 agent 节点卸载 K3s
$ /usr/local/bin/k3s-agent-uninstall.sh
```

##### 3.1.5.3 K3S HA 架构部署

![](/image/k3s-architecture-ha-server.png)

一个高可用 K3s 集群由以下几个部分组成：

- **K3s Server 节点**：两个或更多的 server 节点将为 Kubernetes API 提供服务并运行其他 control-plane 服务
- **外部数据库**：与单节点 k3s 设置中使用的嵌入式 SQLite 数据存储相反，高可用 K3s 需要**挂载一个external database 外部数据库**作为数据存储的媒介。

**固定 agent 节点的注册地址**

在高可用 K3s server 配置中，每个节点还必须使用固定的注册地址向 **Kubernetes API** 注册，注册后，agent 节点直接与其中一个 server 节点建立连接，如下图所示：

![](/image/k3s-production-setup.svg)

先决条件：节点不能有相同的主机名

| 节点角色   | CPU    | 内存  | 磁盘   | 数量 | os        |
| ------ | ------ | --- | ---- | -- | --------- |
| server | >2Core | >4G | >10G | >3 | centos7.x |
| agent  | >2Core | >2G | >10G | >3 | centos7.x |

部署外部数据库 etcd。K3S 支持多种外部数据库，这里我们选用 etcd 数据库，为方便搭建，我们这里搭建的 etcd 集群为非 TLS 访问方式 更多详情内容参阅
[K3S 外部数据库配置](https://docs.rancher.cn/docs/k3s/installation/ha/_index) 和
[etcd 官方文档](https://etcd.io)

```bash
# 所有节点均部署 docker，与上述单节点部署过程一样，不再赘述
# etcd 高可用部署
# 三台 master 节点上均执行下述的 etcd 操作流程

wget https://github.com/etcd-io/etcd/releases/download/v3.3.15/etcd-v3.3.15-linux-amd64.tar.gz $
tar -vxf etcd-v3.3.15-linux-amd64.tar.gz $ cp etcd-v3.3.15-linux-amd64/etcd /usr/bin/ $ cp
etcd-v3.3.15-linux-amd64/etcdctl /usr/bin/

# etcd 采用 systemd 托管服务

vi /etc/systemd/system/etcd.service

# 填入 etcd 配置，保存退出。（etcd.service 配置见附录）
```

```console
$ systemct start etcd $ systemct status etcd ● etcd.service - etcd Loaded: loaded
(/etc/systemd/system/etcd.service; disabled; vendor preset: disabled) Active: active (running) since
Sun 2021-10-03 18:19:52 CST; 10s ago Docs: https://github.com/etcd-io/etcd

$ etcdctl member list member 763586516b512018 is healthy: got healthy result from
http://{etcd1-ip}:2379 member 78bb25a565876d17 is healthy: got healthy result from
http://{etcd2-ip}:2379 member da51b2723088b522 is healthy: got healthy result from
http://{etcd3-ip}:2379
```

注：后续如果重装 k3s 集群，请先停止所有 etcd 服务，并清理所有 etcd 节点的数据，最后重启 etcd 即可

```bash
systemctl stop etcd
rm -rf /var/lib/etcd
systemctl restart etcd
```

###### 3.1.5.3.1 server HA

下述命令中的 INSTALL_K3S_MIRROR,K3S_DATASTORE_ENDPOINT, K3S_TOKEN... 等可以作为环境变量，如 export
K3S_TOKEN="k3s_token"

```console
$ curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_DATASTORE_ENDPOINT='http://{etcd1-ip}:2379,http://{etcd2-ip}:2379,http://{etcd3-ip}:2379' K3S_TOKEN="k3s_token" INSTALL_K3S_VERSION=v1.18.6+k3s1 INSTALL_K3S_EXEC="--docker" sh -s - server
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s

$ kubectl get no
NAME         STATUS   ROLES                  AGE     VERSION
caasnode-1   Ready    control-plane,master   2m58s   v1.21.4+k3s1
caasnode-2   Ready    control-plane,master   78s     v1.21.4+k3s1
caasnode-3   Ready    control-plane,master   82s     v1.21.4+k3s1
```

###### 3.1.5.3.2 加入 agent

在所有 agent 节点中执行 master-ip 可以是多个 master 间共用的 vip 或者第一个 master 的 ip。

下述命令中的 INSTALL_K3S_MIRROR,K3S_DATASTORE_ENDPOINT, K3S_TOKEN... 等可以作为环境变量，如 export
K3S_TOKEN="k3s_token"

```console
$ curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_TOKEN="k3s_token" K3S_URL=https://{master-ip}:6443 INSTALL_K3S_VERSION=v1.18.6+k3s1 INSTALL_K3S_EXEC="--docker" sh -
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
[INFO]  systemd: Starting k3s-agent

$ kubectl get no
caasbastion-3   Ready    <none>                 3m20s   v1.21.4+k3s1
caasbastion-2   Ready    <none>                 2m49s   v1.21.4+k3s1
caasbastion-1   Ready    <none>                 2m27s   v1.21.4+k3s1
caasnode-1      Ready    control-plane,master   21m     v1.21.4+k3s1
caasnode-2      Ready    control-plane,master   19m     v1.21.4+k3s1
caasnode-3      Ready    control-plane,master   19m     v1.21.4+k3s1
```

###### 3.1.5.3.3 K3S 卸载

```console
# 要从 server 节点卸载 K3s
$ /usr/local/bin/k3s-uninstall.sh

# 要从 agent 节点卸载 K3s
$ /usr/local/bin/k3s-agent-uninstall.sh

# 如需重新部署，需要清理 etcd 的 data 目录，文档默认为 '/var/lib/etcd'
```

###### 3.1.5.3.4 附录

etcd.service

```console
# /etc/systemd/system/etcd.service
# etcd-ip: 本机 ip
# etcd1-ip: 本机 ip
# etcd2-ip: 第二台节点 ip
# etcd3-ip: 第三台节点 ip
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd
After=network-online.target

[Service]
Type=notify
Restart=on-failure
RestartSec=5s
TimeoutStartSec=0
ExecStart=/usr/bin/etcd --name etcd1 \
--heartbeat-interval 300 \
--election-timeout 1500 \
--initial-advertise-peer-urls http://{etcd-ip}:2380 \
--listen-peer-urls http://{etcd-ip}:2380 \
--listen-client-urls http://{etcd-ip}:2379,http://127.0.0.1:2379 \
--advertise-client-urls http://{etcd-ip}:2379 \
--data-dir /var/lib/etcd \
--initial-cluster etcd1=http://{etcd1-ip}:2380,etcd2=http://{etcd2-ip}:2380,etcd3=http://{etcd3-ip}:2380 \
--initial-cluster-token etcd-cluster \
--initial-cluster-state new \
--auto-compaction-retention=1 \
--snapshot-count=5000  \
--quota-backend-bytes=171798691840 \
--heartbeat-interval=100 \
--election-timeout=500
ExecReload=/bin/kill -HUP
KillMode=process

[Install]
WantedBy=multi-user.target
```

#### 3.1.6 AutoK3S

参考：<https://github.com/cnrancher/autok3s>

```bash
curl -sS https://rancher-mirror.oss-cn-beijing.aliyuncs.com/autok3s/install.sh | INSTALL_AUTOK3S_MIRROR=cn sh -

# The commands will start autok3s daemon with an interactionable UI.
autok3s -d serve --bind-address 0.0.0.0
```

注意：

1. 默认会启动在 8080 端口
2. 创建集群时，建议打开高级选项，用 oss 上的脚本部署（避免后续安装存在网络延迟），enable cluster etcd，选 explorer
3. explorer 对每一个集群都是一个单独的进程，以 kubeconfig 为启动参数，为一个 k8s 集群提供启动界面服务（kube-explorer-ui），然后 autok3s-ui
   集成各个 kube-explorer-ui（通过 proxy）。

优点：

1. 公有云 & native provider
2. 自定义配置参数
3. 实时 log
4. kubectl web console
5. rollback

缺点：

1. 只支持 k3s

#### 3.1.7 K0S

参考：<https://docs.k0sproject.io/v1.23.6+k0s.2/install/>

1. 参考：<https://docs.k0sproject.io/v1.23.6+k0s.2/system-requirements/#host-operating-system> 内核必须先升级到
   4.3 以上
2. 得弄好离线包…… load 好镜像再装……
3. 有延迟…… 2022.06.08 CentOS 7.9 1.24.x 都出来了，这最新还在 1.23.6

### 3.2 版本升级

[返回目录](#课程目录)

参考：<https://v1-23.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/>

#### 3.2.1 Ubuntu 案例

```bash
alias k=kubectl

k drain cluster3-master1 --ignore-daemonsets
apt-mark unhold kubeadm
apt-mark hold kubectl kubelet
apt install kubeadm=1.23.1-00
apt-mark hold kubeadm

kubeadm upgrade plan
kubeadm upgrade apply v1.23.1

apt update
apt-mark unhold kubelet kubectl
apt install kubelet=1.23.1-00 kubectl=1.23.1-00
apt-mark hold kubelet kubectl
service kubelet restart
service kubelet status
kubectl get node

k uncordon cluster3-master1

k drain cluster3-worker1 --ignore-daemonsets

apt update
apt-mark unhold kubeadm
apt-mark hold kubectl kubelet
apt install kubeadm=1.23.1-00
apt-mark hold kubeadm
kubeadm upgrade node

apt-mark unhold kubectl kubelet
apt install kubelet=1.23.1-00 kubectl=1.23.1-00
service kubelet restart
service kubelet status
k get node
k uncordon cluster3-worker1
```

#### 3.2.2 CentOS 案例

先升级 Master 节点

```bash
yum list --showduplicates kubeadm --disableexcludes=kubernetes
# find the latest 1.23 version in the list
# it should look like 1.23.x-0, where x is the latest patch

# replace x in 1.23.x-0 with the latest patch version
yum install -y kubeadm-1.23.x-0 --disableexcludes=kubernetes

# Verify that the download works and has the expected version:
kubeadm version

# Verify the upgrade plan:
kubeadm upgrade plan

# replace x with the patch version you picked for this upgrade
sudo kubeadm upgrade apply v1.23.x

# Same as the first control plane node but use:
sudo kubeadm upgrade node
# instead of: sudo kubeadm upgrade apply
# Also calling "kubeadm upgrade plan" and upgrading the CNI provider plugin is no longer needed.

# replace <node-to-drain> with the name of your node you are draining
kubectl drain <node-to-drain> --ignore-daemonsets

# replace x in 1.23.x-0 with the latest patch version
yum install -y kubelet-1.23.x-0 kubectl-1.23.x-0 --disableexcludes=kubernetes

# Restart the kubelet:
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# replace <node-to-drain> with the name of your node
kubectl uncordon <node-to-drain>
```

再升级 Worker 节点，参考
<https://v1-23.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-worker-nodes>

### 3.3 迁移和纳管

[返回目录](#课程目录)

##### 3.3.1 Rancher 和 K3S

在实际边缘生成应用中，我们边缘环境可能分布着多个 k3s 集群，当集群数量过多时，后续的运维管理会变得非常的不方便。为了解决这样的问题，下面我们通过云端中心的 rancher 去纳管边缘的多个 k3s

![](/image/cloud-rancher.png)

| 节点角色    | CPU    | 内存  | 磁盘   | 数量 | os        |
| ------- | ------ | --- | ---- | -- | --------- |
| rancher | >2Core | >4G | >10G | 1  | centos7.x |
| k3s     | >2Core | >2G | >10G | n  | centos7.x |

##### 3.3.1.1 部署 K3S 和 rancher

[部署 K3S](#223-单节点部署)

部署 rancher 服务

```console
$ docker run -d --restart=unless-stopped -p 1234:80 -p 2234:443 rancher/rancher:v2.3.6
```

##### 3.3.1.2 纳管 k3s

通过浏览器访问 https://{rancher_ip}:2234/

![](/image/rancher-addcluster.png)

![](/image/rancher-import-cluster.png)

![](/image/rancher-create.png)

![](/image/rancher-agent.png)

复制上图中的命令，到 k3s 端运行

```bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.3.6 --server https://172.20.149.95:2234 --token ndhsgr9msksr4s6g6xqq6dl8cns42txlkxt96tsdwcl56hds568nbb --ca-checksum 5b778f411871d7caca521a0cf25cd40d8a7e28aeb1def47af829e22092c114cd --etcd --controlplane --worker
```

最后通过界面我们能看到 rancher 完成了对 k3s 集群的纳管

![](/image/rancher-cluster.png)

同样，后续的其余 k3s 也是用这个方式去纳管，就完成中心端对边缘端集群的管理。

### 3.4 统一认证和鉴权

[返回目录](#课程目录)

#### 3.4.1 KubeSphere

先装好
K8S，准备检查：<https://kubesphere.io/docs/installing-on-kubernetes/introduction/prerequisites/>。注意：**要有
default 的 storageclass**，可以用 NFS 配置一个。

然后安装 KubeSphere：参考 <https://kubesphere.io/docs/quick-start/minimal-kubesphere-on-k8s/>。

### 3.5 监控、计量和告警

[返回目录](#课程目录)

### 3.6 日志管理

[返回目录](#课程目录)

### 3.7 最佳实践

1. K8S 的操作要记得参考：<https://kubernetes.io>
2. 部署 K8S 节点时，要记得 root / kubelet / etcd / 备份 / cri 用不同的存储分区，etcd 要注意用 ssd
3. K8S 架构中，一般会增加 infra 节点来避免 worker node 直接对外暴露
4. 生产环境中：推荐 kubeadm，K8S + Kubeadm + Calico + KubeSphere
5. 边缘环境：推荐 K3S + AutoK3S + Rancher
6. KubeKey / Kubeasz / K0S 各有优势，但需要学习和深入理解（Debug）不同的工具和技术栈
7. Kubeadm 有良好稳定可靠的升级方案、集群的备份恢复方案、以及 Master 0 节点的备份恢复方案
8. KubeSphere 能良好地纳管（统一管理、认证健全、监控计量、跨集群调度）K8S；而 Rancher 可以对 K3S 较好地统一管理。

## 4. 存储管理

[返回目录](#课程目录)

### 4.1 CSI 一般概念

[返回目录](#课程目录)

### 4.2 对接 NFS 和 NAS

[返回目录](#课程目录)

#### 4.2.1 搭建 NFS Server

1. 准备好 NFS server 机器（另开一台 CentOS 7.9，单独配一块数据盘），机器规格按需指定。最好挂载一块单独的磁盘。
2. 保证需要使用 NFS 存储的客户端机器与 NFS server 机器的网络互通性
3. 部署步骤
   1. 下载相关包，并启动相关服务

      ```bash
      apt-get install nfs-common -y || yum install nfs-utils -y
      ```

   2. 创建 NFS 数据路径

      ```bash
      mkdir -p /nfs/data
      chmod -R 777 /nfs/data
      ```

   3. 如果有额外挂单独的数据盘给 NFS 用，需要格式化这块磁盘并挂载，比如 vdb。如果没有额外挂盘请忽略这一步

      ```bash
      mkfs.xfs /dev/vdb
      mount /dev/vdb /nfs/data
      echo "/dev/vdb /nfs/data xfs defaults 0 0" >> /etc/fstab
      ```

   4. 编辑 NFS 配置文件

      ```bash
      # echo "/nfs/data *(rw,no_root_squash,sync)" > /etc/exports
      echo "/nfs/data *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)" > /etc/exports
      # 首次登入Internal error occurred: account is not active 的问题，https://kubesphere.com.cn/forum/d/2058-kk-internal-error-occurred-account-is-not-active
      exportfs -r
      ```

   5. 启动 rpcbind、nfs 服务

      ```bash
      systemctl restart rpcbind && systemctl enable rpcbind
      systemctl restart nfs && systemctl enable nfs
      ```

   6. 查看 RPC 服务的注册状况

      ```bash
      rpcinfo -p localhost
      program vers proto   port  service
          ...
          100003    3   tcp   2049  nfs
          100003    4   tcp   2049  nfs
          100227    3   tcp   2049  nfs_acl
          100003    3   udp   2049  nfs
          100003    4   udp   2049  nfs
          100227    3   udp   2049  nfs_acl
          ...
      ```

#### 4.2.2 使用动态 PersistentVolume

1. 创建 RBAC.yaml 文件，内容如下。

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: default
   ---
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: nfs-client-provisioner-runner
   rules:
     - apiGroups: [""]
       resources: ["persistentvolumes"]
       verbs: ["get", "list", "watch", "create", "delete"]
     - apiGroups: [""]
       resources: ["persistentvolumeclaims"]
       verbs: ["get", "list", "watch", "update"]
     - apiGroups: ["storage.k8s.io"]
       resources: ["storageclasses"]
       verbs: ["get", "list", "watch"]
     - apiGroups: [""]
       resources: ["events"]
       verbs: ["create", "update", "patch"]
   ---
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: run-nfs-client-provisioner
   subjects:
     - kind: ServiceAccount
       name: nfs-client-provisioner
       # replace with namespace where provisioner is deployed
       namespace: default
   roleRef:
     kind: ClusterRole
     name: nfs-client-provisioner-runner
     apiGroup: rbac.authorization.k8s.io
   ---
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: default
   rules:
     - apiGroups: [""]
       resources: ["endpoints"]
       verbs: ["get", "list", "watch", "create", "update",     "patch"]
   ---
   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: default
   subjects:
     - kind: ServiceAccount
       name: nfs-client-provisioner
       # replace with namespace where provisioner is deployed
       namespace: default
   roleRef:
     kind: Role
     name: leader-locking-nfs-client-provisioner
     apiGroup: rbac.authorization.k8s.io
   ```

2. 执行命令创建 RBAC

   ```bash
   kubectl create -f RBAC.yaml
   ```

3. 创建 deployment.yaml 文件，内容如下:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nfs-client-provisioner
     labels:
       app: nfs-client-provisioner
     namespace: default
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nfs-client-provisioner
     strategy:
       type: Recreate
     selector:
       matchLabels:
         app: nfs-client-provisioner
     template:
       metadata:
         labels:
           app: nfs-client-provisioner
       spec:
         serviceAccountName: nfs-client-provisioner
         containers:
           - name: nfs-client-provisioner
             image: 99cloud/nfs-subdir-external-provisioner:v4.0.0
             volumeMounts:
               - name: nfs-client-root
                 mountPath: /persistentvolumes
             env:
               - name: PROVISIONER_NAME
                 value: fuseim.pri/ifs
               - name: NFS_SERVER
                 value: localhost
               - name: NFS_PATH
                 value: /nfs/data
         volumes:
           - name: nfs-client-root
             nfs:
               server: localhost
               path: /nfs/data
   ```

   > 其中 localhost 请替换成 NFS Server 的地址

   如果 driver 起不来，多半是 client 端没有安装 nfs-common(ubuntu) 或者 nfs-utils(centos)

4. 部署deploy

   ```bash
   kubectl create -f deployment.yaml
   ```

5. 创建 storageclass.yaml 文件

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: nfs
   provisioner: fuseim.pri/ifs
   parameters:
     archiveOnDelete: "false"
   reclaimPolicy: Delete
   ```

   > provisioner 要对应 驱动所传入的环境变量 PROVISIONER_NAME 的值。

6. 创建 storage class

   ```bash
   kubectl apply -f storageclass.yaml
   ```

7. 如果要把 nfs 设置成默认 StorageClass：(option)

   ```
   kubectl patch storageclass nfs -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

#### 4.2.3 测试

1. 创建 pvc.yaml 文件

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-pvc
     annotations:
       volume.beta.kubernetes.io/storage-class: "nfs"
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 1Gi
   ```

2. 创建 pvc

   ```bash
   kubectl apply -f pvc.yaml
   ```

   此时可以看一下 pvc 状态，应该是 Bound，如果是 pending，看一下 nfs driver pod 的 log。K8S 1.20
   以后，会有这个报错：`unexpected error getting claim reference: selfLink was empty, can't make reference`，有两个办法，参考：<https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/issues/25>，在
   api-server 的 static pod 里添加启动参数：`--feature-gates=RemoveSelfLink=false`，或者更新 NFS
   驱动镜像：`gcr.io/k8s-staging-sig-storage/nfs-subdir-external-provisioner:v4.0.0`

3. 测试

   ```yaml
   kind: Pod
   apiVersion: v1
   metadata:
     name: test-pod
   spec:
     containers:
     - name: test-pod
       image: 99cloud/busybox:1.24
       command:
         - "/bin/sh"
       args:
         - "-c"
         - "touch /mnt/SUCCESS && exit 0 || exit 1"
       volumeMounts:
         - name: nfs-pvc
           mountPath: "/mnt"
     restartPolicy: "Never"
     volumes:
       - name: nfs-pvc
         persistentVolumeClaim:
           claimName: test-pvc
   ```

   > 其中 `99cloud/busybox:1.24` 是 `gcr.io/google_containers/busybox:1.24`，可以换成任意的镜像。

### 4.3 对接 Ceph RBD

[返回目录](#课程目录)

#### 4.3.1 基本环境

- OS: CentOS 7.9
- Kernel: 升级到 5.4
- 单网卡，系统盘 20G + 数据盘 20G

#### 4.3.2 部署 AIO O 版本

参考：<https://masantu.com/blog/2020-03-22/cephadm-install-Ceph-Octopus-on-CentOS7/>

```console
[root@lab-c2009-ceph-aio ~]# uname -r
5.4.182-1.el7.elrepo.x86_64
[root@lab-c2009-ceph-aio ~]# cat /etc/system-release
CentOS Linux release 7.9.2009 (Core)

[root@lab-c2009-ceph-aio ~]# yum install python3 -y
[root@lab-c2009-ceph-aio ~]# python3 -V
Python 3.6.8

[root@lab-c2009-ceph-aio ~]# ls /dev/vd*
/dev/vda  /dev/vda1  /dev/vda2  /dev/vdb
```

确认内核升级完毕，python3 部署完成（最好是 3.8，但实验环境中用 3.6 也行）

```bash
yum install -y yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

yum list docker-ce --showduplicates | sort -r

yum install -y docker-ce-20.10.16 docker-ce-cli-20.10.16 containerd.io docker-compose-plugin

systemctl enable docker --now

docker version
```

安装 Cephadm

```bash
curl -k --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm

# ./cephadm add-repo --release octopus
# 报错：repomd.xml: [Errno 14] curl#60 - "The certificate issuer‘s certificate has expired"
# 参考：https://blog.csdn.net/sulia1234567890/article/details/121956448
# 加入 vi /etc/yum.conf，增加：sslverify=0
vi /etc/yum.conf # sslverify=0
yum upgrade ca-certificates
./cephadm add-repo --release octopus
./cephadm install

which cephadm
# /usr/sbin/cephadm
```

引导新集群

```console
[root@lab-c2009-ceph-aio ~]# mkdir -p /etc/ceph
[root@lab-c2009-ceph-aio ~]# ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:cd:72:83 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.229/24 brd 192.168.122.255 scope global noprefixroute dynamic eth0
       valid_lft 2736sec preferred_lft 2736sec
    inet6 fe80::5390:413f:f60f:6034/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

如果要从开始就设置 osd pool 为 1

```bash
cat <<EOF > initial-ceph.conf
[global]
osd pool default size = 1
osd pool default min size = 1
EOF

cephadm bootstrap --config initial-ceph.conf --mon-ip 192.168.122.229

# Ceph Dashboard is now available at:

# 	     URL: https://lab-c2009-ceph-aio:8443/
# 	    User: admin
# 	Password: itgwv8r3xy

# You can access the Ceph CLI with:

# 	sudo /usr/sbin/cephadm shell --fsid 090a76ec-9bb4-11ec-91af-525400cd7283 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring
```

以上命令会操作：

- 在本地主机上为集群创建 mon 和 mgr 守护程序。
- 为 Ceph 集群生成一个新的 SSH 密钥，并将其添加到 root 用户的 `/root/.ssh/authorized_keys` 文件中。
- 将与新集群通信所需的最小配置文件写入 `/etc/ceph/ceph.conf`。
- 将 `client.admin` 管理（特权！）秘密密钥的副本写入 `/etc/ceph/ceph.client.admin.keyring`。
- 将公共密钥的副本写入 `/etc/ceph/ceph.pub`。

使用 Ceph CLI

```console
[root@lab-c2009-ceph-aio ~]# cephadm add-repo --release octopus
[root@lab-c2009-ceph-aio ~]# yum install -y ceph-common
[root@lab-c2009-ceph-aio ~]# ceph -s
  cluster:
    id:     090a76ec-9bb4-11ec-91af-525400cd7283
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 1

  services:
    mon: 1 daemons, quorum lab-c2009-ceph-aio (age 58m)
    mgr: lab-c2009-ceph-aio.nuvwkm(active, since 57m)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

[root@lab-c2009-ceph-aio ~]# ceph health
HEALTH_WARN OSD count 0 < osd_pool_default_size 1
```

接下来，**添加 OSD**，参考 <https://docs.ceph.com/en/latest/cephadm/services/osd/#cephadm-deploy-osds>

```console
[root@lab-c2009-ceph-aio ~]# ceph orch device ls
Hostname            Path      Type  Serial  Size   Health   Ident  Fault  Available
lab-c2009-ceph-aio  /dev/vdb  hdd           21.4G  Unknown  N/A    N/A    Yes

[root@lab-c2009-ceph-aio ~]# ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...
```

从命令行添加会卡住，然后失败（当副本数为 1 的时候，也显示如下，但可以成功，不必走界面添加）

```console
[root@lab-c2009-ceph-aio ~]# ceph orch daemon add osd lab-c2009-ceph-aio:/dev/vdb
Created no osd(s) on host lab-c2009-ceph-aio; already created?
```

从 ceph dashboard 可以添加 osd，先筛选，然后 add 即可。

```console
[root@lab-c2009-ceph-aio ~]# ceph -s
  cluster:
    id:     090a76ec-9bb4-11ec-91af-525400cd7283
    health: HEALTH_WARN
            1 pool(s) have no replicas configured

  services:
    mon: 1 daemons, quorum lab-c2009-ceph-aio (age 13m)
    mgr: lab-c2009-ceph-aio.fgkrqf(active, since 12m)
    osd: 1 osds: 1 up (since 2m), 1 in (since 2m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   1.0 GiB used, 19 GiB / 20 GiB avail
    pgs:     1 active+clean
```

#### 4.3.3 对接到 K8S

参考：<https://cloud.tencent.com/developer/article/1700941?from=article.detail.1927694>

CEPH 侧创建 Pool 和新用户

```console
[root@lab-c2009-ceph-aio ~]# ceph osd pool create kubernetes
pool 'kubernetes' created

[root@lab-c2009-ceph-aio ~]# ceph osd lspools
1 device_health_metrics
2 kubernetes

[root@lab-c2009-ceph-aio ~]# ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
[client.kubernetes]
	key = AQBaByJiJbL8KRAAW77WOh1hu6Pz9if9EPj9mA==

[root@lab-c2009-ceph-aio ~]# ceph auth get client.kubernetes
exported keyring for client.kubernetes
[client.kubernetes]
	key = AQBaByJiJbL8KRAAW77WOh1hu6Pz9if9EPj9mA==
	caps mgr = "profile rbd pool=kubernetes"
	caps mon = "profile rbd"
	caps osd = "profile rbd pool=kubernetes"
```

K8S 侧对接

参考：<https://github.com/ceph/ceph-csi>

| Ceph CSI Version | Container Orchestrator Name | Version Tested      |
| ---------------- | --------------------------- | ------------------- |
| v3.6.1           | Kubernetes                  | v1.21, v1.22, v1.23 |
| v3.6.0           | Kubernetes                  | v1.21, v1.22, v1.23 |
| v3.5.1           | Kubernetes                  | v1.21, v1.22, v1.23 |
| v3.5.0           | Kubernetes                  | v1.21, v1.22, v1.23 |

下载 `ceph-csi-3.6.1.tar.gz`，解压

```bash
yum install wget -y
wget --no-check-certificate https://github.com/ceph/ceph-csi/archive/refs/tags/v3.6.1.tar.gz
tar zxvf v3.6.1.tar.gz
```

```console
[root@lab-c2009-ceph-aio ~]# ceph mon dump
dumped monmap epoch 1
epoch 1
fsid 090a76ec-9bb4-11ec-91af-525400cd7283
last_changed 2022-03-04T12:12:10.455157+0000
created 2022-03-04T12:12:10.455157+0000
min_mon_release 15 (octopus)
0: [v2:192.168.122.229:3300/0,v1:192.168.122.229:6789/0] mon.lab-c2009-ceph-aio
```

- fsid：这个是 Ceph 的集群 ID。
- 监控节点信息。**目前 ceph-csi 只支持 v1 版本的协议？**所以监控节点那里我们只能用 v1 的那个 IP 和端口号（例如，192.168.122.229:6789）。

进入 ceph-csi 的 `deploy/rbd/kubernetes` 目录：

config-map

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "ceph-csi-config"
data:
  config.json: |-
    [
      {
        "clusterID": "090a76ec-9bb4-11ec-91af-525400cd7283",
        "monitors": [
          "192.168.122.229:6789"
        ]
      }
    ]
```

```bash
cd /root/ceph-csi-3.6.1/deploy/rbd/kubernetes

kubectl apply -f csi-config-map.yaml

cat <<EOF > csi-rbd-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
stringData:
  userID: kubernetes
  userKey: AQBaByJiJbL8KRAAW77WOh1hu6Pz9if9EPj9mA==
EOF

kubectl apply -f csi-rbd-secret.yaml
```

如果希望将所有 ceph-csi 相关的配置和服务启动在单独的命名空间中，参考 [deprecated.md](deprecated.md)

创建必须的 ServiceAccount 和 RBAC ClusterRole/ClusterRoleBinding 资源对象：

```bash
kubectl create -f csi-provisioner-rbac.yaml
kubectl create -f csi-nodeplugin-rbac.yaml
```

创建 PodSecurityPolicy：

```
kubectl create -f csi-provisioner-psp.yaml
kubectl create -f csi-nodeplugin-psp.yaml
```

删除 kms 配置

```diff
diff kubernetes-bak/csi-rbdplugin-provisioner.yaml kubernetes/csi-rbdplugin-provisioner.yaml
160,161c160,161
<             - name: ceph-csi-encryption-kms-config
<               mountPath: /etc/ceph-csi-encryption-kms-config/
---
>             # - name: ceph-csi-encryption-kms-config
>             #   mountPath: /etc/ceph-csi-encryption-kms-config/
230,232c230,232
<         - name: ceph-csi-encryption-kms-config
<           configMap:
<             name: ceph-csi-encryption-kms-config
---
>         # - name: ceph-csi-encryption-kms-config
>         #   configMap:
>         #     name: ceph-csi-encryption-kms-config

diff kubernetes-bak/csi-rbdplugin.yaml kubernetes/csi-rbdplugin.yaml
107,108c107,108
<             - name: ceph-csi-encryption-kms-config
<               mountPath: /etc/ceph-csi-encryption-kms-config/
---
>             # - name: ceph-csi-encryption-kms-config
>             #   mountPath: /etc/ceph-csi-encryption-kms-config/
188,190c188,190
<         - name: ceph-csi-encryption-kms-config
<           configMap:
<             name: ceph-csi-encryption-kms-config
---
>         # - name: ceph-csi-encryption-kms-config
>         #   configMap:
>         #     name: ceph-csi-encryption-kms-config
```

```console
[root@lab-c2009-k8s-aio-ceph kubernetes]# kubectl apply -f ../../../examples/ceph-conf.yaml

[root@lab-c2009-k8s-aio-ceph kubernetes]# kubectl get cm -A
NAMESPACE         NAME                                 DATA   AGE
ceph-csi          ceph-config                          2      27s
```

部署 csi-rbdplugin-provisioner 和 RBD CSI driver

```bash
kubectl create -f csi-rbdplugin-provisioner.yaml
kubectl create -f csi-rbdplugin.yaml
```

然后看一下是否所有的 pod 都起来了，master 节点上可能需要 untaint，镜像可能下载失败，deployment 可能要限制 replicas

```bash
kubectl scale deploy csi-rbdplugin-provisioner --replicas=1
kubectl taint nodes $(hostname) node-role.kubernetes.io/master:NoSchedule-
```

```bash
cat <<EOF > storageclass.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: 090a76ec-9bb4-11ec-91af-525400cd7283
   pool: kubernetes
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
   csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF

kubectl apply -f storageclass.yaml
```

这里的 clusterID 对应之前步骤中的 fsid。 imageFeatures 用来确定创建的 image 特征，如果不指定，就会使用 RBD 内核中的特征列表，但 Linux
不一定支持所有特征，所以这里需要限制一下。

进入 `ceph-csi` 项目的 `example/rbd` 目录，然后直接创建 PVC

```console
[root@lab-c2009-k8s-aio-ceph rbd]# kubectl apply -f pvc.yaml
persistentvolumeclaim/rbd-pvc created

[root@lab-c2009-k8s-aio-ceph rbd]# kubectl get pvc
NAMESPACE   NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
default     rbd-pvc   Pending                                      csi-rbd-sc     7s

[root@lab-c2009-k8s-aio-ceph rbd]# kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-pvc   Bound    pvc-d45b457f-9c67-4b82-b28b-329a1eb1747d   1Gi        RWO            csi-rbd-sc     4m30s

[root@lab-c2009-k8s-aio-ceph rbd]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
pvc-d45b457f-9c67-4b82-b28b-329a1eb1747d   1Gi        RWO            Delete           Bound    default/rbd-pvc   csi-rbd-sc              1s
```

在 ceph 侧，可以看到：

```console
[root@lab-c2009-ceph-aio ~]# rbd ls -p kubernetes
csi-vol-7b95cc49-9c1f-11ec-855a-ea06f4beb16f

[root@lab-c2009-ceph-aio ~]# rbd info csi-vol-7b95cc49-9c1f-11ec-855a-ea06f4beb16f -p kubernetes
rbd image 'csi-vol-7b95cc49-9c1f-11ec-855a-ea06f4beb16f':
	size 1 GiB in 256 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 37b4396277f1
	block_name_prefix: rbd_data.37b4396277f1
	format: 2
	features: layering
	op_features:
	flags:
	create_timestamp: Fri Mar  4 19:59:19 2022
	access_timestamp: Fri Mar  4 19:59:19 2022
	modify_timestamp: Fri Mar  4 19:59:19 2022
```

创建示例 Pod

```
[root@lab-c2009-k8s-aio-ceph rbd]# kubectl apply -f pod.yaml

[root@lab-c2009-k8s-aio-ceph rbd]# lsblk -l|grep rbd
rbd0        251:0    0    1G  0 disk /var/lib/kubelet/pods/9f47c56f-675c-40fc-9b6e-5f00676bf8d0/volumes/kubernetes.io~csi/pvc-d45b457f-9c67-4b82-b28b-329a1eb1747d/mount

[root@lab-c2009-k8s-aio-ceph rbd]# kubectl exec -it csi-rbd-demo-pod -- bash

root@csi-rbd-demo-pod:/# lsblk -l|grep rbd
rbd0 251:0    0    1G  0 disk /var/lib/www/html
```

遇到问题需要排查的话：`kubectl logs csi-rbdplugin-provisioner-5b4464666c-bzxf7 -c csi-provisioner`

### 4.4 跨命名空间的快照和备份

[返回目录](#课程目录)

PVC 是不可以跨 namespace 复制的。参考：<https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/>

_You can only clone a PVC when it exists in the same namespace as the destination PVC (source and
destination must be in the same namespace)._

可以借用 snapshot 来完成跨 namespace 复制
PVC，参考：<https://v1-20.docs.kubernetes.io/docs/concepts/storage/volume-snapshots/>

VolumeSnapshotContent 是 cluster 级别

1. create Snapshot（会自动生成 SnapshotContent 和 volumeHandle）
2. create new SnapshotContent（根据 volumeHandle）

   注意，创建时需要带两条注解，参考：<https://github.com/kubernetes-csi/external-snapshotter/issues/251>，否则删除新创建的 pvc
   时，会 hang 住

   - `snapshot.storage.kubernetes.io/deletion-secret-namespace: default`
   - `snapshot.storage.kubernetes.io/deletion-secret-name: csi-rbd-secret`

3. create Snapshot in other namespace（根据新的 SnapshotContent）
4. create pvc by snapshot（根据新的 snapshot）

底层基于快照，在 Ceph 上飞快（copy on write）

### 4.5 Local 和动态分配

[返回目录](#课程目录)

参考：<https://github.com/rancher/local-path-provisioner>

```console
[root@lab-k8s004 ~]# kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created

[root@lab-k8s004 ~]# kubectl get sc
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  50s

[root@lab-k8s004 ~]# kubectl patch storageclass local-path -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/local-path patched

[root@lab-k8s004 ~]# kubectl get sc
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  75s

[root@lab-k8s004 ~]# kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pvc/pvc.yaml
persistentvolumeclaim/local-path-pvc created

[root@lab-k8s004 ~]# kubectl create -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/examples/pod/pod.yaml
pod/volume-test created

[root@lab-k8s004 ~]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
pvc-013cdb6b-fe6e-4fa8-9e5b-8a5e4957a91b   128Mi      RWO            Delete           Bound    default/local-path-pvc   local-path              23s

[root@lab-k8s004 ~]# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
local-path-pvc   Bound    pvc-013cdb6b-fe6e-4fa8-9e5b-8a5e4957a91b   128Mi      RWO            local-path     40s
```

### 4.6 最佳实践

1. K8S 上的存储一般用商用 NAS（协议用 NFS）或者 CephRBD，相对成熟可靠。尽量不要用 CephFS（生产环境中存在丢数据风险）。
2. NFS 对接时，要考虑版本，v3 或 v4，Netapps 对接错版本会存在访问权限的问题
3. CephRBD 需要选用 N 以上版本，才支持 CSI 的快照等高级特性
4. PVC 复制可以借用 Snapshot，方便且快
5. Local 存在丢数据风险，需要在应用层做数据可靠性保重
6. DynamicLocal 在 K3S 环境生产级可用

## 5. 网络管理

[返回目录](#课程目录)

### 5.1 CNI 的一般概念

[返回目录](#课程目录)

Container Network Interface (CNI) 最早是由 CoreOS 发起的容器网络规范，是 Kubernetes 网络插件的基础。其基本思想为：Container
Runtime 在创建容器时，先创建好 network namespace，然后调用 CNI 插件为这个 netns 配置网络，其后再启动容器内的进程。现成为 CNCF 主推的网络模型。

CNI 插件包括两部分：CNI Plugin 和 IPAM Plugin

- CNI Plugin负责给容器配置网络，它包括两个基本的接口
  - 配置网络: AddNetwork(net NetworkConfig, rt RuntimeConf) (types.Result, error)
  - 清理网络: DelNetwork(net NetworkConfig, rt RuntimeConf) error
- IPAM Plugin 负责给容器分配IP地址，主要实现包括 host-local 和 dhcp。

Kubernetes Pod 中的其他容器都是 Pod 所属 pause 容器的网络，创建过程为：

1. kubelet 先创建 pause 容器生成 network namespace
2. 调用网络 CNI driver，根据配置调用具体的 cni 插件
3. cni 插件给 pause 容器配置网络
4. pod 中其他的容器都使用 pause 容器的网络

参考：

- <https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/>
- <https://kubernetes.io/docs/concepts/cluster-administration/networking/>
- <https://feisky.gitbooks.io/kubernetes/content/network/cni/cni.html>
- <https://zhuanlan.zhihu.com/p/110648535>

所有 CNI 插件均支持通过环境变量和标准输入传入参数：

```bash
echo '{"cniVersion": "0.3.1","name": "mynet","type": "macvlan","bridge": "cni0","isGateway": true,"ipMasq": true,"ipam": {"type": "host-local","subnet": "10.244.1.0/24","routes": [{ "dst": "0.0.0.0/0" }]}}' | sudo CNI_COMMAND=ADD CNI_NETNS=/var/run/netns/a CNI_PATH=./bin CNI_IFNAME=eth0 CNI_CONTAINERID=a CNI_VERSION=0.3.1 ./bin/bridge

echo '{"cniVersion": "0.3.1","type":"IGNORED", "name": "a","ipam": {"type": "host-local", "subnet":"10.1.2.3/24"}}' | sudo CNI_COMMAND=ADD CNI_NETNS=/var/run/netns/a CNI_PATH=./bin CNI_IFNAME=a CNI_CONTAINERID=a CNI_VERSION=0.3.1 ./bin/host-local
```

### 5.2 对接 Calico

[返回目录](#课程目录)

参考：<https://github.com/projectcalico/calico>

### 5.3 对接 OVN

[返回目录](#课程目录)

参考：<https://github.com/kubeovn/kube-ovn#kube-ovn-vs-calico>

### 5.4 多网络平面

[返回目录](#课程目录)

参考：<https://github.com/k8snetworkplumbingwg/multus-cni>

### 5.5 物理网卡加速

[返回目录](#课程目录)

参考：<https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin>

### 5.6 最佳实践

1. 推荐 Calico，需要追求性能可以用 BGP 模式
2. [如果是 IPIP/vxlan 模式，需要考虑 MTU](http://blog.wuwenxiang.net/k8s-app-debug)
3. 只有网元虚拟化，或者 ECS 场景，才推荐用 Multus 和 KubeOVN
4. DPDK 取决于 CNI 支持，比如 ovs-dpdk
5. 物理网卡加速取决于网卡硬件和驱动支持

## 6. 安全相关

[返回目录](#课程目录)

### 6.1 安全基线

[返回目录](#课程目录)

#### 6.1.1 Network Policy

参考：<https://kubernetes.io/docs/concepts/services-networking/network-policies/>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  # podSelector: {} 表示选择所有 pod 应用 NetworkPolicy
  podSelector: # 表示选择包含标签 role=db 的 pod 应用下面的 NetworkPolicy
    matchLabels:
      role: db
  policyTypes: # 表示 NetworkPolicy 包含 ingress 和 egress 流量规则
  - Ingress
  - Egress
  ingress: # ingress 规则白名单列表，每条规则允许同时匹配 from 和 ports  流，可以有条个规则。
  # 第 1 条白名单，包含 from + ports 的组合规则，允许来自172.17网段(172.17.1除外)、或标 签 project=myproject 的命名空间的所 有 pod 、或 default 命名空间下标签 role=frontend 的 pod 访问（限 tcp 6379 端口）
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  # 第二条白名单，只包含 from 规则，允许来自所有命名空间包含 environment=testing 标签的 pod 访问（不限端口）
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          environment: testing
  egress: # egress 规则白名单列表，同 ingress 规则一样，每条规则包含 to+ports，可以有多条规则。
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

#### 6.1.2 PodSecurityPolicy

参考：<https://kubernetes.io/docs/concepts/security/pod-security-policy/>，1.25 以后转
<https://kubernetes.io/docs/concepts/security/pod-security-admission/> (1.23 是 beta)

psp 的用法，包括 5 步骤

1. 创建 psp

   ```yaml
   apiVersion: policy/v1beta1
   kind: PodSecurityPolicy
   metadata:
   name: restrict-policy
   spec:
   privileged: false
   seLinux:
       rule: RunAsAny
   supplementalGroups:
       rule: RunAsAny
   runAsUser:
       rule: RunAsAny
   fsGroup:
       rule: RunAsAny
   volumes:
   - '*'
   ```

2. 创建 clusterrole，使用 psp

   `kubectl create clusterrole restrict-access-role --verb=use --resource=psp --resource-name=restrict-policy`

3. 创建 serviceaccount

   `kubectl create sa psp-denial-sa -n staging`

4. 绑定 clusterrole 到 serviceaccount

   `kubectl create clusterrolebinding dany-access-bind --clusterrole=restrict-access-role --serviceaccount=staging:psp-denial-sa`

5. 启用 PSP

   ```yaml
   vi /etc/kubernetes/manifests/kube-apiserver.yaml
   # 确保有以下内容：
   - --enable-admission-plugins=NodeRestriction,PodSecurityPolicy
   ```

### 6.2 权限限制

[返回目录](#课程目录)

#### 6.2.1 使用 AppArmor 限制容器对资源的访问

参考：<https://kubernetes.io/zh/docs/tutorials/security/apparmor/>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: podx
  annotations:
    container.apparmor.security.beta.kubernetes.io/podx: localhost/nginx-profile-3
spec:
  containers:
  - image: busybox
    imagePullPolicy: IfNotPresent
    name: podx
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
    resources: {}
  nodeName: node01
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

#### 6.2.2 限制 Pod 访问 K8S API 的权限

参考：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server>

serviceaccount 有个选项 automountServiceAccountToken, 这个选项决定是否自动挂载 secret 到 pod。 有这个选项，我们可以控制 pod 创建并绑定
serviceaccount 时，不自动挂载对应的 secret，这样 pod 就没有权限访问 apiserver，提高了业务 pod 的安全性。

可以在 serviceaccount 和 pod 的 spec 里设置，pod 的设置优先于 serviceaccount 里的设置。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend-sa
  namespace: qa
automountServiceAccountToken: false
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: qa
spec:
  serviceAccountName: backend-sa
  containers:
  - image: nginx:1.9
    imagePullPolicy: IfNotPresent
    name: backend
```

删除未使用的 serviceaccount

自动挂载的 service token 的 pod 可以做很多事，比如 get secret

```bash
k -n restricted get pod -o yaml | grep automountServiceAccountToken

curl https://kubernetes.default/api/v1/namespaces/restricted/secrets -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)" -k
```

### 6.3 K8S 安全扫描工具

[返回目录](#课程目录)

#### 6.3.1 修复 kube-bench 发现的安全问题

参考：<https://github.com/aquasecurity/kube-bench>

```console
$ kubectl apply -f https://github.com/aquasecurity/kube-bench/raw/main/job.yaml
job.batch/kube-bench created

$ kubectl get pods
NAME                      READY   STATUS              RESTARTS   AGE
kube-bench-j76s9   0/1     ContainerCreating   0          3s

# Wait for a few seconds for the job to complete
$ kubectl get pods
NAME                      READY   STATUS      RESTARTS   AGE
kube-bench-j76s9   0/1     Completed   0          11s

# The results are held in the pod's logs
kubectl logs kube-bench-j76s9
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
...
```

`kube-bench` 是一个 CIS 评估工具，扫描 kubernetes 集群存在的安全问题，基本上按照扫描结果的修复建议进行修复就可以了，系统会给出很具体的修复措施。

```bash
kube-bench run --targets=master | grep kube-controller -A 3

kube-bench run --targets=node | grep /var/lib/kubelet/config.yaml -B2
```

比较常见的是 kubeadm 创建的 kubernetes 服务器权限设置有问题，允许未经授权的访问。

- 使用 Node,RBAC 授权模式和 NodeRestriction 准入控制器

      ```yaml
      vi /etc/kubernetes/manifests/kube-apiserver.yaml
      # 确保以下内容
      - --authorization-mode=Node,RBAC
      - --enable-admission-plugins=NodeRestriction
      - --client-ca-file=/etc/kubernetes/pki/ca.crt
      - --enable-bootstrap-token-auth=true
      ```

- system:anonymous 的 ClusterRolebinding 角色绑定，匿名用户有集群管理员权限，需要取消

      ```bash
      kubectl delete clusterrolebinding system:anonymous
      ```

```bash
# 修复 kube-apiserver 安全问题
vi /etc/kubernetes/manifests/kube-apiserver
#修改：
--authorization-mode=Node,RBAC
#添加
--insecure-port=0
#删除
# --insecure-bind-address=0.0.0.0

#修复 kubelet 安全问题
vi /var/lib/kubelet/config.yaml
# 将authentication.anonymous.enabled 设置为 false
authentication:
  anonymous:
    enabled: false
# authorization.mode 设置为 Webhook
authorization:
  mode: Webhook

# 修复 etcd 安全问题
vi /etc/kubernetes/manifests/etcd.yaml
# 修改为true：
- --client-cert-auth=true

# 以上修复完成后 重新加载配置文件并重启kubelet

systemctl daemon-reload
systemctl restart kubelet
```

#### 6.3.2 筛选可能不符合最佳实践的 pod

- 启用了特权的 pod，主要是检查 pod 是否含 privileged: true

  `kubectl get po xxx -n production -o yaml| grep -i "privileged: true"`

- 检查有状态 pod

  `kubectl get pods XXXX -n production -o jsonpath={.spec.volumes} | jq`

#### 6.3.3 使用 sysdig 检查容器里里的异常进程

参考：<https://sysdig.com/use-cases/kubernetes-security/>

sysdig 的基本用法，记住两个帮助命令：

- `sysdig -h` 查看 sysdig 帮助
- `sysdig -l` 查看 sysdig 支持的元数据

另外 sysdig 支持指定 containerid 分析特定容器

```bash
# 查看容器id
docker ps |grep tomcat
sysdig -M 30 -p "*%evt.time,%user.uid,%proc.name" container.id=xxxx>opt/DFA/incidents/summary
```

### 6.4 容器镜像安全扫描

[返回目录](#课程目录)

#### 6.4.1 dockerfile 的不安全指令

dockerfile 里常见的不安全指令包括：

1. 使用 root 用户
2. 添加特定能力的 securityContext 安全上下文

```dockerfile
USER root

securityContext:
  {"Capabilities": {'add':{NET_BIND_SERVICE}, 'drop: []'}, 'privileged': TRUE}
```

#### 6.4.2 扫描镜像安全漏洞并处理使用有安全漏洞镜像的 pod

镜像扫描工具 trivy，参考：<https://github.com/aquasecurity/trivy>

```bash
# 获取镜像名
kubect get pod XXXX -n kamino -o yaml | grep image
# 扫描镜像
trivy imagename | grep (HIGH|CRITICAL)
# kubectl delete po xxx
```

### 6.5 审计日志

[返回目录](#课程目录)

参考：<https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/>

启动日志审计，包括两个步骤：

1. 编写日志审计策略文件

   ```yaml
   apiVersion: audit.k8s.io/v1
   kind: Policy
   omitStages:
   - "RequestReceived"
   rules:
   - level: RequestResponse
       resources:
       - group: ""
       resources: ["namespaces"]

   - level: Request
       resources:
       - group: ""
       resources: ["persistentvolumes"]
       namespaces: ["front-apps"]

   - level: Metadata
       resources:
       - group: ""
       resources: ["secrets", "configmaps"]

   - level: Metadata
       omitStages:
       - "RequestReceived"
   ```

2. 修改 `kube-apiserver.yaml` 配置文件，启用日志审计策略，日志策略配置文件位置、日志文件存储位置、循环周期。

   `vi /etc/kubernetes/manifests/kube-apiserver.yaml`

   ```yaml
   # 设置日志审计策略文件在 pod 里的 mount 位置
   - --audit-policy-file=/etc/kubernetes/logpolicy/sample-policy.yaml

   # 设置日志文件存储位置
   - --audit-log-path=/var/log/kubernetes/audit-logs.txt

   # 设置日志文件循环
   - --audit-log-maxage=10
   - --audit-log-maxbackup=2

   # mount 日志策略和日志文件的
   volumeMounts:
   - mountPath: /etc/kubernetes/logpolicy/sample-policy.yaml
       name: audit
       readOnly: true
   - mountPath: /var/log/kubernetes/audit-logs.txt
       name: audit-log
       readOnly: false
   volumes:
   - name: audit
       hostPath:
       path: /etc/kubernetes/logpolicy/sample-policy.yaml
       type: File
   - name: audit-log
       hostPath:
       path: /var/log/kubernetes/audit-logs.txt
       type: FileOrCreate
   ```

3. 重启 API Server 来检查 audit.log

   ```bash
   cd /etc/kubernetes/manifests/
   mv kube-apiserver.yaml ..
   watch crictl ps # wait for apiserver gone
   truncate -s 0 /etc/kubernetes/audit/logs/audit.log
   mv ../kube-apiserver.yaml .

   cat audit.log | tail | jq

   # shows Secret entries
   cat audit.log | grep '"resource":"secrets"' | wc -l

   # confirms Secret entries are only of level Metadata
   cat audit.log | grep '"resource":"secrets"' | grep -v '"level":"Metadata"' | wc -l

   # shows RequestResponse level entries
   cat audit.log | grep -v '"level":"RequestResponse"' | wc -l

   # shows RequestResponse level entries are only for system:nodes
   cat audit.log | grep '"level":"RequestResponse"' | grep -v "system:nodes" | wc -l
   ```

### 6.6 镜像准入检查

[返回目录](#课程目录)

ImagePolicyWebhook

参考：<https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook>

ImagePolicyWebhook 准入控制器的使用，分 4 个步骤

1. 修改控制器配置文件，将未找到有效后端时的默认拒绝改为默认不拒绝

   `vi /etc/kubernetes/epconfig/admission_configuration.json`

   ```json
   {

   "imagePolicy": {
       "kubeConfigFile": "/etc/kubernetes/epconfig/kubeconfig.yaml",
       "allowTTL": 50,
       "denyTTL": 50,
       "retryBackoff": 500,
       "defaultAllow": false
   }
   }
   ```

2. 修改控制器访问 webhook server 的 kubeconfig

   `vi /etc/kubernetes/epconfig/kubeconfig.yaml`

   修改如下内容

   ```yaml
   apiVersion: v1
   kind: Config
   clusters:
   - cluster:
       certificate-authority: /etc/kubernetes/epconfig/webhook.pem
       server: https://acme.local:8082/image_policy  # web hook server 的地址
   name: bouncer_webhook
   # 以下省略
   ```

3. 启用 ImagePolicyWebhook

   `vi /etc/kubernetes/manifests/kube-apiserver.yaml`

   ```yaml
   # 启用 ImagePolicyWebhook
   - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
   # 指定准入控制器配置文件
   - --admission-control-config-file=/etc/kubernetes/epconfig/admission_configuration.json
   # mount
       volumeMounts:
       - mountPath: /etc/kubernetes/epconfig
       name: epconfig
   # 映射 volumes
   volumes:
       - name: epconfig
       hostPath:
       path: /etc/kubernetes/epconfig
   ```

4. 测试是否生效

   ```bash
   systemctl daemon-reload
   systemctl restart kubelet
   kubectl apply -f /cks/img/web1.yaml
   ```

### 6.7 最佳实践

综上～

## 7. GPU 相关

[返回目录](#课程目录)

### 7.1 适合于计算的 GPU

#### 7.1.1 Nvidia

#### 7.1.2 AMD

#### 7.1.3 Huawei 昇腾 Ascend

#### 7.1.4 曙光海光

#### 7.1.5 阿里平头哥

#### 7.1.6 摩尔线程和 IMG ip core

翰博 帝先 景嘉微

#### 7.1.7 天数

### 7.2 GPU 驱动安装

### 7.3 GPU Operator

### 7.4 在 Pod 中使用 GPU

#### 7.4.1 如何令 Pod 启动时，缺省不使用 GPU？

##### 7.4.1.1 问题现象

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  containers:
  - name: cuda
    command: ["sleep", "infinity"]
    image: nvidia/cuda:12.2.0-base-ubuntu20.04
    resources:
      limits:
        cpu: 0.5
        memory: 500Mi
```

nvidia device plugin 默认是通过环境变量(NVIDIA_VISIBLE_DEVICES)来判断是否要将 gpu 暴露给容器的，而 nvidia 的大部分镜像都是基于 cuda
的，cuda 镜像默认会将这个变量的 value 设置为 all，参考
<https://hub.docker.com/layers/nvidia/cuda/12.2.0-base-ubuntu20.04/images/sha256-5a9257b8c6ac3f6c53ce559903558dce5ec1fa7a91ed8020dfb56ab7af1a8733?context=explore>，也可以从容器中验证：

```console
root@node1:~# kubectl  exec -it gpu-test -- env
...
NVARCH=x86_64
NVIDIA_REQUIRE_CUDA=cuda>=12.2 brand=tesla,driver>=450,driver<451 brand=tesla,driver>=470,driver<471 brand=unknown,driver>=470,driver<471 brand=nvidia,driver>=470,driver<471 brand=nvidiartx,driver>=470,driver<471 brand=geforce,driver>=470,driver<471 brand=geforcertx,driver>=470,driver<471 brand=quadro,driver>=470,driver<471 brand=quadrortx,driver>=470,driver<471 brand=titan,driver>=470,driver<471 brand=titanrtx,driver>=470,driver<471 brand=tesla,driver>=525,driver<526 brand=unknown,driver>=525,driver<526 brand=nvidia,driver>=525,driver<526 brand=nvidiartx,driver>=525,driver<526 brand=geforce,driver>=525,driver<526 brand=geforcertx,driver>=525,driver<526 brand=quadro,driver>=525,driver<526 brand=quadrortx,driver>=525,driver<526 brand=titan,driver>=525,driver<526 brand=titanrtx,driver>=525,driver<526
NV_CUDA_CUDART_VERSION=12.2.53-1
NV_CUDA_COMPAT_PACKAGE=cuda-compat-12-2
CUDA_VERSION=12.2.0
...
NVIDIA_VISIBLE_DEVICES=all
...
NVIDIA_DRIVER_CAPABILITIES=compute,utility
```

```console
root@node1:~# kubectl exec -it gpu-test -- nvidia-smi
Tue Oct 10 04:22:36 2023
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.12             Driver Version: 535.104.12   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  Tesla T4                       On  | 00000000:09:00.0 Off |                  Off |
| N/A   28C    P8              15W /  70W |      2MiB / 16384MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+

+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|  No running processes found                                                           |
+---------------------------------------------------------------------------------------+

root@node1:~# kubectl exec -it gpu-test -- df -h
Filesystem      Size  Used Avail Use% Mounted on
...
udev            2.0G     0  2.0G   0% /run/nvidia-container-devices/GPU-5fa68d66-4cb3-891a-bf8e-45519ce2b3ee
...
```

所以就导致了，如果容器没有申请 gpu 则会使用默认的环境变量，从而将所有的 gpu 暴露给容器(这部分工作应该是由 hook 完成的)；如果容器申请了 gpu 则在创建容器时 device
plugin 会去覆盖这个环境变量，变成所申请的 gpu 序号，申请 gpu 的容器环境变量如下所示：

```console
root@ecs-gpu:~# kubectl  exec -it gpu-test -- env
...
NVIDIA_VISIBLE_DEVICES=GPU-5fa68d66-4cb3-891a-bf8e-45519ce2b3ee
...
```

#### 7.4.1.2 解决思路-1：将 gpu 发现方式改为 volume-mount

社区给出了一个临时的[解决方案](https://docs.google.com/document/d/1uXVF-NWZQXgP1MLb87_kMkQvidpnkNWicdpO2l9g-fw/edit)，将
gpu 发现方式改为 volume-mount，不过也在文档中提到了，未来的最终解决方案会是 CDI。下面以 gpu-operator 部署的集群为例来演示一下流程，仅部署 nvidia device
plugin 的集群也可以通过修改 plugin 的启动参数 --device-list-strategy=<envvar | volume-mounts> 来进行修改。

先修改 gpu-operator 的 cr cluster-policy

```yaml
# kubectl edit clusterpolicy cluster-policy
  devicePlugin:
    env:
    - name: DEVICE_LIST_STRATEGY
      value: volume-mounts
```

再修改 toolkit 配置，配置文件的位置分为两种情况：

1. 如果 toolkit 是使用 gpu-operator 安装的，则配置文件的位置在
   /usr/local/nvidia/toolkit/.config/nvidia-container-runtime/config.toml
2. 如果 toolkit 为手动安装的，则配置文件位置在/ etc/nvidia-container-runtime/config.toml

修改配置文件，改 `accept-nvidia-visible-devices-*` 相关变量：

```ini
disable-require = false
#swarm-resource = "DOCKER_RESOURCE_GPU"
# 
accept-nvidia-visible-devices-envvar-when-unprivileged = false
accept-nvidia-visible-devices-as-volume-mounts = true

[nvidia-container-cli]
#root = "/run/nvidia/driver"
#path = "/usr/bin/nvidia-container-cli"
...
```

此时，如果不申请 GPU，Pod 中就不会主动挂载：

```console
root@node1:~# kubectl  exec -it gpu-test -- nvidia-smi
error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "fc8e28159bca207f9023881f20df1fdad46edc3c49c69576e616b12084de239c": OCI runtime exec failed: exec failed: unable to start container process: exec: "nvidia-smi": executable file not found in $PATH: unknown
```

#### 7.4.1.3 解决思路-2：使用 limitrange

从另一个维度来思考，不限制的话，Pod 能看到所有的 CPU / 内存，那么看到所有的 GPU 也合理。

需要限制的话，就显示限制。如果要默认限制，那就用 LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: gpu-resource-constraint
  namespace: default
spec:
  limits:
  - default:
      nvidia.com/gpu: "0"
    defaultRequest:
      nvidia.com/gpu: "0"
    type: Container
```

创建 pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  containers:
  - name: cuda
    command: ["sleep", "infinity"]
    image: nvidia/cuda:12.2.0-base-ubuntu20.04
```

创建出的 pod 会自动添加 limit

```yaml
# kubectl get po gpu-test -0 yaml i grep -i request -A 10 -B 10
spec:
  containers:
  - command:
    - sleep
    - infinity
    image: nvidia/cuda:12.2.0-base-ubuntu20.04
    imagePullPolicy: IfNotPresent
    name: cuda
    resources:
      limits:
        nvidia.com/gpu: "0"
      requests:
        nvidia.com/gpu: "0"
```

### 7.5 切分 GPU

#### 7.5.1 What & Why

除了大模型，通常我们不需要完整的一块显卡算力来做推理。这时我们就需要切分 GPU 算力给到多个 Pod 同时使用。

切分 GPU 以充分利用算力，这并不符合 Nvidia 希望多卖显卡的圈钱逻辑，所以 Nvidia 并不热衷于推 GPU 切分方案。

在虚拟化平台上，Nvidia 有 vGPU 的解决方案，但也需要额外卖 License（通过一个 vGPU license server
服务来授权）。真妥妥的雁过拔毛、为富不仁。这种“不仁”也必须是当然的了，Nvidia GPU 那么贵，而且“正经客户”还不能买游戏卡做训练，逼良为那啥。

通过容器来切分 GPU 的用户故事大致如下：

1. 多个容器能同时共享使用单张 GPU 的算力
2. 共享单张 GPU 资源的多个容器之间能进行 GPU 算力资源隔离，互相不影响
3. 分配不同 GPU 资源的容器都可以在自己分配资源限制内完成不超过其限制的大模型任务

#### 7.5.2 GPU 切分的原理

##### 7.5.2.1 GPU 切分的基本理念

参考：<https://github.com/zw0610/zw0610.github.io/blob/master/notes-cn/gpu-sharing-1.md>

为什么需要切分 GPU？

1. 类似于线程分时复用 CPU Core，GPU 共享也是靠任务分时复用来完成。任务的 GPU context 相当于线程的栈上下文。GPU context 切换和 CPU 线程切换一样，有损耗。
2. 有人说能不能你程序就用完 GPU 好了，干嘛要剩呢？跑完任务就释放出来。这就是扯淡了。推理请求首先是按需的，其次和模型有关。这和让线程吃光 CPU 一样荒谬。

被共享的 GPU 算力资源包括哪些？

- 通常从两个相对独立的维度来描述：GPU 显存和 GPU 算力（对应于内存和 CPU）。一个任务，其对 GPU 显存的占用和其对 GPU 算力的使用并不一定呈现线性关系。显然是存在占用 GPU
  显存很多但算力很少的应用，亦或是只占用少量显存但频繁计算的应用。
- 在深度学习（DL）领域，我们一般可以认为真实显存的占用和 GPU 算力的使用是正相关的。为什么要说“真实显存占用”呢？因为存在 TensorFlow
  这样比较“自以为是”的应用，觉得自己管理显存更在行，于是在程序运行之初便申请全部显存后自己管理。显然其占用的（全部）显存并不全部被用作训练或推理。我们可以改变这种行为。

GPU 使用场景大致分为 3 类：推理，训练，其它

1. **推理服务** 适合用 GPU 共享：推理时，GPU 显存和算力使用都较小。首先是 GPU
   显存的占用远小于训练（训练时，大量激活函数由于其非光滑的特性，必须保留中间变量以便日后求导，而这些变量在推理服务中即可舍去）。其次推理也没有反向传播的需求，对于计算量的需求大幅下降。加之一般推理请求的特性就是
   request-after-request，往往不像训练时那样组成 batch，导致单次推理操作对 GPU 算力的使用也很低。
2. **训练服务** 不适合 GPU 共享，多机多卡分布式训练还差不多。batch 类，训练完进程就退出了，不需要服务进程一直在线。
3. 其他 GPU 应用，如果依然遵循显存与算力正相关的规律的话，可以依照“小显存，小算力”和“大显存，大算力”来划分。Nvidia MPS 提供的案例为 N-Body 模拟。

虽然推理服务适合 GPU 切片，但不是唯一的，还有其它解决思路，比如：Nvidia Triton Inference Server，实现同一个 context 下利用多 stream
的方式实现多个模型同时实现前向推理功能的。结合更灵活且智能的 model scheduling，有可能效果更好（需要验证）。

##### 7.5.2.2 Slurm 切分方案

默认的调度器无法处理 GRES，一般来说需要安装对应的 GRES Plugin。如果在部署 Slurm 之前就已经在集群上装好了 NVML（一般随着驱动就会安装），Slurm 会利用
select/cons_tres plugin 自动进行检测并将（Nvidia）GPU 资源登记。 而在调度 GPU 任务时，Slurm 依旧使用 `CUDA_VISIBLE_DEVICES` 来对多
GPU 的节点进行以卡为单位的 GPU 资源隔离。每个任务在节点上被拉起时，Slurm 都会在 Epilog 和 Prolog 来配置 `CUDA_VISIBLE_DEVICES` 使得任务可以使用的
GPU 受到限制。这些配置在 Slurm 的 master 节点上完成。

Slurm 通过 GRES（Generic Resources，非常见资源） 支持 Nvidia CUDA Multi-Process Service
[MPS](https://docs.nvidia.com/deploy/pdf/CUDA_Multi_Process_Service_Overview.pdf)。MPS 允许多个小任务同时运行在一块
GPU 上，相比较 GPU context 切换，损耗（overhead）较低 。MPS 通过设置 `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` 来限制每个任务的算力，取值在
(0,100] 之间（即百分比）。Slurm 对 GPU 共享的调度已经做到相当原生。而如果有进一步的需求，也可以通过
[Slurm Plugin API](https://slurm.schedmd.com/gres_plugins.html) 自己来实现一个。

##### 7.5.2.3 K8S 切分方案

K8S 通过 Device Plugin 来实现 GPU 的非共享调度。想要做到共享切片，必须：

1. Node 上必须可以将一块 GPU 多次绑定到不同的容器
2. Scheduler 必须处理非 nvidia.com/gpu 这一类的资源

与 Slurm + MPS 按照算力分割略有不同的是，K8S 切分方案（比如阿里）的方案以显存为分割尺度，并且默认地认为 GPU 算力的需求和显存的需求是成正比的。这有一定合理性。

#### 7.5.3 切分方案：Nvidia MIG

Nvidia 原则两种 GPU 共享的方案：

1. GRID 方案可以做到完全资源（算力和显存）隔离，但是必须依托虚拟机，无法在容器上直接挂在分割后的 sub-GPU。
2. MPS 方案可以对接容器（Volta 之后的卡），对算力也能做限制，且无需担心 context switch 带来的 overhead，因为 MPS daemon 将各位 clients 发来的
   context 通过 daemon 自身的 context 透传到 GPU。但是受限于硬件，对于显存、IO 带宽，MPS
   方案无法做到限制，往往需要额外的组件来处理显存相关的操作。除此之外，之前提到的 MPS context 的错误无法被隔离。这样一来，一个 client 发生了错误，或者说 daemon
   context 发生了错误，会影响到其他的 client CUDA 程序的运行。

Ampere 带来的 A100 所具备的
[Multi-Instance GPU (MIG)](https://devblogs.nvidia.com/nvidia-ampere-architecture-in-depth/)：

1. MIG 从硬件的层面不仅对 SM（Stream-Multiprocessor，流处理器）进行了分割，还对整个内存系统进行了分割，包括了 Bus、DRAM、L2、Memory Controller
   等。这样一来，不同 GPU instance 上的用户程序可以享受不受打扰的显存、带宽等资源。
2. 与此同时，blog 中也提到，新的 QoS 使得单个 instance 上的错误并不会影响到其他 instance 上的 CUDA 程序。这对生产实践助益颇多。
3. 当前 MIG 的分割方案还是比较固定的，**一张卡最多可以分成 7 份**。当然除了分成 7 份，还有其他力度的分割方案。鉴于 A100 的庞大算力，即便分成 7
   份做一般模型（不是很庞大的模型）的推理服务其实并不划算，可能更多的使用场景还是多用户同时调试模型或直接做**小模型的训练**。

总体来看，利用 Ampere 这代架构在硬件上的隔离，无论是公有容器云还是私有容器云都可以很快地部署带 GPU 共享的 Kubernetes 集群，并且做到完整的算力、显存隔离而不需要额外的一些组件。

#### 7.5.4 切分方案：阿里方案

参考：<https://github.com/AliyunContainerService/GPUshare-scheduler-extender>

实现原理：

1. 拉起 container 过程由 kubelet 完成，节点上的 device-plugin 只提供节点上加速器（即 GPU）的状态。阿里提供了新的 device-plugin。它以共享 GPU
   当作 Extended Resource 注册，并且统计节点上所有 GPU 显存的总和。当 kubelet 调用 device-plugin 的 allocate API
   时，device-plugin 会先通过 k8s API 获取所有被调度到该节点但尚未被处理的 GPU Sharing Pod，而后选择老的 Pod（等待时间最久），为其在环境变量中配置从
   annotation 获取的 Device ID 给 `CUDA_VISIBLE_DEVICES` 以便实现 GPU 与 GPU 之间的隔离，最后标记为 assigned。
2. 而在调度器一层，阿里的方案使用的工具是
   [Extended Scheduler](https://github.com/AliyunContainerService/gpushare-scheduler-extender)。共享的
   GPU 被注册为一种新的资源 Extended Resource，而一旦 Pod 被调度到。现在需要做的就是让 k8s 的调度器可以正确处理相关的 Pod。由于默认调度器仅能判断一个节点上的
   GPU 显存是否足够容纳当前的 Pod，因此 GPU Share Scheduler Extender 会帮助其在做一次过滤，将那些单个 GPU
   不足以容纳下显存申请的节点过滤掉。在选择好节点之后，binding 的过程也交由 Scheduler Extender 完成。当中主要的工作是在选择好的节点中，以避免资源浪费的形式选择合适的
   GPU、将选择好的 GPU ID 写入 Pod 的 annotation、将 Pod 与 Node 绑定。**这是 cgpu 的部分开源项目，能对 gpu
   进行显存划分，多容器调度，但是没有实现显存隔离**。

#### 7.5.5 切分方案：腾讯方案

TKEStack 的 GaiaGPU：<https://github.com/tkestack/gpu-manager>，**cuda 12 之后就不支持了**。

##### 7.5.5.1 显存限制：API 劫持

CUDA 任务使用显存的基本逻辑（以申请一块 NxN 的矩阵为例）：

1. 可以预先计算一下 NxN 的 float32 矩阵需要多少空间：uint32 bytes = N * N * sizeof(float);
2. 查询整块 GPU 的显存总量或当前剩余的 GPU 显存总量
3. （如果剩下的显存量够多）利用 CUDA Runtime API 或者 CUDA Driver API 向 GPU 申请 bytes 那么多的显存
4. 确认申请显存的操作成功

当多个任务同时在一块 GPU 上进行显存申请的时候，我们希望做到和上述步骤不同的地方是：**2. 查询显存总量/剩余显存总量时，返回的不希望是整块 GPU
的显存相关信息，而是限制之后的相关信息**。举个例子，一块 GPU 一共有 12Gi 显存，我们分配给任务 A 5Gi 显存，给任务 B 3Gi 显存。当任务 A 中的进程调用
cuDeviceTotalMem 时，应该返回 5Gi。假设任务 A 中的进程已经使用了 1.5 Gi 的显存，那么当调用 cuMemGetInfo 时，应该返回 3.5 Gi。

而如果有的用户程序不经过步骤 2，直接执行步骤 3 呢？显然，我们需要根据上述逻辑对 **如果剩下的显存量够多** 作出判断。

那么要修改 CUDA 函数的实现，一般可以走 `LD_PRELOAD` 或 `LD_LIBRARY_PATH` 两种方式。前者的模式适用范围有限，无法进一步劫持由 Runtime API 对
Driver API 的调用。因此 TKEStack 采取修改 libcuda 并通过 `LD_LIBRARY_PATH`
加载。具体的代码可以参看：[vcuda-controller](https://github.com/tkestack/vcuda-controller)。

##### 7.5.5.2 算力限制：负反馈调节

每次发起 CUDA kernel 的时候，都检查一下当前任务对 GPU 的使用率，并对本次 CUDA kernel 会增加的 GPU 使用率作出估计。如果预计本次 CUDA kernel 会使得
GPU 使用率超标，则延缓 kernel 的运行，直到当前的 GPU 使用率下降至允许本次 CUDA kernel 的运行之后。然后这样做，无法避免多个任务的 context 切换带来的
overhead。

劫持的工程实现是做在 vcuda-controller 上的。

#### 7.5.6 切分方案：VirtAI 的社区版方案

VirtAI 社区版其实只是供社区使用的一个安装包，不含任何代码。通过分析安装包内的信息大致作出推测：

1. 采取了劫持 CUDA API 的方式，但 VirtAI 不仅劫持了 Driver API，还同时劫持了 Runtime API
2. VirtAI 的将所有的 API 用 rpc 进行了包装，使之可以运行在调用 API 的进程之外，这样也就解释了为什么 GPU 节点上会有 orind 这个 daemon 进程存在
3. VirtAI 号称实现了类似 MPS 的多 CUDA 进程无 context 切换，这是怎么操作的尚还不知晓

#### 7.5.7 切分方案：第四范式方案

##### 7.5.7.1 原理

第四范式开源的 vgpu 上层实现，底层核心逻辑是 libvgpu.so 提供的，没有开源，可以实现对物理 gpu 的切分，实现了显存隔离。

- <https://github.com/4paradigm/k8s-vgpu-scheduler>：OpenAIOS vGPU scheduler for Kubernetes is
  originated from the OpenAIOS project to virtualize GPU device memory.
- <https://github.com/4paradigm/k8s-device-plugin>：vGPU device plugin 基于 NVIDIA
  官方插件（NVIDIA/k8s-device-plugin），在保留官方功能的基础上，[实现了对物理GPU进行切分，并对显存和计算单元进行限制，从而模拟出多张小的 vGPU 卡。在 k8s
  集群中，基于这些切分后的 vGPU 进行调度，使不同的容器可以安全的共享同一张物理 GPU，提高 GPU
  的利用率。此外，插件还可以对显存做虚拟化处理（使用到的显存可以超过物理上的显存），运行一些超大显存需求的任务，或提高共享的任务数](https://github.com/4paradigm/k8s-device-plugin/blob/master/README_cn.md#关于)。

##### 7.5.7.2 实验

参考：<https://github.com/4paradigm/k8s-vgpu-scheduler#quick-start>，环境中装好 OS，GPU 驱动， K8S with nvidia
containerd。

```bash
kubectl label nodes {nodeid} gpu=on

helm repo add vgpu-charts https://4paradigm.github.io/k8s-vgpu-scheduler
helm install vgpu vgpu-charts/vgpu --set scheduler.kubeScheduler.imageTag=v1.23.17 -n kube-system

kubectl apply -f test-gpu-2.yaml
```

```yaml
# cat test-gpu-2.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod-2
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu:18.04
      command: ["bash", "-c", "sleep 86400"]
      resources:
        limits:
          nvidia.com/gpu: 2 # requesting 2 vGPUs
          nvidia.com/gpumem: 3000 # Each vGPU contains 3000m device memory （Optional,Integer）
          nvidia.com/gpucores: 30 # Each vGPU uses 30% of the entire GPU （Optional,Integer)
```

进 pod 看：

```console
root@cloud-NF5468M6:~# kubectl exec -it gpu-pod-2 -- bash

root@gpu-pod-2:/# nvidia-smi 
[4pdvGPU Msg(17:139640230053696:libvgpu.c:870)]: Initializing.....
Thu Oct 26 10:19:58 2023       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.113.01             Driver Version: 535.113.01   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA A40                     Off | 00000000:9D:00.0 Off |                    0 |
|  0%   39C    P0              87W / 300W |      0MiB /  3000MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
|   1  NVIDIA A40                     Off | 00000000:A0:00.0 Off |                    0 |
|  0%   77C    P0             299W / 300W |      0MiB /  3000MiB |     96%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
+---------------------------------------------------------------------------------------+
[4pdvGPU Msg(17:139640230053696:multiprocess_memory_limit.c:475)]: Calling exit handler 17
```

符合预期。

### 7.6 多机多卡 GPU 方案

#### 7.6.1 Ray

#### 7.6.2 Kuberay
