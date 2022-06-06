# Kubernetes 最佳实践

## 注意 ⚠️

- *斜体表示引用*
- **未经允许，禁止转载**

## 学习本课程前，你应该具备如下知识：

- Linux 系统的基本配置和命令
- Docker 命令
- K8S 基本配置和 kubectl 命令
- 对存储和网络有基本了解更佳

## 课程目录

## 1. 操作系统

[返回目录](#课程目录)

### 1.1 CentOS 7

[返回目录](#课程目录)

参考 <https://www.centos.org/download/>，EOF（End-of-life）时间 2024/06/30。CentOS 7 目前仍是生产环境中使用最广泛的容器操作系统。

CentOS 最麻烦的地方是配套的 yum/rpm 包过于久远。

#### 1.1.1 内核升级

首先是 Kernel，3.10 的 Kernel 会遇到的问题

1. KVM 无法启用硬件加速（kvm_intel 模块无法加载），告警：`QEMU: Checking if device /dev/kvm exists : FAIL (Check that the 'kvm-intel' or 'kvm-amd' modules are loaded & the BIOS has enabled virtualization)`
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
# [global]
# index-url = https://pypi.tuna.tsinghua.edu.cn/simple
python3 -m pip install --upgrade pip
```

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

| LTS(Long Term Support) | Release | EOL(End Of Life) | HWE(Hardware Enablement) |
| - | - | - | - |
| 22.04 Jammy Jellyfish | 2022/04/21 | 2027/04 |
| 20.04 Focal Fossa | 2020/04 | 2025/04 | 2030/04 |
| 18.04 Bionic Beaver | 2018/04/26 | 2023/04 | 2028/04 |

Ubuntu 的策略比较简单，保持循 LTS 升级即可。

### 1.3 CentOS 8 Stream

[返回目录](#课程目录)

[Comparing CentOS Stream and CentOS Linux](https://www.centos.org/cl-vs-cs/):

*End of Life*

- CentOS Linux 8 EOL: 2021-12-31
- CentOS Stream 8 EOL: 2024-05-31
- CentOS Stream 9 EOL: estimated 2027, dependent on RHEL9 end of “Full Support Phase”

*Upstream vs downstream*

- CentOS Linux 是 Red Hat Enterprise Linux (RHEL) 的 rebuild，是 RHEL 的 Downstream。CentOS Linux 的版本号是 RHEL 的发布日期，比如 CentOS 8.2105 就是 RHEL 8.3（发布于 2021/05） 的 rebuild
- CentOS Stream 是 RHEL 的 upstream, RHEL 的 public development 分支。简单说，就是：**不稳定**。

综上，**CentOS 8 已经停止支持，CentOS 8 Stream 不稳定，两者都不要用于生产了**。

参考：<https://www.ispsystem.com/news/what-would-be-the-alternative-to-centos>

![](/image/CentOS-alternative-survey.png)

注：Oracle Linux 以 RHEL 为起始，与 RHEL 完全兼容，移除了 Red Hat 商标，加入了 Linux 错误修正。其可取之处是 7*24 服务…… 像另一个 RHEL。

### 1.4 Rocky

[返回目录](#课程目录)

参考：<https://www.logicweb.com/what-is-rocky-linux/>

2020 年 12 月（IBM 收购 Red Hat 之后），收购 CentOS 的 Red Hat 宣布，CentOS Linux 8 将于 2021 年底结束支持，比早些时候承诺的 10 年计划要短得多。CentOS Stream 是 development 版本，将按计划在2024年结束。从此以后 CentOS 将位于 RHEL 的上游，而不是下游，CentOS 用户实际上将是 RHEL 的测试人员。

这引发了广大 CentOS 用户的极大不满，CentOS 创始人 Gregory Kurtzer 随即宣称领导创建新的 “CentOS” 发行版的工作。Rocky 这个名字是为了纪念已故 CentOS 联合创始人 Rocky McGaugh。Kurtzer 曾在加州大学伯克利分校从事高性能计算工作很长时间。鉴于CentOS 在 CERN 等机构的粒子物理学中得到了广泛应用，这可能会是Rocky Linux 的主要关注点之一。

参考：<https://rockylinux.org/>

*Enterprise Linux, the community way.
Rocky Linux is an open-source enterprise operating system designed to be 100% bug-for-bug compatible with Red Hat Enterprise Linux®. It is under intensive development by the community.*

参考：<https://rockylinux.org/download/>，Planned EOL: 2029/05/31

综上，**Rocky 是另起炉灶的 CentOS Linux**。当下进展不错。参考 <https://github.com/rocky-linux/rocky>，2021/06，Rocky Linux 已经生产环境 ready。建议在生产环境中使用（CentOS 7 结束支持前半年，也就是 2023 年应该转向 Rocky Linux）。

### 1.5 AlmaLinux

AlmaLinux 由 CloudLinux 的开发人员构建和维护，CloudLinux 是一家提供服务器托管和 Linux 软件的公司。这是一家在 RHEL 分支方面经验丰富的公司，十多年来一直构建和维护其内部发行版 CloudLinux OS，它本身就是一个分支。

你**应该选 Alma 还是 Rocky**？它俩都致力于提供社区版的 RHEL，这有点难回答：

从所有权来说（AlmaLinux 开放，Rocky 独裁）：

- Rocky Linux 由 Kurtzer 创立的 Rocky Enterprise Software Foundation (RESF) 控制和管理。同时，他还是为 Rocky Linux 提供保护伞的 Public Benefit Corporation (PBC) 所有者。所以，Kurtzer 基本上拥有 Rocky。是的，RESF 有一个管理委员会，但无论你怎么看，Kurtzer 都是公司持有人，并且可能是 Rocky Linux 的决策者。“独裁”可能是好事，也可能是坏事。理论上讲，他可能像卖 CentOS 一样，再卖一次 Rocky。我们只需要相信他，他会阻止之前发生的事情再次发生。
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

参考：<https://developer.huaweicloud.com/ict/en/site-euleros/euleros>，EulerOS 是华为推出的企业 Linux 系统（诞生于华为科学实验室），Openeuler 是 EulerOS 的社区版本，2019 年开源，现贡献给开放原子基金会，代码托管在 <https://gitee.com/openeuler>。

OpenEuler 兼容 CentOS（但是它并不是蹭 CentOS 8 结束支持热点才开源的，开源在那之前），相较 CentOS 对核内关键功能如进程调度、内存管理、IO 读写进行了深度优化；同时在核外构筑了容器 iSula、虚拟化 StraitVirt、机密计算 SecGear、毕昇 JDK（Huawei JDK 的开源版本）。

综上，**OpenEuler 的优势是国产、信创，能较好地适配国产 ARM 服务器（特别是华为鲲鹏、泰山等）。缺点是兼容性和稳定性验证稍显不足**。

### 1.7 龙蜥

[返回目录](#课程目录)

龙蜥由阿里巴巴在 2021/10/20 孵化出来，诞生背景是 CentOS 8 结束支持（CentOS 8 结束支持这事，堪称“一鲸落，万物生”，可惜 Alma 和 Rocky Linux 起得太快）。

参考：<https://openanolis.cn/anolisos>，100% 兼容 CentOS 8 软件生态。

参考：<https://www.zhihu.com/question/502615238/answer/2408765289>，一言难尽。

### 1.8 麒麟

这个就说来话长了，我也查了半天，诸君姑且听之。

- **中标麒麟**：2010/12/16，中标 Linux 和军方的“银河麒麟”在上海宣布合并，开发方中标软件有限公司和国防科技大学同日缔结了战略合作协议，双方今后将共同以“中标麒麟”的新品牌出现。
- **银河麒麟** Kylin Operating System 是天津麒麟信息技术有限公司旗下的国产 Linux 操作系统，源自国防科大“麒麟”、“银河麒麟”操作系统，支持主流 X86 架构 CPU 以及国产飞腾 CPU 平台。国防科大继续了老“麒麟”的开发。
- **优麒麟** UbuntuKylin 是 Ubuntu 社区中面向中文用户的 Ubuntu 衍生版本，中文名称优麒麟，与麒麟系统没有关系。优麒麟有两个身份，首先它是 Ubuntu 的一个官方 Flavor 版本。其次，它背后也有国防科大和天津麒麟的支持，可以看做银河麒麟的社区版。优麒麟最初的目标是像 Ubuntu 一样占领中国市场，可是很多人直接选择了 Ubuntu，一些选择了更接地气的 Deepin，所以优麒麟并不算非常成功。
- **湖南麒麟**，湖南麒麟信息工程技术有限公司（简称湖南麒麟）是 2007 年成立的一家民营企业，公司成立之初依托国防科技大学计算机学院，长期致力于信息安全的研发，在集中管控和机要密码等领域有一定的影响力。2014 年天津麒麟成立时，国防科大将“麒麟”、“银河麒麟”等无形资产注入了天津麒麟，湖南麒麟原有的操作系统研发团队整体转入天津麒麟。现在的湖南麒麟，只是一家单纯的民营企业了，可以视为一个新的系统。

综上：

- 中标麒麟像 CentOS，原来兼容性很好，2021 版本改了很多 rpm 依赖
- 银河麒麟像 Ubuntu
- 优麒麟基本不会在生产环境中用到
- 2022/05/16，科创板上市委公告，湖南麒麟信安科技股份有限公司首发获通过

emmmmm，得承认，麒麟在信创方面有一定优势。

### 1.9 最佳实践

1. 如选用 CentOS 系列，**2023/06/30 之前 CentOS 7 没问题**（需要升级 linux kernel 和工具库，比如 python 3.8，git v2）；但之后要选择 AlmaLinux 或者 Rocky Linux，二选一，**2024/06/30 之前要完成生产环境 OS 升级**。不要使用 CentOS Stream。
1. 如选用 Ubuntu 系列，Ubuntu LTS（或者 Debian）都可以，持续升级就行
1. 如考虑信创，OpenEuler 对国产 ARM 服务器兼容性良好；如果政策有倾向性，那么麒麟系列、龙蜥对应考虑

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

- Linux 容器（LXC）技术主要两个内核特性组成：namespace & cgroup。namespace 最早是 2002 年在 2.4.19 内核中引入（mount 单元），用于实现**资源隔离**。cgroup 2000 年以前就在 google 使用，2006 年以后贡献到 Linux Kernel，用于实现**资源限制**，2008 年 LXC 技术基本完成。
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

K8S 和 Docker 有什么关系？参考：[Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

Docker 和 Rocket 之争

![](/image/k8s-cri-rocket.png)

- Orchestration API -> Container API (cri-runtime) -> Kernel API(oci-runtime)

**OCI 标准是什么**？：runC / Kata（ 及其前身 runV 和 Clear Containers ），gVisor，Rust railcar

- 容器镜像的制作标准，即 ImageSpec。大致规定：容器镜像这个压缩了的文件夹里以 xxx 结构放 xxx 文件
- 容器要需要能接收哪些指令，以及这些指令的行为，即 RuntimeSpec。这里面的大致内容就是“容器”要能够执行 "create"，"start"，"stop"，"delete" 这些命令，并且行为要规范。
- Docker 则把 libcontainer 封装了一下，变成 runC 捐献出来作为 OCI 的参考实现。

**CRI 标准是什么**？：Docker（ 借助 dockershim ），containerd（ 借助 CRI-containerd ），CRI-O，Frakti。是一组 gRPC 接口，[cri-api/pkg/apis/services.go](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/services.go)：

- 一套针对容器操作的接口，包括创建，启停容器等等
- 一套针对镜像操作的接口，包括拉取镜像删除镜像等
- 一套针对 PodSandbox（容器沙箱环境）的操作接口

containerd-1.0，对 CRI 的适配通过一个单独的进程 CRI-containerd 来完成

![](/image/k8s-containerd-1-0.png)

containerd-1.1，把适配逻辑作为插件放进了 containerd 主进程中

![](/image/k8s-containerd-1-1.png)

综上：

1. 容器运行时是管理容器和容器镜像的程序。有两个标准，一个是 CRI，抽象了 kubelet 如何启动和管理容器；另一个是 OCI，抽象了怎么调用内核 API 来管理容器。标准实际上是定义了一系列接口，让上层应用与底层实现接耦。
1. 实现 CRI 的 runtime 有 CRI-O、CRI-containred 等，CRI 的命令行客户端是 crictl。containerd 的客户端是 ctr。dockerd 的客户端是 docker。它们通过 unix sock 与对应的 daemon 交互。
1. OCI 的默认实现是 runc。runc 是一个命令行工具，而不是一个 daemon。通过 runc 我们可以手动启动一个容器，也可以查看其他进程启动的容器。

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

>Note：2021 年 7 月之后，ubuntu 环境 kubeadmin 默认都是 1.22+ 版本，因此需要将 docker 的 cgroup driver 改成 systemd（原来是 cgroup）。如果不改，后续 kubeadm init 时，会报错：`[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.`
>
>检查 journalctl -x -u kubelet，可以看到：`Aug 07 15:10:45 ckalab2 kubelet[11394]: E0807 15:10:45.179485   11394 server.go:294] "Failed to run kubelet" err="failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"cgroupfs\""`
>
>看官方文档：<https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/>：`In v1.22, if the user is not setting the cgroupDriver field under KubeletConfiguration, kubeadm will default it to systemd.`
>
> 所以我们需要把 docker 的 cgroup driver 改成 systemd
>
> 修改步骤参考：<https://stackoverflow.com/questions/43794169/docker-change-cgroup-driver-to-systemd>
>
> 修改完成后，检查一下 docker cgroup，确保 docker cgroup 是 systemd 了：`sudo docker info | grep -i cgroup`

如何创建一个镜像？如何启动和调试容器？[Github](https://github.com/99cloud/lab-openstack/tree/master/src/docker-quickstart) 或 [Gitee](https://gitee.com/dev-99cloud/lab-openstack/tree/master/src/docker-quickstart)

```bash
mkdir ~/test
cd ~/test

wget https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/docker-quickstart/app.py
wget https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/docker-quickstart/requirements.txt
wget https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/docker-quickstart/Dockerfile

# 如果是 CentOS 7.x 需要安装下 python3
# yum install python3 python3-pip

# pip3 install -r requirements.txt
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple -r requirements.txt

python3 app.py
# * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
# 此时可以从浏览器访问 http://<ip>/

docker build --tag=friendlyhello .

# 可以看一下本地镜像列表
docker images

docker rm testFlask
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

参考：<https://docs.docker.com/network/#network-drivers>，Docker 网络模型分为若干种，在生产环境中会被用到的主要是 bridge 和 host 模式。

host 模式是容器和宿主机共享同一个 TCP/IP 协议栈（network namespace），容器的网络和宿主机网络不做隔离（只是网络不隔离，PID / IPC / MNT / UTS 还是 namespace 隔离的）

bridge 模式下，容器网络和宿主机网络是通过 network namespace 隔离开的，然后再通过类似 NAT 或者 port-mapping 技术完成转发。

因此，我们通过 bridge 模式来研究 network namespace 的底层逻辑。

![](/image/bridge_network.jpeg)

docker 容器实现没有把 network namespaces 放到标准路径 `/var/run/netns` 下，所以 `ip netns list` 命令看不到。但是可以看 `ll /proc/<pid>/ns`，两个进程的 namespaces id 相同说明在同一个 namespaces

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

    Docker 直接带 Containerd，Containerd 可以单独装，也可以装 Docker，再用 Docker 中的 Containerd 对接 K8S。这在某些必须使用到 docker 的场景中比较受欢迎，比如超融合场景，需要依赖 docker 部署 ceph 等

1. 直接兼容 K8S CRI
    - 不再需要 docker-shim 适配器
    - 可直接对接 K8S CRI 接口

1. 性能优良

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

| crictl | Description |
| - | - |
| imagefsinfo | Return image filesystem info |
| inspectp | Display the status of one or more pods |
| port-forward | Forward local port to a pod |
| pods | List pods |
| runp | Run a new pod |
| rmp | Remove one or more pods |
| stopp | Stop one or more running pods |

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

这里 crictl 不可用是因为没有为 crictl 命令配置 endpoint。在 centos 7 上部署的 docker 版本是 1.13，版本太低了，使用的不是 containerd 而是 libcontainerd，略有不同。可以卸载原有的 docker，重新安装高版本 docker。

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

yum list docker-ce --showduplicates | sort -r

yum install docker-ce-20.10.16 docker-ce-cli-20.10.16 containerd.io docker-compose-plugin

systemctl enable docker --now

docker version
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
1. crictl 命令的优点是和 docker 命令非常像，几乎一样。差异是 image 相关的处理逻辑（load / save / tag）缺失，这些不是 cri 考虑的范畴。这个可以由 ctr 或 nerdctl 补齐。
1. crictl 兼容 cri API，这就使得它不仅可以用于 containerd，而且适用于 CRIO 等所有支持 CRI 接口的容器运行时**。

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

crictl 不能像 ctr 那样通过参数给定用户名和密码的方式从开启认证的私有仓库中 pull 镜像。需要对 containerd 进行配置。 containerd 提供的各种功能在其内部都是通过插件实现的，可以使用 `ctr plugins ls` 查看 containerd 的插件。

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

私有镜像仓库相关的配置在 cri 插件中，文档Configure Image Registry中包含了镜像仓库的配置。 关于私有仓库和认证信息配置示例如下，修改/etc/containerd/config.toml：

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

##### 2.2.2.3 nerdctr

参考：<https://github.com/containerd/nerdctl>

nerdctr 是一个非常棒的客户端：

- 保持了和 Docker 一致的使用习惯
- 甚至兼容 Docker-Compose
- 与 docker / ctr / crictl 相比，它又有独到之处

参考：<https://github.com/containerd/nerdctl#features-present-in-nerdctl-but-not-present-in-docker>

- [Lazy-pulling is a technique to running containers before completion of pulling the images.](https://github.com/containerd/nerdctl/blob/master/docs/stargz.md)
- [Image encryption and decryption](https://github.com/containerd/nerdctl/blob/master/docs/ocicrypt.md)：This command only encrypts image layers, but does NOT encrypt container configuration such as Env and Cmd
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

1. nerdctl pull 的镜像在 default namespace 下

    ```
    [root@lab-kubernetes tmp]# ctr -n default i ls | grep nginx
    docker.io/library/nginx:alpine       application/vnd.docker.distribution.manifest.list.v2+json sha256:a74534e76ee1121d418fa7394ca930eb67440deda413848bc67c68138535b989 9.7 MiB  linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x
    ```

综上：

1. containerd 可以有多个 namespaces，default/k8s.io/moby。这里的 namespace 不是 k8s 层面的，而是 containerd 用来隔离不同的 plugin 的。
1. 通过 kubelet/crictl 启动的容器，ns 就是 k8s.io，通过 docker 启动的就是 moby。docker ps 是看不到 k8s.io 下的容器的。对于 containerd 而言， docker 和 kubelet 是两个不同的客户端。
1. nerdctl 和 ctr 默认都是 default namespaces
1. docker 启动的容器进程在 moby namespace，拉取的镜像不在 namespaces。
1. podman 不和 containerd 打交道，拉取的镜像和启动的容器都与 containerd namespace 无关。

##### 2.2.2.4 podman

podman 是 CRI-O 项目分裂出来的，由 redhat 发起。

参考：<https://github.com/containers/podman>，podman 致力于创造一个用户体验类似 docker，但没有 dockerd daemon 服务的工具。

参考：<https://podman.io/getting-started/>

```bash
yum install podman

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

- podman 有个称之为 conmon 的守护进程，它是各个容器进程的父进程，每个容器各有一个，conmon 的父进程是 1 号进程。podman 中的 conmon 相当于 docker/containerd 中的 containerd-shim。

podman 相比较 docker 的优势：

- docker 在实现 CRI 时，需要一个守护进程（dockerd daemon），该进程需要以 root 运行（有安全隐患）。而 podman 不需要守护程序，因此也不需要 root 用户运行。
- 在 docker 的运行体系中，需要多个 daemon 才能调用到 OCI 的实现 RunC（dockerd 调用 containerd，containerd 调用containerd-shim，然后才能调用 runC）podman 直接调用 OCI,runtime（runC），通过 conmon 作为容器进程的管理工具，不需要dockerd 这种以 root 身份运行的守护进程。

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

`cri-containerd-cni-1.6.5-linux-amd64.tar.gz` 包含的 runc 动态链接库与 CentOS 7 有兼容性问题。运行 runc 的时候，会遇到报错：`runc: undefined symbol: seccomp_notify_respond`

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

### 2.3 CRI-O

[返回目录](#课程目录)

CRI-O 是 RedHat 发布的容器运行时，旨在同时满足 CRI 标准和 OCI 标准。kubelet 通过 CRI 与 CRI-O 交互，CRI-O 通过 OCI 与 runC 交互，追求简单明了。

![](/image/k8s-cri-o-flow.png)

CRI 接口是：`unix:///var/run/crio/crio.sock`

![](/image/k8s-cri-o-arch-2.jpg)

在同一个集群中，混用不同的 CRI 实验，参考：<https://gobomb.github.io/post/container-runtime-note/>

安装 cri-o

```bash
yum install yum-utils
yum-config-manager --add-repo=https://cbs.centos.org/repos/paas7-crio-311-candidate/x86_64/os/
yum install --nogpgcheck cri-o
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
1. crictl / podman / runc 可以看到 pause 容器和主容器

### 2.4 Kata 和它的朋友们

[返回目录](#课程目录)

容器毕竟还是共享内核的（容器的天然不安全与天然安全），安全性和隔离型对于想要实现多租户是不够。所以又出现了许多基于虚拟机隔离的方案出来。

- [Kata Container](https://katacontainers.io/learn/)
- [Kata PDF](https://katacontainers.io/collateral/kata-containers-1pager.pdf)

![](/image/katacontainers-traditionalvskata-diagram.jpg)

![](/image/katacontainers-architecture-diagram.jpg)

除 Kata 以外，还有 Frakti / gVisor 等

- Frakti 提供了 hypervisor 级别的隔离性，提供的是内核级别的而非 Linux 命名空间级别的隔离：

    *Frakti lefts Kubernetes run pods and containers directly inside hypervisors via runV. It is light weighted and portable, but can provide much stronger isolation with independent kernel than linux-namespace-based container runtimes.*
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
1. 第四范式有一种基于 CUDA 的切分 GPU 方案，比 MIG 灵活，但不被 Nvidia 官方支持。参考：<https://github.com/4paradigm/k8s-device-plugin>

### 2.6 最佳实践

1. 生产环境建议用 Containerd，稳定性、性能和生态支持都有明显优势
1. 如果生产环境中其它依赖服务需要 docker 支持，可以安装，与 Containerd 不冲突
1. 关于命令行，首选 crictl 代替，用法和 docker 命令一致，并且能兼容所有支持 CRI 接口的容器运行时，比如 containerd / CRI-O 等。
1. Image 相关的操作，containerd 可以用自带的 ctr 命令（容器相关的也可以用 ctr，但命令格式和 docker CLI 不一致）。如果是 CRI-O，用 podman。
1. 如果需要延迟加载、P2P 镜像服务、镜像加密存取等功能，可以使用 nerdctr
1. 关于用 RPM 包还是二进制安装，it depends。如果我们可以预见不会频繁 apply patch，比如基础服务/工具：python/git，甚至 runc/containerd/docker，那么推荐 RPM。如果我们对安全或者 SLA 有较高的要求（等不及 rpm 提供方出 hotfix），自身技术实力也能充分保证， 那建议二进制部署，这样方便及时升级（和 bug 修复）。
1. 关于 GPU 支持，Nvidia 之前提供了 Docker 运行时支持，后续提供了 Containerd 支持。建议用 containerd + K8S 搭配。
1. 注意，不同的 GPU 型号对应不同的应用场景，训练和推理不混用。
1. 关于切分 GPU 支持，建议用 Nvidia 官方提供的 MIG 方案，优点是官方支持，缺点是切分数量不够灵活。第四范式的方案切分数量灵活，但不推荐上生产，因为出问题后（比如 TF 或者 Pytorch 版本兼容性问题，改方案只支持特定版本） Nvidia 会说不支持。
1. ECI 解决方案中，Kata 相对成熟，可以应用于 OpenStack Zun 或者 K8S。
1. GPU 容器方案，建议选 Nvidia 的 MIG 方法，要注意区分不同的 GPU 类型，推理或者训练。

## 3. K8S 生命周期管理

[返回目录](#课程目录)

### 3.1 生命周期管理

[返回目录](#课程目录)

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

1. 部署 Containerd 和 ctictl 等工具，参考 [2.2.2.1 Crictl](#2221-crictl)

    ```bash
    # 如果之前安装 centos 默认源的 docker，要先删除掉
    yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

    yum install -y yum-utils
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    yum install -y containerd.io

    systemctl enable containerd --now
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

    [可选]：然后部署 crictl

    ```bash
    VERSION="v1.24.1"
    wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz

    tar zxvf crictl-$VERSION-linux-amd64.tar.gz
    mv crictl /usr/bin/

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

1. 部署 Kubernetes

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

    # 8. 安装 CRI

    # 10. 下载 kubernetes
    export k8s_version="1.23.3"

    yum install -y kubelet-${k8s_version}-0 kubeadm-${k8s_version}-0 kubectl-${k8s_version}-0  --disableexcludes=kubernetes

    # 11. 启动 kubelet
    systemctl restart kubelet
    systemctl enable kubelet

    # 12. 用 kubeadm 初始化创建 K8S 集群
    kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version=v${k8s_version} --pod-network-cidr=10.244.0.0/16

    # 13. 配置 .kube/config 用于使用 kubectl
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config

    # 15. 安装 calico
    kubectl apply -f https://gitee.com/dev-99cloud/lab-openstack/raw/master/src/ansible-cloudlab-centos/playbooks/roles/init04-prek8s/files/calico-${k8s_version}.yml

    # 看到 node Ready 就 OK
    kubectl get nodes
    ```

##### 3.1.1.2 增加 CRI-O Worker 节点

拓扑结构：1 Master（Containerd） + 1 Worker（CRI-O）

此处采用 Containerd + CRI-O 混合 CRI 部署。生产环境中不会这么用（生产环境中会尽量用较少、较成熟的模块完成搭建，减少依赖，减少技术栈复杂度），这里这么实验是用于同时展示 Kubelet 对接不同 CRI 时的情况。

##### 3.1.1.3 删除 K8S

参考：<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/>

`kubectl reset -f`

#### 3.1.2 Sealos

#### 3.1.3 KubeKey

#### 3.1.4 K0S

#### 3.1.5 K3S

##### 3.1.5.1 什么 K3S

K3S 参考文档

- [Kubernetes 官网](https://kubernetes.io/zh)
- [Kubernetes 官方文档](https://kubernetes.io/zh/docs/home)
- [k3s 搭建步骤](https://github.com/k3s-io/k3s#quick-start---install-script)

Kubernetes 集群在管理大规模服务上有极佳的运维自动化优势（自愈 / 扩缩容 / 负载均衡 / 服务注册和服务发现 / 部署），但也需要耗费相对较多的资源，许多功能对特定对特定场景来说是冗余的，K3S 作为一个轻量级的 Kubernetes 发行版应运而生，它针对边缘计算、物联网等场景进行了高度优化。

K3s 有以下增强功能：

- 打包为单个二进制文件。
- 使用基于 sqlite3 的轻量级存储后端作为默认存储机制。同时支持使用 etcd3、MySQL 和 PostgreSQL 作为存储机制。
- 封装在简单的启动程序中，通过该启动程序处理很多复杂的 TLS 和选项。
- 默认情况下是安全的，对轻量级环境有合理的默认值。
- 添加了简单但功能强大的 batteries-included 功能，例如：本地存储提供程序，服务负载均衡器，Helm controller 和 Traefik Ingress controller。
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

在这种配置中，每个 **agent** 节点都注册到**同一个** server 节点。K3s 用户可以通过调用 server 节点上的 K3s API 来操作 Kubernetes 资源。外部流量则通过 Traeffic 导入，且经过 loadBalance 进行负载均衡。

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

部署脚本安装

``` bash
# curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--docker" sh -
# 受限国内网络，大多数时候上述脚本无法安装，建议采用下述国内加速安装脚本
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn INSTALL_K3S_EXEC="--docker" sh -
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
# 默认情况下，K3s 将以 flannel 作为 CNI 运行，使用 VXLAN 作为默认后端。如需修改 CNI 的模式，可以参考下述命令安装
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
- **外部数据库**：与单节点 k3s 设置中使用的嵌入式
SQLite 数据存储相反，高可用 K3s 需要**挂载一个external database 外部数据库**作为数据存储的媒介。

**固定 agent 节点的注册地址**

在高可用 K3s server 配置中，每个节点还必须使用固定的注册地址向 **Kubernetes API** 注册，注册后，agent 节点直接与其中一个 server 节点建立连接，如下图所示：

![](/image/k3s-production-setup.svg)

先决条件：节点不能有相同的主机名

| 节点角色   | CPU    | 内存  | 磁盘   | 数量 | os        |
| ------ | ------ | --- | ---- | -- | --------- |
| server | >2Core | >4G | >10G | >3 | centos7.x |
| agent  | >2Core | >2G | >10G | >3 | centos7.x |

部署外部数据库 etcd。K3S 支持多种外部数据库，这里我们选用 etcd 数据库，为方便搭建，我们这里搭建的 etcd 集群为非 TLS 访问方式
更多详情内容参阅 [K3S 外部数据库配置](https://docs.rancher.cn/docs/k3s/installation/ha/_index) 和 [etcd 官方文档](https://etcd.io)

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

下述命令中的 INSTALL_K3S_MIRROR,K3S_DATASTORE_ENDPOINT, K3S_TOKEN... 等可以作为环境变量，如 export K3S_TOKEN="k3s_token"

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

下述命令中的 INSTALL_K3S_MIRROR,K3S_DATASTORE_ENDPOINT, K3S_TOKEN... 等可以作为环境变量，如 export K3S_TOKEN="k3s_token"

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

再升级 Worker 节点，参考 <https://v1-23.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrade-worker-nodes>

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

参考 <https://kubesphere.io/docs/quick-start/minimal-kubesphere-on-k8s/>

准备检查：<https://kubesphere.io/docs/installing-on-kubernetes/introduction/prerequisites/>

### 3.5 监控、计量和告警

[返回目录](#课程目录)

### 3.6 日志管理

[返回目录](#课程目录)

### 3.7 最佳实践

1. 生产环境中：推荐 kubeadm，K8S + Kubeadm + Calico + KubeSphere
1. 边缘环境：推荐 K3S + AutoK3S + Rancher
1. KubeKey / Kubeasz / K0S 各有优势，但需要学习和深入理解（Debug）不同的工具和技术栈
1. Kubeadm 有良好稳定可靠的升级方案、集群的备份恢复方案、以及 Master 0 节点的备份恢复方案
1. KubeSphere 能良好地纳管（统一管理、认证健全、监控计量、跨集群调度）K8S；而 Rancher 可以对 K3S 较好地统一管理。

## 4. 存储管理

[返回目录](#课程目录)

### 4.1 CSI 一般概念

[返回目录](#课程目录)

### 4.2 对接 NFS 和 NAS

[返回目录](#课程目录)

#### 4.2.1 搭建 NFS Server

1. 准备好 NFS server 机器（另开一台 CentOS 7.9，单独配一块数据盘），机器规格按需指定。最好挂载一块单独的磁盘。
1. 保证需要使用 NFS 存储的客户端机器与 NFS server 机器的网络互通性
1. 部署步骤
    1. 下载相关包，并启动相关服务

        ```bash
        apt-get install nfs-common -y || yum install nfs-utils -y
        ```

    1. 创建 NFS 数据路径

        ```bash
        mkdir -p /nfs/data
        chmod -R 777 /nfs/data
        ```

    1. 如果有额外挂单独的数据盘给 NFS 用，需要格式化这块磁盘并挂载，比如 vdb。如果没有额外挂盘请忽略这一步

        ```bash
        mkfs.xfs /dev/vdb
        mount /dev/vdb /nfs/data
        echo "/dev/vdb /nfs/data xfs defaults 0 0" >> /etc/fstab
        ```

    1. 编辑 NFS 配置文件

        ```bash
        # echo "/nfs/data *(rw,no_root_squash,sync)" > /etc/exports
        echo "/nfs/data *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)" > /etc/exports
        # 首次登入Internal error occurred: account is not active 的问题，https://kubesphere.com.cn/forum/d/2058-kk-internal-error-occurred-account-is-not-active
        exportfs -r
        ```

    1. 启动 rpcbind、nfs 服务

        ```bash
        systemctl restart rpcbind && systemctl enable rpcbind
        systemctl restart nfs && systemctl enable nfs
        ```

    1. 查看 RPC 服务的注册状况

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

1. 执行命令创建 RBAC

    ```bash
    kubectl create -f RBAC.yaml
    ```

1. 创建 deployment.yaml 文件，内容如下:

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

1. 部署deploy

    ```bash
    kubectl create -f deployment.yaml
    ```

1. 创建 storageclass.yaml 文件

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

1. 创建 storage class

    ```bash
    kubectl apply -f storageclass.yaml
    ```

1. 如果要把 nfs 设置成默认 StorageClass：(option)

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

1. 创建 pvc

    ```bash
    kubectl apply -f pvc.yaml
    ```

    此时可以看一下 pvc 状态，应该是 Bound，如果是 pending，看一下 nfs driver pod 的 log。K8S 1.20 以后，会有这个报错：`unexpected error getting claim reference: selfLink was empty, can't make reference`，有两个办法，参考：<https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/issues/25>，在 api-server 的 static pod 里添加启动参数：`--feature-gates=RemoveSelfLink=false`，或者更新 NFS 驱动镜像：`gcr.io/k8s-staging-sig-storage/nfs-subdir-external-provisioner:v4.0.0`

2. 测试

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

[root@lab-c2009-ceph-aio ~]# yum install python3
[root@lab-c2009-ceph-aio ~]# python3 -V
Python 3.6.8

[root@lab-c2009-ceph-aio ~]# ls /dev/vd*
/dev/vda  /dev/vda1  /dev/vda2  /dev/vdb

[root@lab-c2009-ceph-aio ~]# yum install docker -y
[root@lab-c2009-ceph-aio ~]# systemctl enable docker --now
[root@lab-c2009-ceph-aio ~]# docker --version
Docker version 1.13.1, build 7d71120/1.13.1
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
[root@lab-c2009-ceph-aio ~]# yum install ceph-common
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

| Ceph CSI Version | Container Orchestrator Name | Version Tested |
| - | - | - |
| v3.6.1 | Kubernetes | v1.21, v1.22, v1.23 |
| v3.6.0 | Kubernetes | v1.21, v1.22, v1.23 |
| v3.5.1 | Kubernetes | v1.21, v1.22, v1.23 |
| v3.5.0 | Kubernetes | v1.21, v1.22, v1.23 |

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

这里的 clusterID 对应之前步骤中的 fsid。
imageFeatures 用来确定创建的 image 特征，如果不指定，就会使用 RBD 内核中的特征列表，但 Linux 不一定支持所有特征，所以这里需要限制一下。

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

*You can only clone a PVC when it exists in the same namespace as the destination PVC (source and destination must be in the same namespace).*

可以借用 snapshot 来完成跨 namespace 复制 PVC，参考：<https://v1-20.docs.kubernetes.io/docs/concepts/storage/volume-snapshots/>

VolumeSnapshotContent 是 cluster 级别

1. create Snapshot（会自动生成 SnapshotContent 和 volumeHandle）
2. create new SnapshotContent（根据 volumeHandle）

    注意，创建时需要带两条注解，参考：<https://github.com/kubernetes-csi/external-snapshotter/issues/251>，否则删除新创建的 pvc 时，会 hang 住

    - `snapshot.storage.kubernetes.io/deletion-secret-namespace: default`
    - `snapshot.storage.kubernetes.io/deletion-secret-name: csi-rbd-secret`

3. create Snapshot in other namespace（根据新的 SnapshotContent）
4. create pvc by snapshot（根据新的 snapshot）

底层基于快照，在 Ceph 上飞快（copy on write）

### 4.5 Local 和动态分配

[返回目录](#课程目录)

### 4.6 最佳实践

1. K8S 上的存储一般用商用 NAS（协议用 NFS）或者 CephRBD，相对成熟可靠。尽量不要用 CephFS（生产环境中存在丢数据风险）。
1. NFS 对接时，要考虑版本，v3 或 v4，Netapps 对接错版本会存在访问权限的问题
1. CephRBD 需要选用 N 以上版本，才支持 CSI 的快照等高级特性
1. PVC 复制可以借用 Snapshot，方便且快
1. Local 存在丢数据风险，需要在应用层做数据可靠性保重
1. DynamicLocal 在 K3S 环境生产级可用

## 5. 网络管理

[返回目录](#课程目录)

### 5.1 CNI 的一般概念

[返回目录](#课程目录)

### 5.2 对接 Calico

[返回目录](#课程目录)

### 5.3 对接 OVN

[返回目录](#课程目录)

### 5.4 多网络平面

[返回目录](#课程目录)

### 5.5 物理网卡加速

[返回目录](#课程目录)

### 5.6 最佳实践

1. 推荐 Calico，需要追求性能可以用 BGP 模式
1. 如果是 IPIP/vxlan 模式，需要考虑 MTU
1. 只有网元虚拟化，或者 ECS 场景，才推荐用 Multus 和 KubeOVN
1. DPDK 取决于 CNI 支持，比如 ovs-dpdk
1. 物理网卡加速取决于网卡硬件和驱动支持

## 6. 安全相关

[返回目录](#课程目录)

### 6.1 安全基线

[返回目录](#课程目录)

### 6.2 权限限制

[返回目录](#课程目录)

### 6.3 K8S 安全扫描工具

[返回目录](#课程目录)

### 6.4 容器镜像安全扫描

[返回目录](#课程目录)

### 6.5 审计日志

[返回目录](#课程目录)

### 6.6 镜像准入检查

[返回目录](#课程目录)

### 6.7 最佳实践
