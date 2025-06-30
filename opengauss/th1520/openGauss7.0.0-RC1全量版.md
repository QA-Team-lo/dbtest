# openGauss 数据库 (7.0.0-RC1 全量版) TH1520 性能测试报告

## 测试环境

- 硬件：TH1520 SoC (Sipeed LicheePi 4A 开发板)
- 操纵系统：openEuler 24.03 LTS-SP1

## 数据库初始化

`opengauss_install_c910` 为解压的openGauss二进制程序目录

导出环境变量(`GAUSSHOME` 后面跟 `opengauss` 二进制包的绝对路径，根据实际情况调整，`PGDATA` 是数据库存放路径，也是根据实际情况调整)

```bash
export GAUSSHOME=/home/openeuler/opengauss_install_c910
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
gsql ((openGauss 7.0.0-RC1 build db163840) compiled at 2025-04-06 01:56:58 commit 0 last mr  )
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

### 读写性能

#### 10个线程读写(rw)

```bash
[openeuler@openeuler-riscv64 ~]$ sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run

sysbench: /home/openeuler/opengauss_install_c910/lib/libpq.so.5: no version information available (required by sysbench)
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 10
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 2s ] thds: 10 tps: 69.57 qps: 1442.58 (r/w/o: 1016.22/282.25/144.11) lat (ms,95%): 337.94 err/s: 0.00 reconn/s: 0.00
[ 4s ] thds: 10 tps: 84.88 qps: 1706.01 (r/w/o: 1197.75/338.51/169.75) lat (ms,95%): 189.93 err/s: 0.00 reconn/s: 0.00
[ 6s ] thds: 10 tps: 79.18 qps: 1579.16 (r/w/o: 1104.56/316.23/158.37) lat (ms,95%): 183.21 err/s: 0.00 reconn/s: 0.00
[ 8s ] thds: 10 tps: 81.47 qps: 1626.33 (r/w/o: 1136.03/327.36/162.93) lat (ms,95%): 196.89 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 10 tps: 85.46 qps: 1731.15 (r/w/o: 1212.90/347.83/170.42) lat (ms,95%): 183.21 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 10 tps: 74.45 qps: 1466.51 (r/w/o: 1022.81/295.80/147.90) lat (ms,95%): 227.40 err/s: 0.00 reconn/s: 0.00
[ 14s ] thds: 10 tps: 82.03 qps: 1627.94 (r/w/o: 1142.58/320.31/165.04) lat (ms,95%): 186.54 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 10 tps: 89.26 qps: 1785.20 (r/w/o: 1247.08/359.61/178.52) lat (ms,95%): 173.58 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 10 tps: 76.27 qps: 1544.80 (r/w/o: 1083.20/308.56/153.03) lat (ms,95%): 204.11 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 10 tps: 83.46 qps: 1677.62 (r/w/o: 1180.38/330.33/166.91) lat (ms,95%): 176.73 err/s: 0.00 reconn/s: 0.00
[ 22s ] thds: 10 tps: 87.24 qps: 1723.99 (r/w/o: 1202.03/347.47/174.48) lat (ms,95%): 176.73 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 10 tps: 77.56 qps: 1534.32 (r/w/o: 1069.95/309.25/155.12) lat (ms,95%): 196.89 err/s: 0.00 reconn/s: 0.00
[ 26s ] thds: 10 tps: 83.06 qps: 1682.33 (r/w/o: 1179.45/336.77/166.12) lat (ms,95%): 179.94 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 10 tps: 81.44 qps: 1632.88 (r/w/o: 1144.74/326.27/161.87) lat (ms,95%): 200.47 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 10 tps: 81.05 qps: 1615.50 (r/w/o: 1130.21/322.21/163.08) lat (ms,95%): 179.94 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 10 tps: 82.94 qps: 1665.89 (r/w/o: 1168.72/331.28/165.89) lat (ms,95%): 186.54 err/s: 0.00 reconn/s: 0.00
[ 34s ] thds: 10 tps: 83.55 qps: 1679.98 (r/w/o: 1178.70/334.18/167.09) lat (ms,95%): 179.94 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 10 tps: 81.03 qps: 1588.12 (r/w/o: 1104.43/321.63/162.06) lat (ms,95%): 193.38 err/s: 0.00 reconn/s: 0.00
[ 38s ] thds: 10 tps: 83.20 qps: 1672.47 (r/w/o: 1170.78/335.77/165.92) lat (ms,95%): 183.21 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 10 tps: 77.12 qps: 1582.68 (r/w/o: 1112.32/315.62/154.74) lat (ms,95%): 231.53 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 10 tps: 83.27 qps: 1646.03 (r/w/o: 1148.40/331.09/166.54) lat (ms,95%): 193.38 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 10 tps: 84.21 qps: 1684.16 (r/w/o: 1182.42/333.32/168.42) lat (ms,95%): 186.54 err/s: 0.00 reconn/s: 0.00
[ 46s ] thds: 10 tps: 81.54 qps: 1657.40 (r/w/o: 1158.62/335.71/163.07) lat (ms,95%): 183.21 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 10 tps: 76.81 qps: 1489.66 (r/w/o: 1040.56/295.49/153.61) lat (ms,95%): 200.47 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 10 tps: 86.67 qps: 1753.79 (r/w/o: 1232.24/348.21/173.34) lat (ms,95%): 173.58 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 10 tps: 76.75 qps: 1570.04 (r/w/o: 1093.51/323.04/153.49) lat (ms,95%): 170.48 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 10 tps: 82.97 qps: 1648.93 (r/w/o: 1158.60/324.39/165.94) lat (ms,95%): 179.94 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 10 tps: 87.76 qps: 1718.60 (r/w/o: 1199.45/343.62/175.53) lat (ms,95%): 173.58 err/s: 0.00 reconn/s: 0.00
[ 58s ] thds: 10 tps: 77.70 qps: 1593.87 (r/w/o: 1116.06/322.41/155.40) lat (ms,95%): 189.93 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 9 tps: 82.10 qps: 1590.52 (r/w/o: 1113.27/314.54/162.71) lat (ms,95%): 193.38 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            68558
        write:                           19588
        other:                           9794
        total:                           97940
    transactions:                        4897   (81.50 per sec.)
    queries:                             97940  (1629.92 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0807s
    total number of events:              4897

Latency (ms):
         min:                                   45.64
         avg:                                  122.60
         max:                                  538.23
         95th percentile:                      189.93
         sum:                               600357.50

Threads fairness:
    events (avg/stddev):           489.7000/5.92
    execution time (avg/stddev):   60.0358/0.03

```

#### 4个线程读写(rw)

```bash
[openeuler@openeuler-riscv64 ~]$ sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=4 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run

sysbench: /home/openeuler/opengauss_install_c910/lib/libpq.so.5: no version information available (required by sysbench)
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 4
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 2s ] thds: 4 tps: 77.01 qps: 1562.65 (r/w/o: 1096.59/310.05/156.02) lat (ms,95%): 56.84 err/s: 0.00 reconn/s: 0.00
[ 4s ] thds: 4 tps: 78.16 qps: 1570.67 (r/w/o: 1099.72/314.64/156.32) lat (ms,95%): 70.55 err/s: 0.00 reconn/s: 0.00
[ 6s ] thds: 4 tps: 77.40 qps: 1542.04 (r/w/o: 1080.62/307.11/154.30) lat (ms,95%): 78.60 err/s: 0.00 reconn/s: 0.00
[ 8s ] thds: 4 tps: 74.57 qps: 1492.82 (r/w/o: 1044.42/298.76/149.63) lat (ms,95%): 95.81 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 4 tps: 82.98 qps: 1660.08 (r/w/o: 1161.20/333.42/165.46) lat (ms,95%): 56.84 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 4 tps: 76.03 qps: 1518.68 (r/w/o: 1061.98/304.14/152.57) lat (ms,95%): 87.56 err/s: 0.00 reconn/s: 0.00
[ 14s ] thds: 4 tps: 73.46 qps: 1469.24 (r/w/o: 1030.47/292.35/146.42) lat (ms,95%): 84.47 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 4 tps: 84.03 qps: 1682.16 (r/w/o: 1176.96/336.63/168.57) lat (ms,95%): 54.83 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 4 tps: 72.83 qps: 1450.53 (r/w/o: 1014.07/290.80/145.65) lat (ms,95%): 92.42 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 4 tps: 76.23 qps: 1540.70 (r/w/o: 1079.29/308.94/152.46) lat (ms,95%): 71.83 err/s: 0.00 reconn/s: 0.00
[ 22s ] thds: 4 tps: 81.44 qps: 1620.32 (r/w/o: 1134.67/322.77/162.88) lat (ms,95%): 62.19 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 4 tps: 73.52 qps: 1474.46 (r/w/o: 1033.33/294.09/147.05) lat (ms,95%): 92.42 err/s: 0.00 reconn/s: 0.00
[ 26s ] thds: 4 tps: 82.20 qps: 1644.54 (r/w/o: 1150.33/329.80/164.40) lat (ms,95%): 58.92 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 4 tps: 75.74 qps: 1502.79 (r/w/o: 1050.85/300.96/150.98) lat (ms,95%): 89.16 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 4 tps: 75.47 qps: 1498.43 (r/w/o: 1047.10/299.89/151.44) lat (ms,95%): 89.16 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 4 tps: 82.80 qps: 1675.90 (r/w/o: 1177.12/333.18/165.59) lat (ms,95%): 53.85 err/s: 0.00 reconn/s: 0.00
[ 34s ] thds: 4 tps: 71.20 qps: 1423.58 (r/w/o: 994.35/286.82/142.41) lat (ms,95%): 90.78 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 4 tps: 77.09 qps: 1537.35 (r/w/o: 1076.79/306.38/154.18) lat (ms,95%): 73.13 err/s: 0.00 reconn/s: 0.00
[ 38s ] thds: 4 tps: 84.35 qps: 1680.40 (r/w/o: 1176.33/335.38/168.69) lat (ms,95%): 52.89 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 4 tps: 68.37 qps: 1376.44 (r/w/o: 964.71/274.99/136.75) lat (ms,95%): 90.78 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 4 tps: 74.88 qps: 1489.57 (r/w/o: 1041.80/298.01/149.76) lat (ms,95%): 77.19 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 4 tps: 81.83 qps: 1637.09 (r/w/o: 1145.61/327.82/163.66) lat (ms,95%): 55.82 err/s: 0.00 reconn/s: 0.00
[ 46s ] thds: 4 tps: 76.96 qps: 1550.10 (r/w/o: 1086.87/309.32/153.91) lat (ms,95%): 90.78 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 4 tps: 74.37 qps: 1490.42 (r/w/o: 1043.20/298.48/148.74) lat (ms,95%): 78.60 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 4 tps: 84.39 qps: 1684.30 (r/w/o: 1178.96/336.56/168.78) lat (ms,95%): 53.85 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 4 tps: 76.33 qps: 1521.01 (r/w/o: 1063.05/305.31/152.65) lat (ms,95%): 71.83 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 4 tps: 74.85 qps: 1489.57 (r/w/o: 1042.45/297.41/149.71) lat (ms,95%): 86.00 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 4 tps: 83.61 qps: 1679.13 (r/w/o: 1175.99/335.93/167.21) lat (ms,95%): 54.83 err/s: 0.00 reconn/s: 0.00
[ 58s ] thds: 4 tps: 75.91 qps: 1527.21 (r/w/o: 1070.25/305.14/151.82) lat (ms,95%): 84.47 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 4 tps: 77.22 qps: 1549.82 (r/w/o: 1082.02/313.37/154.43) lat (ms,95%): 81.48 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            65170
        write:                           18620
        other:                           9310
        total:                           93100
    transactions:                        4655   (77.54 per sec.)
    queries:                             93100  (1550.78 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0257s
    total number of events:              4655

Latency (ms):
         min:                                   38.78
         avg:                                   51.56
         max:                                  174.02
         95th percentile:                       77.19
         sum:                               240027.63

Threads fairness:
    events (avg/stddev):           1163.7500/1.64
    execution time (avg/stddev):   60.0069/0.00

[openeuler@openeuler-riscv64 ~]$
```

### 只读性能

#### 10个线程只读(ro)

```bash
[openeuler@openeuler-riscv64 ~]$ sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/select.lua run
sysbench: /home/openeuler/opengauss_install_c910/lib/libpq.so.5: no version information available (required by sysbench)
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 10
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 2s ] thds: 10 tps: 1151.34 qps: 1151.34 (r/w/o: 1151.34/0.00/0.00) lat (ms,95%): 18.95 err/s: 0.00 reconn/s: 0.00
[ 4s ] thds: 10 tps: 1756.12 qps: 1756.12 (r/w/o: 1756.12/0.00/0.00) lat (ms,95%): 15.27 err/s: 0.00 reconn/s: 0.00
[ 6s ] thds: 10 tps: 1927.48 qps: 1927.48 (r/w/o: 1927.48/0.00/0.00) lat (ms,95%): 12.30 err/s: 0.00 reconn/s: 0.00
[ 8s ] thds: 10 tps: 1914.43 qps: 1914.43 (r/w/o: 1914.43/0.00/0.00) lat (ms,95%): 11.45 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 10 tps: 1959.54 qps: 1959.54 (r/w/o: 1959.54/0.00/0.00) lat (ms,95%): 11.04 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 10 tps: 1957.16 qps: 1957.16 (r/w/o: 1957.16/0.00/0.00) lat (ms,95%): 11.45 err/s: 0.00 reconn/s: 0.00
[ 14s ] thds: 10 tps: 1924.17 qps: 1924.17 (r/w/o: 1924.17/0.00/0.00) lat (ms,95%): 11.04 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 10 tps: 1999.74 qps: 1999.74 (r/w/o: 1999.74/0.00/0.00) lat (ms,95%): 10.84 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 10 tps: 2014.04 qps: 2014.04 (r/w/o: 2014.04/0.00/0.00) lat (ms,95%): 10.46 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 10 tps: 1997.20 qps: 1997.20 (r/w/o: 1997.20/0.00/0.00) lat (ms,95%): 10.27 err/s: 0.00 reconn/s: 0.00
[ 22s ] thds: 10 tps: 2034.59 qps: 2034.59 (r/w/o: 2034.59/0.00/0.00) lat (ms,95%): 9.91 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 10 tps: 2023.11 qps: 2023.11 (r/w/o: 2023.11/0.00/0.00) lat (ms,95%): 10.09 err/s: 0.00 reconn/s: 0.00
[ 26s ] thds: 10 tps: 2010.18 qps: 2010.18 (r/w/o: 2010.18/0.00/0.00) lat (ms,95%): 10.65 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 10 tps: 2069.40 qps: 2069.40 (r/w/o: 2069.40/0.00/0.00) lat (ms,95%): 10.09 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 10 tps: 2037.38 qps: 2037.38 (r/w/o: 2037.38/0.00/0.00) lat (ms,95%): 9.73 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 10 tps: 2032.39 qps: 2032.39 (r/w/o: 2032.39/0.00/0.00) lat (ms,95%): 9.73 err/s: 0.00 reconn/s: 0.00
[ 34s ] thds: 10 tps: 2072.57 qps: 2072.57 (r/w/o: 2072.57/0.00/0.00) lat (ms,95%): 9.39 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 10 tps: 2058.86 qps: 2058.86 (r/w/o: 2058.86/0.00/0.00) lat (ms,95%): 9.22 err/s: 0.00 reconn/s: 0.00
[ 38s ] thds: 10 tps: 2084.22 qps: 2084.22 (r/w/o: 2084.22/0.00/0.00) lat (ms,95%): 9.06 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 10 tps: 2095.90 qps: 2095.90 (r/w/o: 2095.90/0.00/0.00) lat (ms,95%): 9.56 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 10 tps: 2082.28 qps: 2082.28 (r/w/o: 2082.28/0.00/0.00) lat (ms,95%): 9.39 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 10 tps: 2109.23 qps: 2109.23 (r/w/o: 2109.23/0.00/0.00) lat (ms,95%): 8.90 err/s: 0.00 reconn/s: 0.00
[ 46s ] thds: 10 tps: 2062.67 qps: 2063.17 (r/w/o: 2063.17/0.00/0.00) lat (ms,95%): 9.06 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 10 tps: 2109.12 qps: 2108.61 (r/w/o: 2108.61/0.00/0.00) lat (ms,95%): 8.58 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 10 tps: 2108.46 qps: 2108.46 (r/w/o: 2108.46/0.00/0.00) lat (ms,95%): 8.43 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 10 tps: 2125.39 qps: 2125.39 (r/w/o: 2125.39/0.00/0.00) lat (ms,95%): 8.58 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 10 tps: 2110.21 qps: 2110.21 (r/w/o: 2110.21/0.00/0.00) lat (ms,95%): 8.74 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 10 tps: 2123.78 qps: 2123.78 (r/w/o: 2123.78/0.00/0.00) lat (ms,95%): 8.43 err/s: 0.00 reconn/s: 0.00
[ 58s ] thds: 10 tps: 2106.90 qps: 2106.90 (r/w/o: 2106.90/0.00/0.00) lat (ms,95%): 8.74 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            120395
        write:                           0
        other:                           0
        total:                           120395
    transactions:                        120395 (2005.36 per sec.)
    queries:                             120395 (2005.36 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0288s
    total number of events:              120395

Latency (ms):
         min:                                    1.32
         avg:                                    4.98
         max:                                  345.25
         95th percentile:                       10.09
         sum:                               598970.41

Threads fairness:
    events (avg/stddev):           12039.5000/159.11
    execution time (avg/stddev):   59.8970/0.01

```

#### 4个线程只读(ro)

```bash
[openeuler@openeuler-riscv64 ~]$ sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=4 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/select.lua run
sysbench: /home/openeuler/opengauss_install_c910/lib/libpq.so.5: no version information available (required by sysbench)
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 4
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 2s ] thds: 4 tps: 2119.22 qps: 2119.22 (r/w/o: 2119.22/0.00/0.00) lat (ms,95%): 2.03 err/s: 0.00 reconn/s: 0.00
[ 4s ] thds: 4 tps: 2259.08 qps: 2259.08 (r/w/o: 2259.08/0.00/0.00) lat (ms,95%): 1.93 err/s: 0.00 reconn/s: 0.00
[ 6s ] thds: 4 tps: 2297.63 qps: 2297.63 (r/w/o: 2297.63/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 8s ] thds: 4 tps: 2271.76 qps: 2271.76 (r/w/o: 2271.76/0.00/0.00) lat (ms,95%): 1.93 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 4 tps: 2308.72 qps: 2308.72 (r/w/o: 2308.72/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 4 tps: 2272.79 qps: 2272.79 (r/w/o: 2272.79/0.00/0.00) lat (ms,95%): 1.93 err/s: 0.00 reconn/s: 0.00
[ 14s ] thds: 4 tps: 2271.04 qps: 2271.04 (r/w/o: 2271.04/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 4 tps: 2287.75 qps: 2287.75 (r/w/o: 2287.75/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 4 tps: 2271.68 qps: 2271.68 (r/w/o: 2271.68/0.00/0.00) lat (ms,95%): 1.93 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 4 tps: 2286.10 qps: 2286.10 (r/w/o: 2286.10/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 22s ] thds: 4 tps: 2223.34 qps: 2223.34 (r/w/o: 2223.34/0.00/0.00) lat (ms,95%): 2.14 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 4 tps: 2285.72 qps: 2285.72 (r/w/o: 2285.72/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 26s ] thds: 4 tps: 2292.89 qps: 2292.89 (r/w/o: 2292.89/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 4 tps: 2204.84 qps: 2204.84 (r/w/o: 2204.84/0.00/0.00) lat (ms,95%): 2.14 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 4 tps: 2288.85 qps: 2288.85 (r/w/o: 2288.85/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 4 tps: 2287.01 qps: 2287.01 (r/w/o: 2287.01/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 34s ] thds: 4 tps: 2281.83 qps: 2281.83 (r/w/o: 2281.83/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 4 tps: 2314.26 qps: 2314.26 (r/w/o: 2314.26/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 38s ] thds: 4 tps: 2278.72 qps: 2278.72 (r/w/o: 2278.72/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 4 tps: 2302.14 qps: 2302.14 (r/w/o: 2302.14/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 4 tps: 2320.74 qps: 2320.74 (r/w/o: 2320.74/0.00/0.00) lat (ms,95%): 1.79 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 4 tps: 2271.12 qps: 2271.12 (r/w/o: 2271.12/0.00/0.00) lat (ms,95%): 1.93 err/s: 0.00 reconn/s: 0.00
[ 46s ] thds: 4 tps: 2292.55 qps: 2292.55 (r/w/o: 2292.55/0.00/0.00) lat (ms,95%): 1.82 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 4 tps: 2290.06 qps: 2290.06 (r/w/o: 2290.06/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 4 tps: 2283.71 qps: 2283.71 (r/w/o: 2283.71/0.00/0.00) lat (ms,95%): 1.89 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 4 tps: 2293.31 qps: 2293.31 (r/w/o: 2293.31/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 4 tps: 2306.51 qps: 2306.51 (r/w/o: 2306.51/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 4 tps: 2306.41 qps: 2306.41 (r/w/o: 2306.41/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
[ 58s ] thds: 4 tps: 2282.09 qps: 2282.09 (r/w/o: 2282.09/0.00/0.00) lat (ms,95%): 1.86 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            136711
        write:                           0
        other:                           0
        total:                           136711
    transactions:                        136711 (2277.73 per sec.)
    queries:                             136711 (2277.73 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0135s
    total number of events:              136711

Latency (ms):
         min:                                    1.31
         avg:                                    1.75
         max:                                  118.47
         95th percentile:                        1.89
         sum:                               238787.31

Threads fairness:
    events (avg/stddev):           34177.7500/47.56
    execution time (avg/stddev):   59.6968/0.00

```

## 性能测试总结

> rw: oltp 测试,包含读写 ro:select 测试,仅读

SQL statistics:

| Platform               | read   | write  | other | total  | transactions | transactions/s | queries | queries/s | ignored errors | reconnects |
|------------------------|--------|--------|-------|--------|--------------|----------------|---------|-----------|----------------|------------|
| TH1520 @ 10 Threads rw | 68558  | 19588  | 9794  | 97940  | 4897         | 81.50          | 97940   | 1629.92   | 0              | 0          |
| TH1520 @  4 Threads rw | 65170  | 18620  | 9310  | 93100  | 4655         | 77.54          | 93100   | 1550.78   | 0              | 0          |
| TH1520 @ 10 Threads ro | 120395 | 0      | 0     | 120395 | 120395       | 2005.36        | 120395  | 2005.35   | 0              | 0          |
| TH1520 @  4 Threads ro | 136711 | 0      | 0     | 136711 | 136711       | 2277.73        | 136711  | 2277.73   | 0              | 0          |

Latency (ms)

| Platform               | min   | avg    | max    | 95th percentile | sum        |
|------------------------|-------|--------|--------|-----------------|------------|
| TH1520 @ 10 Threads rw | 45.64 | 122.60 | 538.23 | 189.93          | 600357.50  |
| TH1520 @  4 Threads rw | 38.78 | 51.56  | 174.02 | 77.19           | 240027.63  |
| TH1520 @ 10 Threads ro | 1.32  | 4.98   | 345.25 | 10.09           | 598970.41  |
| TH1520 @  4 Threads ro | 1.31  | 1.75   | 118.47 | 1.89            | 238787.31  |

Threads fairness

| Platform               | events avg | events stddev | execution time avg | execution time stddev |
|------------------------|------------|---------------|--------------------|-----------------------|
| TH1520 @ 10 Threads rw | 489.7000   | 5.92          | 60.0358            | 0.03                  |
| TH1520 @  4 Threads rw | 1163.7500  | 1.64          | 60.0069            | 0.00                  |
| TH1520 @ 10 Threads ro | 12039.5000 | 159.11        | 59.8970            | 0.01                  |
| TH1520 @  4 Threads ro | 34177.7500 | 47.56         | 59.6968            | 0.00                  |
