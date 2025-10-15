# openGauss 数据库 (7.0.0 全量版) SG2042 测试报告

## 测试环境

- 硬件：SG2042 SoC (Milk-V Pioneer 开发板)
- 操作系统（容器内）：openEuler 24.03

## 安装
获取源码并运行 openEuler 容器：
```bash
git clone https://gitcode.com/MengMengDeXiaoJi/riscv.git
docker run -it --name oerv-opengauss-build -v$(pwd):/root/rpmbuild/SOURCES xfan1024/openeuler:24.03-riscv64
```

在容器内执行:
```bash
cd /root/rpmbuild/SOURCES
git checkout upgrade-7.0.0-full
dnf install -y wget rpm-build rpmdevtools dnf-plugins-core
yum-builddep -y opengauss-server.spec
spectool -g opengauss-server.spec

# 如未安装 git-lfs
wget https://github.com/git-lfs/git-lfs/releases/download/v3.7.0/git-lfs-linux-riscv64-v3.7.0.tar.gz
tar -xvf git-lfs-linux-riscv64-v3.7.0.tar.gz
./git-lfs-3.7.0/install.sh

git lfs pull
rpmbuild -ba opengauss-server.spec
dnf install -y /root/rpmbuild/RPMS/riscv64/opengauss-server-7.0.0-1.riscv64.rpm
```

## 启动、数据库初始化

启动 openGauss:
```bash
su opengauss
sh /opt/opengauss/init-opengaussdb.sh
/opt/opengauss/bin/gs_ctl start -D /var/lib/opengauss/data
```

进入控制台，创建新数据库和用户：

```bash
[opengauss@b170b76b0685 SOURCES]$ gsql -d postgres

openGauss=# CREATE USER testuser WITH PASSWORD 'openEuler12#$';
openGauss=# CREATE DATABASE testdb owner testuser;
openGauss=# GRANT ALL ON SCHEMA public TO testuser;
GRANT
openGauss=# GRANT ALL PRIVILEGES TO testuser;
ALTER ROLE
```
按 Ctrl-D 退出。

## 本地测试
参见 https://gitcode.com/MengMengDeXiaoJi/riscv/blob/upgrade-7.0.0-full/installation.md， 经测试可用：

```log
...
insert into phonebook (name, phone) values ('邮政快递', '11183');
insert into phonebook (name, phone) values ('顺丰快递', '95338');
insert into phonebook (name, phone) values ('京东物流', '95311');

select * from phonebook where name like '%银行';
select count(*) from phonebook;
delete from phonebook where name = '农业银行';
select * from phonebook;
testdb(> testdb(> testdb(> testdb(> NOTICE:  CREATE TABLE will create implicit sequence "phonebook_id_seq" for serial column "phonebook.id"
NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "phonebook_pkey" for table "phonebook"
CREATE TABLE
testdb=> testdb=> INSERT 0 1
testdb=> INSERT 0 1
testdb=> INSERT 0 1
testdb=> testdb=> INSERT 0 1
testdb=> INSERT 0 1
testdb=> INSERT 0 1
testdb=> testdb=>  id |     name     | phone
----+--------------+-------
  1 | 工商银行 | 95588
  2 | 招商银行 | 95555
  3 | 农业银行 | 95599
(3 rows)

testdb=>  count
-------
     6
(1 row)

testdb=> DELETE 1
testdb=>  id |     name     | phone
----+--------------+-------
  1 | 工商银行 | 95588
  2 | 招商银行 | 95555
  4 | 邮政快递 | 11183
  5 | 顺丰快递 | 95338
  6 | 京东物流 | 95311
(5 rows)

testdb=>

```

## 性能测试

### 准备性能测试环境

按照sysbench

```bash
sudo dnf install sysbench
```

修改 openGauss 配置文件：

```bash
vim gauss_data/postgresql.conf
# 配置 listen_addresses = '*'
# 配置 password_encryption_type = 1

/opt/opengauss/bin/gs_ctl reload -D /var/lib/opengauss/data
```

## 运行测试

初始化数据库：

```bash
sysbench --db-driver=pgsql --oltp-table-size=100000 --oltp-tables-count=24 --threads=1 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb  /usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua run
```

使用 `gsql -U testuser -d testdb` 验证生成的数据：

```bash
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

testdb=>
```

执行读/写测试：

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

数据库初始化成功：
```log
SQL statistics:
    queries performed:
        read:                            0
        write:                           912
        other:                           48
        total:                           960
    transactions:                        1      (0.00 per sec.)
    queries:                             960    (2.64 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          363.5184s
    total number of events:              1

Latency (ms):
         min:                               363515.88
         avg:                               363515.88
         max:                               363515.88
         95th percentile:                   100000.00
         sum:                               363515.88

Threads fairness:
    events (avg/stddev):           1.0000/0.00
    execution time (avg/stddev):   363.5159/0.00
```

但初始化后无法运行任何测试，直接出现 `Segmentation fault`:

```log
[opengauss@b170b76b0685 SOURCES]$ sysbench --db-driver=pgsql --report-interval=2 --oltp-table-size=100000 --oltp-tables-count=24 --threads=10 --time=60 --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-user=testuser --pgsql-password=openEuler12#$ --pgsql-db=testdb /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run
sysbench 1.0.20 (using system LuaJIT 2.1.ROLLING)

Running the test with following options:
Number of threads: 10
Report intermediate results every 2 second(s)
Initializing random number generator from current time


Initializing worker threads...

Segmentation fault (core dumped)

```