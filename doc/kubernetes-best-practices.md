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
```

### 1.2 Ubuntu LTS

[返回目录](#课程目录)

### 1.3 Rocky

[返回目录](#课程目录)

### 1.4 CentOS 8 Stream

[返回目录](#课程目录)

### 1.5 OpenEula

[返回目录](#课程目录)

### 1.6 龙蜥

[返回目录](#课程目录)

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
