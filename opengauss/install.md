# 安装openGauss数据库
因为[官网提供的下载中](https://opengauss.org/zh/download/)没有riscv架构的，所以需要手动构建并安装opengauss数据库

此文档针对riscv平台编写，在其他平台下使用请自行配置qemu

## 编译
下载源码

```bash
git clone https://gitee.com/opengauss/riscv opengauss-riscv
cd opengauss-riscv
```

启动openeuler容器

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

编译rpm包

```bash
rpmbuild -ba opengauss-server.spec
```

等待一段时间，编译完成后，即可安装

```bash
cd ../RPMS/riscv64/
dnf install -y opengauss-server-5.1.0-1.riscv64.rpm
```

```log
Last metadata expiration check: 1:03:44 ago on Tue Nov 12 06:16:13 2024.
Dependencies resolved.
============================================================================================================================================================================================
 Package                                           Architecture                             Version                                     Repository                                     Size
============================================================================================================================================================================================
Installing:
 opengauss-server                                  riscv64                                  5.1.0-1                                     @commandline                                   13 M

Transaction Summary
============================================================================================================================================================================================
Install  1 Package

Total size: 13 M
Installed size: 104 M
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                    1/1 
  Running scriptlet: opengauss-server-5.1.0-1.riscv64                                                                                                                                   1/1 
  Installing       : opengauss-server-5.1.0-1.riscv64                                                                                                                                   1/1 
  Running scriptlet: opengauss-server-5.1.0-1.riscv64                                                                                                                                   1/1 
  Verifying        : opengauss-server-5.1.0-1.riscv64                                                                                                                                   1/1 

Installed:
  opengauss-server-5.1.0-1.riscv64                                                                                                                                                          

Complete!
```

## 初始化 & 启动
原readme中写的systemd无法在docker容器中使用，故手动模拟它的行为

```bash
su opengauss
export GAUSSHOME=/opt/opengauss
cd /var/lib/opengauss
source /opt/opengauss/init-opengaussdb.sh
/opt/opengauss/bin/gs_ctl start -D /var/lib/opengauss/data
```

日志显示成功启动

```log
[2024-11-12 07:41:33.603][45621][][gs_ctl]:  done
[2024-11-12 07:41:33.603][45621][][gs_ctl]: server started (/var/lib/opengauss/data)
```

## ref
https://gitee.com/opengauss/riscv/blob/master/README.md
