# openGauss 数据库 (7.0.0-RC2 全量版) SG2042 性能测试报告

> openGauss 数据库 (7.0.0-RC2 全量版) 提供了两个安装包版本：
>
> 1. `opengauss-server-7.0.0-1.openEuler24.03.riscv64.rpm`
> 2. `opengauss-server-7.0.0-1-with_mot-with_riscv_zbc-with_riscv_v.openEuler24.03.riscv64.rpm`
>
> 其中，`opengauss-server-7.0.0-1-with_mot-with_riscv_zbc-with_riscv_v.openEuler24.03.riscv64.rpm` 版本使用了 RVV 1P0 优化，进迭时空K1/M1上可以运行，但由于openEuler 24.03 LTS-SP1测试在Milk-V Jupiter 开发板上未能成功启动，本次在Bianbu 2.1系统测试两个安装包版本。

## 测试环境

- 硬件： Spacemit K1/M1 SoC (Milk-V Jupiter 开发板)
- 操纵系统：Bianbu 2.1

## 数据库初始化

`opengauss` 为解压的openGauss二进制程序目录

导出环境变量(`GAUSSHOME` 后面跟 `opengauss` 二进制包的绝对路径，根据实际情况调整，`PGDATA` 是数据库存放路径，也是根据实际情况调整)

```bash
export GAUSSHOME=/home/rili/extract/opengauss_nov/opt/opengauss
export PGDATA=/home/rili/data_nov
export LD_LIBRARY_PATH=$GAUSSHOME/lib:$LD_LIBRARY_PATH
export PATH=$GAUSSHOME/bin:$PATH
export PGSYSCONFDIR=$GAUSSHOME/etc/postgresql
```

初始化openGauss数据库

```bash
gs_initdb -D /home/rili/gauss_data -U rili --nodename=single_node
```

启动openGauss服务

```bash
gs_ctl start -D /home/rili/gauss_data -Z single_node
```

## 准备性能测试环境

按照sysbench

```bash
[rili@openeuler-riscv64 ~]$ sudo apt install sysbench
```apt

修改 opengauss 配置文件

```bash
[rili@k1 ~]$ vim gauss_data/postgresql.conf
# 配置 listen_addresses = '*'
# 配置 password_encryption_type = 1

gs_ctl -D $HOME/data reload
# reload 后即可生效
```

在 openGauss 中创建数据库和用户(在修改密码规则后必须新建用户或修改密码才能使用), 并授予权限用于测试

```bash
[rili@k1 ~]$ gsql -d postgres

openGauss=# CREATE USER testuser WITH PASSWORD 'Test123456!';
openGauss=# CREATE DATABASE testdb owner testuser;
openGauss=# GRANT ALL ON SCHEMA public TO testuser;
GRANT
openGauss=# GRANT ALL PRIVILEGES TO testuser; 
ALTER ROLE
```

## 运行测试

初始化数据库

```bash
sysbench --db-driver=pgsql --oltp-table-size=100000 --oltp-tables-count=24 --threads=1 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=Test123456! --pgsql-db=testdb  /usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua run
```

使用下列命令验证生成的数据

```bash
[rili@k1 ~]$ gsql -U testuser -d testdb
Password for user testuser:
gsql ((openGauss 7.0.0-RC2 build ) compiled at 2025-06-18 08:00:46 commit 0 last mr  )
Non-SSL connection (SSL connection is recommended when requiring high-security)
Type "help" for help.

testdb=> \dt
                            List of relations
 Schema |   Name   | Type  |  Owner   |             Storage
--------+----------+-------+----------+----------------------------------
 public | sbtest1  | table | testuser | {orientation=row,compression=no}
 public | sbtest10 | table | testuser | {orientation=row,compression=no}
 public | sbtest11 | table | testuser | {orientation=row,compression=no}
 public | sbtest12 | table | testuser | {orientation=row,compression=no}
 public | sbtest13 | table | testuser | {orientation=row,compression=no}
 public | sbtest14 | table | testuser | {orientation=row,compression=no}
 public | sbtest15 | table | testuser | {orientation=row,compression=no}
 public | sbtest16 | table | testuser | {orientation=row,compression=no}
 public | sbtest17 | table | testuser | {orientation=row,compression=no}
 public | sbtest18 | table | testuser | {orientation=row,compression=no}
 public | sbtest19 | table | testuser | {orientation=row,compression=no}
 public | sbtest2  | table | testuser | {orientation=row,compression=no}
 public | sbtest20 | table | testuser | {orientation=row,compression=no}
 public | sbtest21 | table | testuser | {orientation=row,compression=no}
 public | sbtest22 | table | testuser | {orientation=row,compression=no}
 public | sbtest23 | table | testuser | {orientation=row,compression=no}
 public | sbtest24 | table | testuser | {orientation=row,compression=no}
 public | sbtest3  | table | testuser | {orientation=row,compression=no}
 public | sbtest4  | table | testuser | {orientation=row,compression=no}
 public | sbtest5  | table | testuser | {orientation=row,compression=no}
 public | sbtest6  | table | testuser | {orientation=row,compression=no}
 public | sbtest7  | table | testuser | {orientation=row,compression=no}
 public | sbtest8  | table | testuser | {orientation=row,compression=no}
 public | sbtest9  | table | testuser | {orientation=row,compression=no}
(24 rows)

testdb=> \d sbtest1
                             Table "public.sbtest1"
 Column |      Type      |                      Modifiers
--------+----------------+------------------------------------------------------
 id     | integer        | not null default nextval('sbtest1_id_seq'::regclass)
 k      | integer        | not null default 0
 c      | character(120) | not null default NULL::bpchar
 pad    | character(60)  | not null default NULL::bpchar
Indexes:
    "sbtest1_pkey" PRIMARY KEY, btree (id) TABLESPACE pg_default
    "k_1" btree (k) TABLESPACE pg_default

testdb=> \q
[rili@k1 ~]$
```

