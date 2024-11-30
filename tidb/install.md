# TiDB数据库
## 官方支持
根据[文档中的建议配置](https://docs.pingcap.com/zh/tidb/stable/hardware-and-software-requirements#%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%8F%8A%E5%B9%B3%E5%8F%B0%E8%A6%81%E6%B1%82)，当前仅支持x86_64和ARM 64，不支持risc-v架构

## 编译
### 安装依赖
```bash
cd ~
apt update && apt install -y curl git cmake make pkg-config gcc g++ clang software-properties-common libssl-dev
wget https://go.dev/dl/go1.23.3.linux-riscv64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.23.3.linux-riscv64.tar.gz
export PATH=$PATH:/usr/local/go/bin
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"
rustup install nightly
add-apt-repository ppa:ubuntu-toolchain-r/ppa -y && apt update
sudo apt install g++-12 gcc-12
```
下载并安装go, rust, gcc 12

### TiDB
```bash
cd ~
git clone https://github.com/pingcap/tidb.git
cd tidb
git checkout v8.4.0
make
```
等待编译完成后，会在bin目录下生成二进制文件`tidb-server`，运行即可启动

### TiKV
```bash
. "$HOME/.cargo/env"
export CC=clang
export CXX=clang++
cd ~
git clone https://github.com/tikv/tikv
cd tikv
git checkout v8.4.0
cargo build --no-default-features --features "portable test-engine-kv-rocksdb test-engine-raft-raft-engine"
```

### pd
```bash
cd ~
git clone https://github.com/tikv/pd.git
cd pd
git checkout v8.4.0
make
```

## 支持情况
编译[TiDB](https://github.com/pingcap/tidb)和[pd](https://github.com/tikv/pd)这两个go项目时没有遇到问题，可以正常生成二进制产物并启动

对于[TiKV](https://github.com/tikv/tikv)，虽有第三方的支持，但无论是官方的还是第三方fork，都会遇到以下问题：

编译时若使用gcc11/12/13工具链，会得到报错

```log
error: unrecognized command-line option '-mno-omit-leaf-frame-pointer'; did you mean '-fno-omit-frame-pointer'?
```

若切换clang工具链，在编译[`tikv-jemalloc-sys-0.5.0+5.3.0`](https://github.com/tikv/jemallocator/tree/tikv-jemalloc-sys-0.5.3)时会报错

```log
Invalid configuration `riscv64gc-unknown-linux-gnu': machine `riscv64gc-unknown' not recognized
```

## ref
https://docs.pingcap.com/zh/
https://github.com/KevinMX/PLCT-Works/blob/main/misc/month12/TiDB/TiDB.md
https://tikv.github.io/tikv-dev-guide/get-started/build-tikv-from-source.html
https://github.com/xhebox/tikv/tree/rv64
