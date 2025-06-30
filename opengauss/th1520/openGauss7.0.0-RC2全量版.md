# openGauss 数据库 (7.0.0-RC2 全量版) TH1520 性能测试报告

> openGauss 数据库 (7.0.0-RC2 全量版) 提供了两个安装包版本：
>
> 1. `opengauss-server-7.0.0-1.openEuler24.03.riscv64.rpm`
> 2. `opengauss-server-7.0.0-1-with_mot-with_riscv_zbc-with_riscv_v.openEuler24.03.riscv64.rpm`
>
> 其中，`opengauss-server-7.0.0-1-with_mot-with_riscv_zbc-with_riscv_v.openEuler24.03.riscv64.rpm` 版本使用了 RVV 1P0 优化，无法在 TH1520 上运行，因此本次测试基于 `opengauss-server-7.0.0-1.openEuler24.03.riscv64.rpm` 版本进行。

## 测试环境

- 硬件：TH1520 SoC (Sipeed LicheePi 4A 开发板)
- 操纵系统：openEuler 24.03 LTS-SP1

## 数据库初始化

`opengauss_install` 为解压的openGauss二进制程序目录

导出环境变量(`GAUSSHOME` 后面跟 `opengauss` 二进制包的绝对路径，根据实际情况调整，`PGDATA` 是数据库存放路径，也是根据实际情况调整)

```bash
export GAUSSHOME=/home/openeuler/opengauss_install
export LD_LIBRARY_PATH=$GAUSSHOME/lib:$LD_LIBRARY_PATH
export PATH=$GAUSSHOME/bin:$PATH
export PGDATA=/home/openeuler/gauss_data
```

初始化openGauss数据库

```bash
gs_initdb -D /home/openeuler/gauss_data -U openeuler --nodename=single_node
```

启动openGauss服务

```bash
gs_ctl start -D /home/openeuler/gauss_data -Z single_node
```

## 准备性能测试环境

按照sysbench

```bash
[openeuler@openeuler-riscv64 ~]$ sudo dnf install sysbench
```

修改 opengauss 配置文件

```bash
[openeuler@openeuler-riscv64 ~]$ vim gauss_data/postgresql.conf
# 配置 listen_addresses = '*'
# 配置 password_encryption_type = 1

gs_ctl -D $HOME/data reload
# reload 后即可生效
```

在 openGauss 中创建数据库和用户(在修改密码规则后必须新建用户或修改密码才能使用), 并授予权限用于测试

```bash
[openeuler@openeuler-riscv64 ~]$ gsql -d postgres

openGauss=# CREATE USER testuser WITH PASSWORD 'openEuler12#$';
openGauss=# CREATE DATABASE testdb owner testuser;
openGauss=# GRANT ALL ON SCHEMA public TO testuser;
GRANT
openGauss=# GRANT ALL PRIVILEGES TO testuser; 
ALTER ROLE
```

## 运行测试

初始化数据库

```bash
sysbench --db-driver=pgsql --oltp-table-size=100000 --oltp-tables-count=24 --threads=1 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb  /usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua run
```

使用下列命令验证生成的数据

```bash
[openeuler@openeuler-riscv64 ~]$ gsql -U testuser -d testdb
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
[openeuler@openeuler-riscv64 ~]$
```

执行读/写测试

```bash
sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run
```

上述命令将从名为 /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua 的 LUA 脚本生成 OLTP 工作负载，针对主服务器上 24 个表的 100,000 行（具有 10 个工作线程）持续 60 秒）。每 2 秒，sysbench 将报告中间统计信息（–report-interval=2）。

执行只读测试

```bash
sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/select.lua run
```

清理测试数据

```bash
sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/select.lua cleanup
```

## 性能测试结果

详细结果参见 [logs](./logs) 目录。

## 性能测试总结

> rw: oltp 测试,包含读写 ro:select 测试,仅读

SQL statistics:

| Platform                   | read   | write  | other | total  | transactions | tps      | queries | qps      | ignored errors | reconnects |
| -------------------------- | ------ | ------ | ----- | ------ | ------------ | -------- | ------- | -------- | -------------- | ---------- |
| **TH1520 @ 4 Threads rw**  | 47 544 | 13 584 | 6 792 | 67 920 | 3 396        | 56.55    | 67 920  | 1 131.09 | 0              | 0          |
| **TH1520 @ 10 Threads rw** | 52 500 | 14 999 | 7 501 | 75 000 | 3 750        | 62.44    | 75 000  | 1 248.72 | 0              | 0          |
| **TH1520 @ 4 Threads ro**  | 93 729 | 0      | 0     | 93 729 | 93 729       | 1 561.43 | 93 729  | 1 561.43 | 0              | 0          |
| **TH1520 @ 10 Threads ro** | 91 305 | 0      | 0     | 91 305 | 91 305       | 1 520.85 | 91 305  | 1 520.85 | 0              | 0          |


Latency (ms):

| Platform                   | min   | avg    | max    | 95th %ile | sum        |
| -------------------------- | ----- | ------ | ------ | --------- | ---------- |
| **TH1520 @ 4 Threads rw**  | 40.43 | 70.69  | 346.30 | 127.81    | 240 067.55 |
| **TH1520 @ 10 Threads rw** | 45.09 | 160.07 | 556.44 | 282.25    | 600 273.83 |
| **TH1520 @ 4 Threads ro**  | 1.23  | 2.55   | 207.64 | 6.09      | 239 151.08 |
| **TH1520 @ 10 Threads ro** | 1.33  | 6.56   | 819.57 | 12.75     | 599 202.57 |


Threads fairness:

| Platform                   | events avg | events stddev | time avg (s) | time stddev (s) |
| -------------------------- | ---------- | ------------- | ------------ | --------------- |
| **TH1520 @ 4 Threads rw**  | 849.00     | 5.20          | 60.0169      | 0.01            |
| **TH1520 @ 10 Threads rw** | 375.00     | 10.15         | 60.0274      | 0.01            |
| **TH1520 @ 4 Threads ro**  | 23 432.25  | 171.99        | 59.7878      | 0.00            |
| **TH1520 @ 10 Threads ro** | 9 130.50   | 233.60        | 59.9203      | 0.01            |

