# Mysql Group Replication   多主模式
参考文档：https://www.cnblogs.com/hxlasky/p/11453885.html

服务器三台：

192.168.1.100

192.168.1.101

192.168.1.102

1. 部署
```
下载mysql二进制包或者rpm包，我这里下载的是二进制包
https://downloads.mysql.com/archives/community/ 
选择Compressed TAR Archive，我的版本是5.7.33

tar -xvf mysql-5.7.33-el7-x86_64.tar.gz -C /data/
mv mysql-5.7.33-el7-x86_64 mysql-3307 因为我的环境中还有其他的mysql，这里为了区分所以把目录命名为mysql-{port}

cd mysql-3307 && mkdir conf data   创建两个目录

编辑my.cnf配置文件，改配置文件是实际生成环境的，其实你只需要关注开始的ip和mysql的存放路径，还有就是group replication下的内容即可，其它部分可以不用修改。
需要修改的我已经加粗。

vim conf/my.cnf

[mysqld]

default-time-zone                         = '+08:00'
#### report-host                             = 192.168.1.100

#### port                                    = 3307
#### user                                    = mysql

#### basedir                                 = /data/mysql-3307
#### datadir                                 = /data/mysql-3307/data
#### tmpdir                                  = /data/mysql-3307/
#### slave-load-tmpdir                       = /data/mysql-3307

#### pid-file                                = /var/run/mysqld/mysqld-3307.pid
#### socket                                  = /var/run/mysqld/mysqld-3307.sock

#### server-id                               = 1   #不同的节点需要这只不同的server-id，1-3
#### log-error                               = /data/mysql-3307/error.log


max-allowed-packet                      = 16M
table-open-cache                        = 4096
join-buffer-size                        = 16M
sort-buffer-size                        = 16M


read-buffer-size                        = 16M


read-rnd-buffer-size                    = 32M

query-cache-type                        = 0
query-cache-size                        = 0
max-tmp-tables                          = 256
tmp-table-size                          = 128M
max-heap-table-size                     = 128M
thread-cache-size                       = 64
max-connections                         = 10240
max-user-connections                    = 10240
max-connect-errors                      = 99999999
long-query-time                         = 1
slow-query-log                          = 1
slow-query-log-file                     = slow.log
myisam-repair-threads                   = 1
interactive-timeout                     = 3600
wait-timeout                            = 3600

read-only                               = 0
skip-name-resolve
character-set-server                    = utf8

auto-increment-increment                = 100
auto-increment-offset                   = 1

default-storage-engine                  = InnoDB
transaction-isolation                   = READ-COMMITTED

optimizer-switch                        = 'batched_key_access=off,block_nested_loop=on,condition_fanout_filter=on,derived_merge=on,duplicateweedout=on,engine_condition_pushdown=on,firstmatch=on,index_condition_pushdown=off,index_merge=on,index_merge_intersection=on,index_merge_sort_union=on,index_merge_union=on,loosescan=on,materialization=on,mrr=off,mrr_cost_based=on,semijoin=on,subquery_materialization_cost_based=on,use_index_extensions=on'


-------------  innodb  --------------


innodb-data-file-path                   = ibdata1:12M:autoextend
innodb-buffer-pool-size                 = 6G


innodb-buffer-pool-instances            = 8
innodb-log-file-size                    = 256M

innodb-flush-log-at-trx-commit          = 0
innodb-log-buffer-size                  = 8M
innodb-log-files-in-group               = 3
innodb-max-dirty-pages-pct              = 90
innodb-lock-wait-timeout                = 20
innodb-file-per-table                   = 1
innodb-flush-method                     = O_DIRECT
innodb-io-capacity                      = 2000
innodb-support-xa                       = 1

-------------  myisam  --------------


key-buffer-size                         = 1M
myisam-sort-buffer-size                 = 128M


-------------  binlog  --------------

log-bin                                 = mysql-bin
relay-log                               = relay-bin
binlog-format                           = ROW

max-relay-log-size                      = 1G
relay-log-space-limit                   = 64G
relay-log-purge                         = 1
log-slave-updates                       = 1

slave-parallel-workers                  = 0
slave-parallel-type                     = DATABASE


binlog-group-commit-sync-delay          = 0

gtid-mode                               = ON
enforce-gtid-consistency                = 1
binlog-checksum                         = NONE


skip-slave-start                        = 1


master-info-repository                  = TABLE
relay-log-info-repository               = TABLE


expire-logs-days                        = 15


-------------  group replication  --------------

transaction-write-set-extraction        = XXHASH64

loose-group_replication_group_name            = "e1f5b6f9-6095-409b-b872-5fe8f8c757d7"


#### loose-group_replication_start_on_boot = "on"  #组复制是否开机自启，最好设置成on

#### loose-group_replication_local_address         = "192.168.1.100:4307"  #自定义内部通信的端口
#### loose-group_replication_group_seeds           = "192.168.1.101:4307,192.168.1.100:4307,192.168.1.102:4307" #数据同步时的数据供应节点，一般将集群内的全部节点写入


loose-group_replication_bootstrap_group       = off  #是否开启引导，一般设置成off,在初始化集群的时候会在引导节点通过sql开启引导


slave-preserve-commit-order                   = 1

loose-group-replication-single-primary-mode   = 0

#### loose-group_replication_ip_whitelist          = "127.0.0.1/8,192.168.1.101,192.168.1.100,192.168.1.102" #组成员

super-read-only                               = "off"




[client]

#### port                                    = 3307
#### socket                                  = /var/run/mysqld/mysqld-3307.sock
no-auto-rehash
character-set-client                    = utf8


[myisamchk]

key-buffer                              = 64M
sort-buffer-size                        = 32M
read-buffer                             = 16M
write-buffer                            = 16M
```
2.初始化mysql并启动
```
chown -R mysql:mysql /data/mysql-3307

/data/mysql-3307/bin/mysqld --initialize-insecure --basedir=/data/mysql-3307 --datadir=/data/mysql-3307/data --user=mysql #初始化mysql

/data/mysql-3307/bin/mysqld --defaults-file=/data/mysql-3307/conf/my.cnf --daemonize --pid-file=/var/run/mysqld/mysqld-3307.pid #启动mysql
```
3.在其他节点部署mysql，并将my.cnf复制到其他节点并修改

