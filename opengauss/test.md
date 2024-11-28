# 测试 openGauss 数据库

## 功能测试

### 本地测试


### 远程测试


## 性能测试

### sysbench

首先，在 PostgreSQL 中创建数据库和用户：
```
su - postgres

gsql -d postgres

CREATE USER testuser WITH PASSWORD 'openEuler12#$';

CREATE DATABASE testdb owner testuser;
```

授予权限用于测试
```log
[opengauss@openeuler-riscv64 openeuler]$ gsql -d postgres
gsql ((openGauss-lite 6.0.0 build ) compiled at 2024-11-22 20:54:23 commit 0 last mr  release)
Non-SSL connection (SSL connection is recommended when requiring high-security)
Type "help" for help.

openGauss=# GRANT ALL ON SCHEMA public TO testuser;
GRANT
openGauss=# GRANT ALL PRIVILEGES TO testuser; 
ALTER ROLE
```

初始化数据库
```
sysbench --db-driver=pgsql --oltp-table-size=100000 --oltp-tables-count=24 --threads=1 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb  /usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua run
```

使用下列命令验证生成的数据

```log
[opengauss@openeuler-riscv64 openeuler]$ gsql -U testuser -d testdb
Password for user testuser: 
gsql ((openGauss-lite 6.0.0 build ) compiled at 2024-11-22 20:54:23 commit 0 last mr  release)
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
```

执行读/写测试

```
sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=64 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run
```
上述命令将从名为 /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua 的 LUA 脚本生成 OLTP 工作负载，针对主服务器上 24 个表的 100,000 行（具有 64 个工作线程）持续 60 秒）。每 2 秒，sysbench 将报告中间统计信息（–report-interval=2）。

#### 测试结果

详细结果参见 [logs](./logs) 目录。


#### Raw data / log

<details><summary>Click to expand</summary>

SG2042 openEuler 2403 10线程

```log
[openeuler@openeuler-riscv64 ~]$ sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ -
-pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 10
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 2s ] thds: 10 tps: 309.76 qps: 6252.70 (r/w/o: 4385.71/1242.51/624.48) lat (ms,95%): 34.95 err/s: 0.00 reconn/s: 0.00
[ 4s ] thds: 10 tps: 332.75 qps: 6657.07 (r/w/o: 4660.05/1331.51/665.51) lat (ms,95%): 34.33 err/s: 0.00 reconn/s: 0.00
[ 6s ] thds: 10 tps: 324.53 qps: 6499.64 (r/w/o: 4552.45/1297.13/650.06) lat (ms,95%): 35.59 err/s: 0.50 reconn/s: 0.00
[ 8s ] thds: 10 tps: 332.51 qps: 6644.63 (r/w/o: 4649.59/1331.03/664.01) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 10 tps: 332.99 qps: 6654.26 (r/w/o: 4656.33/1330.95/666.98) lat (ms,95%): 33.72 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 10 tps: 324.51 qps: 6499.60 (r/w/o: 4552.57/1298.02/649.01) lat (ms,95%): 34.33 err/s: 0.00 reconn/s: 0.00
[ 14s ] thds: 10 tps: 333.51 qps: 6668.63 (r/w/o: 4667.09/1334.53/667.01) lat (ms,95%): 33.72 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 10 tps: 324.82 qps: 6477.44 (r/w/o: 4530.01/1297.79/649.64) lat (ms,95%): 34.33 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 10 tps: 328.18 qps: 6561.13 (r/w/o: 4593.54/1311.73/655.86) lat (ms,95%): 36.89 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 10 tps: 335.49 qps: 6735.39 (r/w/o: 4718.92/1344.98/671.49) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 22s ] thds: 10 tps: 331.51 qps: 6628.14 (r/w/o: 4641.60/1323.53/663.01) lat (ms,95%): 34.33 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 10 tps: 336.49 qps: 6707.86 (r/w/o: 4689.40/1345.47/672.99) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 26s ] thds: 10 tps: 338.87 qps: 6789.99 (r/w/o: 4755.24/1357.00/677.75) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 10 tps: 333.13 qps: 6652.01 (r/w/o: 4652.76/1333.00/666.25) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 10 tps: 326.43 qps: 6546.03 (r/w/o: 4585.47/1307.71/652.85) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 10 tps: 329.41 qps: 6575.12 (r/w/o: 4599.18/1317.12/658.81) lat (ms,95%): 33.72 err/s: 0.00 reconn/s: 0.00
[ 34s ] thds: 10 tps: 339.17 qps: 6785.93 (r/w/o: 4754.40/1353.18/678.34) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 10 tps: 333.20 qps: 6674.90 (r/w/o: 4672.23/1336.27/666.40) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 38s ] thds: 10 tps: 337.83 qps: 6747.99 (r/w/o: 4721.02/1352.32/674.65) lat (ms,95%): 32.53 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 10 tps: 335.00 qps: 6709.08 (r/w/o: 4702.56/1335.52/671.01) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 10 tps: 334.88 qps: 6691.17 (r/w/o: 4681.37/1340.03/669.77) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 10 tps: 331.59 qps: 6605.82 (r/w/o: 4616.77/1325.86/663.18) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 46s ] thds: 10 tps: 330.48 qps: 6638.00 (r/w/o: 4651.65/1325.90/660.45) lat (ms,95%): 36.24 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 10 tps: 336.55 qps: 6744.42 (r/w/o: 4725.14/1345.68/673.59) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 10 tps: 337.01 qps: 6728.15 (r/w/o: 4707.10/1347.03/674.01) lat (ms,95%): 32.53 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 10 tps: 332.57 qps: 6646.48 (r/w/o: 4651.03/1330.29/665.15) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 10 tps: 333.61 qps: 6683.78 (r/w/o: 4680.59/1335.95/667.23) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 10 tps: 326.45 qps: 6505.01 (r/w/o: 4546.31/1305.80/652.90) lat (ms,95%): 34.95 err/s: 0.00 reconn/s: 0.00
[ 58s ] thds: 10 tps: 333.98 qps: 6690.57 (r/w/o: 4688.70/1333.91/667.96) lat (ms,95%): 31.94 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 10 tps: 332.89 qps: 6658.81 (r/w/o: 4660.46/1333.06/665.28) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            278796
        write:                           79654
        other:                           39828
        total:                           398278
    transactions:                        19913  (331.56 per sec.)
    queries:                             398278 (6631.51 per sec.)
    ignored errors:                      1      (0.02 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0473s
    total number of events:              19913

Latency (ms):
         min:                                   25.62
         avg:                                   30.13
         max:                                   99.91
         95th percentile:                       33.72
         sum:                               599938.70

Threads fairness:
    events (avg/stddev):           1991.3000/32.68
    execution time (avg/stddev):   59.9939/0.01
```

