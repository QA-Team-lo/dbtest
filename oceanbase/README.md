# OceanBase数据库
## 官方支持
[文档信息](https://oceanbase.github.io/oceanbase/zh/toolchain/)显示，当前支持仅有x86_64，不支持risc-v架构

## 编译
### toolchain
官方提供的依赖均可成功安装
```bash
sudo apt install git wget rpm rpm2cpio cpio make build-essential binutils m4
```

### compile
使用代码仓中的 `build.sh` 编译，会检测到系统架构并报错

```log
[dep_create.sh] [ERROR] 'Ubuntu 22.04.5 LTS (riscv64)' is not supported yet.
```

## 社区尝试
有开发者提交了[PR#847](https://github.com/oceanbase/oceanbase/pull/847)，尝试直接使用cmake直接构建

但使用他的分支 [loongarch64/oceanbase](https://github.com/loongarch64/oceanbase) 编译，也会得到报错

```log
[ERROR] 'Ubuntu 22.04.5 LTS (riscv64)' is not supported yet.
```
