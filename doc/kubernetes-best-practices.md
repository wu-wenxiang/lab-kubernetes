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

### 1.9 结论

1. 2023/06/30 之前，CentOS 7 没问题，稳定支持；但之后要选择 AlmaLinux 或者 Rocky Linux，二选一。生产环境中不要用 CentOS Stream。
1. Ubuntu LTS（或者 Debian）也可以，持续升级就行
1. 如果考虑信创，OpenEuler 对国产 ARM 服务器兼容性良好；如果政策有倾向性，那么麒麟系列、龙蜥看着选

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

- Linux 容器（LXC）技术主要两个内核特性组成：namespace & cgroup。namespace 最早是 2002 年在 2.4.19 内核中引入（mount 单元），用于实现**资源隔离**。cgroup 2000 年以前就在 google 使用，2006 年以后贡献到 Linux Kernel，用于实现**资源限制**，2008 年 LXC 技术基本完成
- Docker 在 2013 年成立之后，对 LXC 进行封装，提供了极简的容器使用方案，几乎成为容器的代名词

Docker Overview，参考：<https://docs.docker.com/engine/docker-overview>

![](/image/docker-arch.png)

![](/image/docker-undertech.png)

Docker QuickStart，参考：<https://docs.docker.com/get-started>

#### 2.1.2 Docker 和 K8S

**K8S 1.24 开始放弃对 Docker 的支持，放弃的是什么？**

![](/image/k8s-CRI-OCI-docker.png)

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

CRI-O，更为专注的 cri-runtime，非常纯粹，兼容 CRI 和 OCI，做一个 Kubernetes 专用的运行时

![](/image/k8s-cri-o-flow.png)

![](/image/k8s-cri-o-arch-2.jpg)

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

生产环境中 Containerd 作为容器运行时应用最为广泛

![](/image/containerd-eco-system.jpeg)

[Containerd 的优势](https://icloudnative.io/posts/getting-started-with-containerd/)：

1. 兼容 Docker
    - Docker 直接带 Containerd，Containerd 可以单独装，也可以装 Docker，再用 Docker 中的 Containerd 对接 K8S。这在某些必须使用到 docker 的场景中比较受欢迎，比如超融合场景，需要依赖 docker 部署 ceph 等
    - crictl 命令和 docker 命令用法基本一样，同一家开发的东西

        ![](/image/k8s-cri-tools.png)

1. 直接兼容 K8S CRI
    - 不再需要 docker-shim 适配器
    - 可直接对接 K8S CRI 接口

### 2.3 CRI-O

[返回目录](#课程目录)

### 2.4 Kata

[返回目录](#课程目录)

### 2.5 GPU

[返回目录](#课程目录)

## 3. K8S 生命周期管理

[返回目录](#课程目录)

### 3.1 生命周期管理

[返回目录](#课程目录)

### 3.2 版本升级

[返回目录](#课程目录)

### 3.3 迁移和纳管

[返回目录](#课程目录)

### 3.4 统一认证和鉴权

[返回目录](#课程目录)

### 3.5 监控、计量和告警

[返回目录](#课程目录)

### 3.6 日志管理

[返回目录](#课程目录)

## 4. 存储管理

[返回目录](#课程目录)

### 4.1 CSI 一般概念

[返回目录](#课程目录)

### 4.2 对接 NFS 和 NAS

[返回目录](#课程目录)

### 4.3 对接 Ceph RBD

[返回目录](#课程目录)

### 4.4 跨命名空间的快照和备份

[返回目录](#课程目录)

### 4.5 Local 和动态分配

[返回目录](#课程目录)

## 5. 网络管理

[返回目录](#课程目录)

### 5.1 CNI 的一般概念

[返回目录](#课程目录)

### 5.2 对接 Flannel

[返回目录](#课程目录)

### 5.3 对接 Calico

[返回目录](#课程目录)

### 5.4 对接 OVN

[返回目录](#课程目录)

### 5.5 多网络平面

[返回目录](#课程目录)

### 5.6 物理网卡加速

[返回目录](#课程目录)

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