SG2042 openEuler 2403 64线程

```log
[openeuler@openeuler-riscv64 ~]$ sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=64 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/shar
e/sysbench/tests/include/oltp_legacy/oltp.lua run
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 64
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 2s ] thds: 64 tps: 908.23 qps: 18469.12 (r/w/o: 12975.05/3646.67/1847.40) lat (ms,95%): 77.19 err/s: 0.49 reconn/s: 0.00
[ 4s ] thds: 64 tps: 1133.28 qps: 22709.17 (r/w/o: 15898.34/4542.24/2268.59) lat (ms,95%): 70.55 err/s: 0.51 reconn/s: 0.00
[ 6s ] thds: 64 tps: 1175.54 qps: 23535.38 (r/w/o: 16473.12/4709.18/2353.09) lat (ms,95%): 63.32 err/s: 0.50 reconn/s: 0.00
[ 8s ] thds: 64 tps: 1093.53 qps: 21849.17 (r/w/o: 15298.47/4363.14/2187.57) lat (ms,95%): 77.19 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 64 tps: 1067.83 qps: 21403.96 (r/w/o: 14993.02/4273.31/2137.64) lat (ms,95%): 81.48 err/s: 0.49 reconn/s: 0.00
[ 12s ] thds: 64 tps: 913.50 qps: 18252.31 (r/w/o: 12764.69/3660.61/1827.01) lat (ms,95%): 287.38 err/s: 0.51 reconn/s: 0.00
[ 14s ] thds: 64 tps: 1138.09 qps: 22758.33 (r/w/o: 15933.28/4546.87/2278.18) lat (ms,95%): 70.55 err/s: 0.50 reconn/s: 0.00
[ 16s ] thds: 64 tps: 1183.26 qps: 23643.64 (r/w/o: 16549.10/4729.03/2365.52) lat (ms,95%): 64.47 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 64 tps: 1135.01 qps: 22797.18 (r/w/o: 15971.12/4554.54/2271.53) lat (ms,95%): 70.55 err/s: 0.50 reconn/s: 0.00
[ 20s ] thds: 64 tps: 1157.61 qps: 23076.14 (r/w/o: 16140.99/4620.43/2314.72) lat (ms,95%): 69.29 err/s: 0.00 reconn/s: 0.00
[ 22s ] thds: 64 tps: 1112.28 qps: 22233.18 (r/w/o: 15561.48/4446.64/2225.06) lat (ms,95%): 74.46 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 64 tps: 1151.87 qps: 23063.98 (r/w/o: 16148.74/4609.49/2305.75) lat (ms,95%): 70.55 err/s: 0.50 reconn/s: 0.00
[ 26s ] thds: 64 tps: 1147.43 qps: 22931.02 (r/w/o: 16049.95/4587.21/2293.85) lat (ms,95%): 69.29 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 64 tps: 1145.66 qps: 22909.15 (r/w/o: 16040.21/4577.13/2291.82) lat (ms,95%): 70.55 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 64 tps: 1176.85 qps: 23516.09 (r/w/o: 16454.97/4705.91/2355.20) lat (ms,95%): 65.65 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 64 tps: 1138.65 qps: 22778.07 (r/w/o: 15942.65/4557.11/2278.31) lat (ms,95%): 71.83 err/s: 0.50 reconn/s: 0.00
[ 34s ] thds: 64 tps: 1131.28 qps: 22671.64 (r/w/o: 15874.95/4533.63/2263.06) lat (ms,95%): 70.55 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 64 tps: 1139.27 qps: 22804.42 (r/w/o: 15974.31/4552.07/2278.04) lat (ms,95%): 74.46 err/s: 0.00 reconn/s: 0.00
[ 38s ] thds: 64 tps: 1179.87 qps: 23597.88 (r/w/o: 16508.17/4727.97/2361.74) lat (ms,95%): 65.65 err/s: 0.50 reconn/s: 0.00
[ 40s ] thds: 64 tps: 1127.22 qps: 22572.91 (r/w/o: 15799.59/4518.88/2254.44) lat (ms,95%): 75.82 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 64 tps: 1179.72 qps: 23602.94 (r/w/o: 16530.10/4713.90/2358.95) lat (ms,95%): 65.65 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 64 tps: 1146.00 qps: 22918.55 (r/w/o: 16038.03/4587.01/2293.51) lat (ms,95%): 70.55 err/s: 0.50 reconn/s: 0.00
[ 46s ] thds: 64 tps: 1157.01 qps: 23117.66 (r/w/o: 16189.10/4614.04/2314.51) lat (ms,95%): 68.05 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 64 tps: 1205.76 qps: 24106.18 (r/w/o: 16869.63/4825.54/2411.02) lat (ms,95%): 63.32 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 64 tps: 1138.44 qps: 22797.83 (r/w/o: 15967.67/4552.77/2277.39) lat (ms,95%): 69.29 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 64 tps: 1140.39 qps: 22755.29 (r/w/o: 15918.43/4554.06/2282.79) lat (ms,95%): 71.83 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 64 tps: 1139.00 qps: 22813.16 (r/w/o: 15974.22/4562.43/2276.51) lat (ms,95%): 69.29 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 64 tps: 1196.97 qps: 23889.99 (r/w/o: 16713.71/4780.32/2395.96) lat (ms,95%): 65.65 err/s: 0.00 reconn/s: 0.00
[ 58s ] thds: 64 tps: 1139.29 qps: 22812.75 (r/w/o: 15972.02/4563.15/2277.57) lat (ms,95%): 70.55 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 64 tps: 1167.62 qps: 23345.45 (r/w/o: 16338.22/4669.99/2337.24) lat (ms,95%): 68.05 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            952280
        write:                           272041
        other:                           136057
        total:                           1360378
    transactions:                        68009  (1128.35 per sec.)
    queries:                             1360378 (22570.22 per sec.)
    ignored errors:                      11     (0.18 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.2661s
    total number of events:              68009

Latency (ms):
         min:                                   38.63
         avg:                                   56.49
         max:                                  421.75
         95th percentile:                       70.55
         sum:                              3842023.49

Threads fairness:
    events (avg/stddev):           1062.6406/24.58
    execution time (avg/stddev):   60.0316/0.03
```

