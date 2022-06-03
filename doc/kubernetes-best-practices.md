# Kubernetes 最佳实践

## 注意 ⚠️

- *斜体表示引用*
- **未经允许，禁止转载**

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

#### 1.1.2 Python 3.8

Python 2 于 2020.01.01 结束支持，Python 3.6 于 2021.12 结束支持。

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

综上，CentOS 8 已经停止支持，CentOS 8 Stream 不稳定，两者都不要用于生产了。

### 1.4 Rocky

[返回目录](#课程目录)

参考：<https://www.logicweb.com/what-is-rocky-linux/>

2020 年 12 月（IBM 收购 Red Hat 之后），收购 CentOS 的 Red Hat 宣布，CentOS Linux 8 将于 2021 年底结束支持，比早些时候承诺的 10 年计划要短得多。CentOS Stream 是 development 版本，将按计划在2024年结束。从此以后 CentOS 将位于 RHEL 的上游，而不是下游，CentOS 用户实际上将是 RHEL 的测试人员。

这引发了广大 CentOS 用户的极大不满，CentOS 创始人 Gregory Kurtzer 随即宣称领导创建新的 “CentOS” 发行版的工作。Rocky 这个名字是为了纪念已故 CentOS 联合创始人 Rocky McGaugh。Kurtzer 曾在加州大学伯克利分校从事高性能计算工作很长时间。鉴于CentOS 在 CERN 等机构的粒子物理学中得到了广泛应用，这可能会是Rocky Linux 的主要关注点之一。

参考：<https://rockylinux.org/>

*Enterprise Linux, the community way.
Rocky Linux is an open-source enterprise operating system designed to be 100% bug-for-bug compatible with Red Hat Enterprise Linux®. It is under intensive development by the community.*

参考：<https://rockylinux.org/download/>，Planned EOL: 2029/05/31

综上，Rocky 是另起炉灶的 CentOS Linux。当下进展不错。参考 <https://github.com/rocky-linux/rocky>，2021/06，Rocky Linux 已经生产环境 ready。建议在生产环境中使用（CentOS 7 结束支持前半年，也就是 2023 年应该转向 Rocky Linux）。

### 1.5 Openeuler

[返回目录](#课程目录)

参考：<https://developer.huaweicloud.com/ict/en/site-euleros/euleros>，EulerOS 是华为推出的企业 Linux 系统（诞生于华为科学实验室），Openeuler 是 EulerOS 的社区版本，2019 年开源，现贡献给开放原子基金会，代码托管在 <https://gitee.com/openeuler>。

OpenEuler 兼容 CentOS（但是它并不是蹭 CentOS 8 结束支持热点才开源的，开源在那之前），相较 CentOS 对核内关键功能如进程调度、内存管理、IO 读写进行了深度优化；同时在核外构筑了容器 iSula、虚拟化 StraitVirt、机密计算 SecGear、毕昇 JDK（Huawei JDK 的开源版本）。

优势是国产、信创，能较好地适配国产 ARM 服务器（特别是华为鲲鹏、泰山等）。缺点是兼容性和稳定性验证稍显不足。

### 1.6 龙蜥

[返回目录](#课程目录)

龙蜥由阿里巴巴在 2021/10/20 孵化出来，诞生背景是 CentOS 8 结束支持（CentOS 8 结束支持这事，堪称“一鲸落，万物生”）。

参考：<https://openanolis.cn/anolisos>，100% 兼容 CentOS 8 软件生态。

### 1.7 麒麟

这个就说来话长了，我也查了半天，诸君姑且听之。

- **中标麒麟**：2010/12/16，中标 Linux 和军方的“银河麒麟”在上海宣布合并，开发方中标软件有限公司和国防科技大学同日缔结了战略合作协议，双方今后将共同以“中标麒麟”的新品牌出现。
- **银河麒麟** Kylin Operating System 是天津麒麟信息技术有限公司旗下的国产 Linux 操作系统，源自国防科大“麒麟”、“银河麒麟”操作系统，支持主流 X86 架构 CPU 以及国产飞腾 CPU 平台。国防科大继续了老“麒麟”的开发。
- **优麒麟** UbuntuKylin 是 Ubuntu 社区中面向中文用户的 Ubuntu 衍生版本，中文名称优麒麟，与麒麟系统没有关系。优麒麟有两个身份，首先它是 Ubuntu 的一个官方 Flavor 版本。其次，它背后也有国防科大和天津麒麟的支持，可以看做银河麒麟的社区版。优麒麟最初的目标是像 Ubuntu 一样占领中国市场，可是很多人直接选择了 Ubuntu，一些选择了更接地气的 Deepin，所以优麒麟并不算非常成功。
- **湖南麒麟**，湖南麒麟信息工程技术有限公司（简称湖南麒麟）是 2007 年成立的一家民营企业，公司成立之初依托国防科技大学计算机学院，长期致力于信息安全的研发，在集中管控和机要密码等领域有一定的影响力。2014 年天津麒麟成立时，国防科大将“麒麟”、“银河麒麟”等无形资产注入了天津麒麟，湖南麒麟原有的操作系统研发团队整体转入天津麒麟。现在的湖南麒麟，只是一家单纯的民营企业了，可以视为一个新的系统。

emmmmm，信创上有一定优势。

## 2. 容器运行时

[返回目录](#课程目录)

### 2.1 Docker

[返回目录](#课程目录)

### 2.2 Containerd

[返回目录](#课程目录)

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