4.连接mysql
```
mysql -uroot  -p --socket=/var/run/mysqld/mysqld-3307.sock
```
5.安装MGR插件(所有节点执行)
```
mysql> install plugin group_replication soname 'group_replication.so';

Query OK, 0 rows affected (0.04 sec)
```
6.设置复制账号(所有节点执行)
```
mysql> CREATE USER repl@'%' IDENTIFIED BY 'repl';

mysql> GRANT REPLICATION SLAVE ON *.* TO repl@'%';

mysql> FLUSH PRIVILEGES;

mysql> CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='repl' FOR CHANNEL 'group_replication_recovery';
```
7.开启组复制
```
#### 选择一个节点作为引导组的引导节点。所谓引导组，就是创建组。组创建之后，其他节点才能加入到组中。开启引导功能后，一般会立即开启该节点的组复制功能来创建组，然后立即关闭组引导功能。所以，在第一个节点上，这3个语句常放在一起执行：

mysql> set global group_replication_bootstrap_group=on;

mysql> start group_replication;

mysql> set global group_replication_bootstrap_group=off;

#### 其他节点执行
mysql> start group_replication;
```
8.查看主节点，如果值为空表明开启的是多主模式
```
select * from performance_schema.global_status where variable_name='group_replication_primary_member';

mysql> select * from performance_schema.global_status where variable_name='group_replication_primary_member';
+----------------------------------+----------------+
| VARIABLE_NAME                    | VARIABLE_VALUE |
+----------------------------------+----------------+
| group_replication_primary_member |                |
+----------------------------------+----------------+
1 row in set (0.01 sec)
```
9.查看mgr组复制信息
```
SELECT * FROM performance_schema.replication_group_members;
mysql>  SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | 726f6635-d748-11eb-be3d-52540097b3e7 | 172.18.2.71 |        3306 | ONLINE       |
| group_replication_applier | 727efe90-d748-11eb-be4d-525400510922 | 172.18.2.70 |        3306 | ONLINE       |
| group_replication_applier | 73986837-d748-11eb-bcbe-5254007c26b8 | 172.18.2.72 |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
```
 
