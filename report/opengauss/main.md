# 简介

## 软件说明
openGauss 是一个免费的开源关系型数据库管理系统，主要由华为开发和维护。它是一个广泛使用的代码库，为企业级应用提供了高性能、高可用性和高安全性的数据库解决方案。

## 测试目的
本次测试旨在验证 openGauss 在 RISC-V 平台上的可用性，特别是在 Milk-V Pioneer Box 和 Sipeed LicheePi 4A 两个典型平台上的表现。本报告通过手动测试的方法，从目前的平台兼容性及用户的日常使用体验两个角度评估了 openGauss 当前在 RISC-V 平台上的可用性，并给出了定性和定量的结论，为其未来进一步的优化和支持提供参考。

## 测试概述
本次测试在 RISC-V 设备 Milk-V Pioneer Box 和 Sipeed LicheePi 4A 的多个 Linux 发行版上对多个版本的 openGauss 进行了 sysbench 测试。对其目前在 RISC-V 上的可用性进行了较为全面的测试并得出了相应的结论。

本报告在部分 Linux 发行版下还使用了 sysbench 以测试数据库的性能。

## 测试总结

性能测试结果如下：

- sysbench：

| Pioneer Box | LicheePi 4A | x86-64 参考 |
|-------------|-------------|-------------|
|    TBD      |    TBD      |     TBD     |

其余未测试系统及原因如下：

# 环境说明

## 硬件环境
本次测试主要在 Milk-V Pioneer Box 和 Sipeed LicheePi 4A 上进行，机器硬件配置为：

Milk-V Pioneer Box:
- CPU: SG2042 64 Core C920@2.0GHz
- RAM: 4 channel 3200Hz 128GB DDR4 SODIMM (32GB * 4)
- SSD: PCIe 3.0 x 4 1TB
- GPU: AMD R5 230

Sipeed LicheePi 4A:
- CPU: TH1520, RISC-V 2.0G C910 x4
- RAM: 16 GB 64bit LPDDR4X-3733
- Storage: 128 GB eMMC

## 软件环境

本次测试涵盖的系统版本和 openGauss 版本如下：

https://gitee.com/opengauss/riscv 6.0.0

## 环境搭建

### 安装 openEuler

#### Sipeed LicheePi 4A

LicheePi 4A 各个系统在[支持矩阵](https://github.com/ruyisdk/support-matrix/tree/main/LicheePi4A)上详细记载了安装过程，可作为参考。

<!-- TODO: openeuler -->

#### Milk-V Pioneer Box

下载[系统镜像](https://mirrors.hust.edu.cn/openeuler/openEuler-24.03-LTS/embedded_img/riscv64/SG2042/openEuler-24.03-LTS-riscv64-sg2042.img.zip)，解压，使用 `dd` 烧录至 NVMe 硬盘。
下载[固件](https://mirrors.hust.edu.cn/openeuler/openEuler-24.03-LTS/embedded_img/riscv64/SG2042/sg2042_firmware_linuxboot.img.zip)，解压，使用 `dd` 烧录至 microSD 卡。

请将下面的 `/dev/sda` `/dev/sdb` 替换成实际使用的硬盘和存储卡位置。

```shell!
unzip openEuler-24.03-LTS-riscv64-sg2042.img.zip
sudo wipefs -af /dev/sda
sudo dd if=openEuler-24.03-LTS-riscv64-sg2042.img of=/dev/sda bs=1M status=progress
sudo eject /dev/sda
unzip sg2042_firmware_linuxboot.img.zip
sudo dd if=sg2042_firmware_linuxboot.img of=/dev/sdb bs=1M status=progress
```
将存储卡和硬盘插入系统上电开机。

### 安装 openGauss 数据库

因为[官网提供的下载中](https://opengauss.org/zh/download/)没有 riscv 架构的，所以需要手动构建并安装 opengauss 数据库

此文档针对 riscv 平台编写，在其他平台下使用请自行配置 qemu

#### 编译
下载源码

```bash
git clone https://gitee.com/opengauss/riscv opengauss-riscv
cd opengauss-riscv
```

启动 openeuler 容器

```bash
docker run -it --name oerv-opengauss-build -v$(pwd):/root/rpmbuild/SOURCES xfan1024/openeuler:24.03-riscv64
```

在容器中配置编译环境

```
cd /root/rpmbuild/SOURCES

# 安装必要工具
dnf install -y rpm-build rpmdevtools dnf-plugins-core
# 安装编译依赖
yum-builddep -y opengauss-server.spec
# 下载源码
spectool -g opengauss-server.spec
```

编译 rpm 包

```bash
rpmbuild -ba opengauss-server.spec
```

#### 安装
等待一段时间，编译完成后，即可安装

```bash
cd ../RPMS/riscv64/
dnf install -y opengauss-server-5.1.0-1.riscv64.rpm
```

#### 初始化 & 启动

```


### 功能测试

...

### 性能测试

...

# 测试内容

## 手动测试

...

## 性能测试

...

# 测试结果


## 功能测试

...

## 性能测试

...

# 总结

<!-- TODO:总结 -->