SG2042 openEuler 2403 64 线程 仅读

```log
[openeuler@openeuler-riscv64 ~]$  sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=64 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/sha
re/sysbench/tests/include/oltp_legacy/select.lua run
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 64
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 2s ] thds: 64 tps: 26520.23 qps: 26520.72 (r/w/o: 26520.72/0.00/0.00) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 4s ] thds: 64 tps: 30936.92 qps: 30936.42 (r/w/o: 30936.42/0.00/0.00) lat (ms,95%): 3.19 err/s: 0.00 reconn/s: 0.00
[ 6s ] thds: 64 tps: 30838.62 qps: 30839.12 (r/w/o: 30839.12/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 8s ] thds: 64 tps: 30584.49 qps: 30584.49 (r/w/o: 30584.49/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 64 tps: 28539.22 qps: 28539.22 (r/w/o: 28539.22/0.00/0.00) lat (ms,95%): 3.49 err/s: 0.00 reconn/s: 0.00
[ 12s ] thds: 64 tps: 30937.69 qps: 30937.69 (r/w/o: 30937.69/0.00/0.00) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 14s ] thds: 64 tps: 31212.48 qps: 31212.48 (r/w/o: 31212.48/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 16s ] thds: 64 tps: 31466.14 qps: 31466.14 (r/w/o: 31466.14/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 18s ] thds: 64 tps: 30756.22 qps: 30755.73 (r/w/o: 30755.73/0.00/0.00) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 64 tps: 32332.23 qps: 32332.23 (r/w/o: 32332.23/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 22s ] thds: 64 tps: 32277.47 qps: 32278.48 (r/w/o: 32278.48/0.00/0.00) lat (ms,95%): 3.19 err/s: 0.00 reconn/s: 0.00
[ 24s ] thds: 64 tps: 31847.74 qps: 31846.74 (r/w/o: 31846.74/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 26s ] thds: 64 tps: 31205.84 qps: 31206.34 (r/w/o: 31206.34/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 28s ] thds: 64 tps: 30860.64 qps: 30860.14 (r/w/o: 30860.14/0.00/0.00) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 64 tps: 31713.07 qps: 31713.57 (r/w/o: 31713.57/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 32s ] thds: 64 tps: 31504.72 qps: 31504.22 (r/w/o: 31504.22/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 34s ] thds: 64 tps: 31218.52 qps: 31218.52 (r/w/o: 31218.52/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 36s ] thds: 64 tps: 31555.87 qps: 31555.87 (r/w/o: 31555.87/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 38s ] thds: 64 tps: 31282.89 qps: 31282.89 (r/w/o: 31282.89/0.00/0.00) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 64 tps: 31200.90 qps: 31201.40 (r/w/o: 31201.40/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 42s ] thds: 64 tps: 31218.27 qps: 31218.77 (r/w/o: 31218.77/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 44s ] thds: 64 tps: 30811.74 qps: 30810.74 (r/w/o: 30810.74/0.00/0.00) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 46s ] thds: 64 tps: 30317.17 qps: 30317.17 (r/w/o: 30317.17/0.00/0.00) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 48s ] thds: 64 tps: 30283.46 qps: 30283.46 (r/w/o: 30283.46/0.00/0.00) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 64 tps: 30143.05 qps: 30143.05 (r/w/o: 30143.05/0.00/0.00) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 52s ] thds: 64 tps: 30486.58 qps: 30486.58 (r/w/o: 30486.58/0.00/0.00) lat (ms,95%): 3.36 err/s: 0.00 reconn/s: 0.00
[ 54s ] thds: 64 tps: 31213.42 qps: 31213.42 (r/w/o: 31213.42/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 56s ] thds: 64 tps: 31230.31 qps: 31230.31 (r/w/o: 31230.31/0.00/0.00) lat (ms,95%): 3.25 err/s: 0.00 reconn/s: 0.00
[ 58s ] thds: 64 tps: 30470.09 qps: 30470.59 (r/w/o: 30470.59/0.00/0.00) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 64 tps: 30453.40 qps: 30452.90 (r/w/o: 30452.90/0.00/0.00) lat (ms,95%): 3.30 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            1851630
        write:                           0
        other:                           0
        total:                           1851630
    transactions:                        1851630 (30766.50 per sec.)
    queries:                             1851630 (30766.50 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.1761s
    total number of events:              1851630

Latency (ms):
         min:                                    1.12
         avg:                                    2.06
         max:                                  353.15
         95th percentile:                        3.30
         sum:                              3822093.08

Threads fairness:
    events (avg/stddev):           28931.7188/1217.10
    execution time (avg/stddev):   59.7202/0.03
```

</details>