执行读/写测试

```bash
sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=Test123456! --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run
```

上述命令将从名为 /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua 的 LUA 脚本生成 OLTP 工作负载，针对主服务器上 24 个表的 100,000 行（具有 10 个工作线程）持续 60 秒）。每 2 秒，sysbench 将报告中间统计信息（–report-interval=2）。

执行只读测试

```bash
sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=Test123456! --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/select.lua run
```

清理测试数据

```bash
sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=Test123456! --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/select.lua cleanup
```

## 性能测试结果

详细结果参见 [logs](./logs) 目录。

## 性能测试总结
### NO RVV
> rw: oltp 测试,包含读写 ro:select 测试,仅读

SQL statistics:

| Platform        |    read |  write |  other |   total | transactions |      tps | queries |      qps | ignored errors | reconnects |
| --------------- | ------: | -----: | -----: | ------: | -----------: | -------: | ------: | -------: | -------------: | ---------: |
| K1\_nov\_rw\_4  |  55 202 | 15 772 |  7 886 |  78 860 |        3 943 |    65.66 |  78 860 | 1 313.27 |              0 |          0 |
| K1\_nov\_rw\_10 |  82 852 | 23 670 | 11 836 | 118 358 |        5 917 |    98.46 | 118 358 | 1 969.56 |              1 |          0 |
| K1\_nov\_ro\_4  |  89 453 |      0 |      0 |  89 453 |       89 453 | 1 490.57 |  89 453 | 1 490.57 |              0 |          0 |
| K1\_nov\_ro\_10 | 156 196 |      0 |      0 | 156 196 |      156 196 | 2 601.81 | 156 196 | 2 601.81 |              0 |          0 |


Latency (ms):


| Platform        |   min |    avg |    max |   95th |        sum |
| --------------- | ----: | -----: | -----: | -----: | ---------: |
| K1\_nov\_rw\_4  | 48.44 |  60.88 | 272.29 |  77.19 | 240 056.78 |
| K1\_nov\_rw\_10 | 54.13 | 101.45 | 618.49 | 176.73 | 600 304.20 |
| K1\_nov\_ro\_4  |  1.63 |   2.67 | 219.69 |   3.49 | 238 730.76 |
| K1\_nov\_ro\_10 |  1.69 |   3.83 | 343.83 |   7.98 | 597 455.09 |


Threads fairness:

| Platform        | events avg | events stddev | time avg (s) | time stddev (s) |
| --------------- | ---------: | ------------: | -----------: | --------------: |
| K1\_nov\_rw\_4  |     985.75 |          6.22 |      60.0142 |            0.01 |
| K1\_nov\_rw\_10 |     591.70 |         15.63 |      60.0304 |            0.03 |
| K1\_nov\_ro\_4  |  22 363.25 |         44.55 |      59.6827 |            0.01 |
| K1\_nov\_ro\_10 |  15 619.60 |        559.53 |      59.7455 |            0.01 |

### RVV


SQL statistics:

| Platform            |    read |  write |  other |   total | transactions |      tps | queries |      qps | ignored errors | reconnects |
| ------------------- | ------: | -----: | -----: | ------: | -----------: | -------: | ------: | -------: | -------------: | ---------: |
| K1\_rvv\_rw\_4  |  57 428 | 16 408 |  8 204 |  82 040 |        4 102 |    68.33 |  82 040 | 1 366.67 |              0 |          0 |
| K1\_rvv\_rw\_10 |  81 872 | 23 391 | 11 697 | 116 960 |        5 848 |    97.20 | 116 960 | 1 944.09 |              0 |          0 |
| K1\_rvv\_ro\_4  |  91 579 |      0 |      0 |  91 579 |       91 579 | 1 525.85 |  91 579 | 1 525.85 |              0 |          0 |
| K1\_rvv\_ro\_10 | 155 265 |      0 |      0 | 155 265 |      155 265 | 2 586.51 | 155 265 | 2 586.51 |              0 |          0 |


Latency (ms):

| Platform            |   min |    avg |    max |   95th |        sum |
| ------------------- | ----: | -----: | -----: | -----: | ---------: |
| K1\_rvv\_rw\_4  | 47.18 |  58.50 | 276.40 |  71.83 | 239 968.71 |
| K1\_rvv\_rw\_10 | 52.83 | 102.75 | 585.04 | 176.73 | 600 869.46 |
| K1\_rvv\_ro\_4  |  1.55 |   2.61 | 197.95 |   3.30 | 238 707.77 |
| K1\_rvv\_ro\_10 |  1.70 |   3.85 | 356.56 |   7.98 | 597 423.53 |

Threads fairness:

| Platform            | events avg | events stddev | time avg (s) | time stddev (s) |
| ------------------- | ---------: | ------------: | -----------: | --------------: |
| K1\_rvv\_rw\_4  |    1 025.5 |          4.72 |      59.9922 |            0.01 |
| K1\_rvv\_rw\_10 |      584.8 |         18.24 |      60.0869 |            0.03 |
| K1\_rvv\_ro\_4  |   22 894.8 |         38.75 |      59.6769 |            0.01 |
| K1\_rvv\_ro\_10 |   15 526.5 |        414.87 |      59.7424 |            0.01 |
