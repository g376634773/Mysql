# Mysql Group Replication
参考文档：https://www.cnblogs.com/hxlasky/p/11453885.html
1. 部署
下载mysql二进制包或者rpm包，我这里下载的是二进制包
https://downloads.mysql.com/archives/community/ 
选择Compressed TAR Archive，我的版本是5.7.33

tar -xvf mysql-5.7.33-el7-x86_64.tar.gz -C /data/
mv mysql-5.7.33-el7-x86_64 mysql-3307 因为我的环境中还有其他的mysql，这里为了区分所以把目录命名为mysql-{port}

cd mysql-3307 && mkdir conf data   创建两个目录

编辑my.cnf配置文件，改配置文件是实际生成环境的，其实你只需要关注开始的ip和mysql的存放路径，还有就是group replication下的内容即可，其它部分可以不用修改。

vim conf/my.cnf

[mysqld]

default-time-zone                         = '+08:00'
#### report-host                             = 192.168.1.100

port                                    = 3307
user                                    = mysql

basedir                                 = /data/mysql-3307
datadir                                 = /data/mysql-3307/data
tmpdir                                  = /data/mysql-3307/
slave-load-tmpdir                       = /data/mysql-3307

pid-file                                = /var/run/mysqld/mysqld-3307.pid
socket                                  = /var/run/mysqld/mysqld-3307.sock

server-id                               = 1
log-error                               = /data/mysql-3307/error.log


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


loose-group_replication_start_on_boot = "off"

loose-group_replication_local_address         = "192.168.1.100:4307"
loose-group_replication_group_seeds           = "192.168.1.101:4307,192.168.1.100:4307,192.168.1.102:4307"


loose-group_replication_bootstrap_group       = off


slave-preserve-commit-order                   = 1

loose-group-replication-single-primary-mode   = 0

loose-group_replication_ip_whitelist          = "127.0.0.1/8,192.168.1.101,192.168.1.100,192.168.1.102"

super-read-only                               = "off"




[client]

port                                    = 3307
socket                                  = /var/run/mysqld/mysqld-3307.sock
no-auto-rehash
character-set-client                    = utf8


[myisamchk]

key-buffer                              = 64M
sort-buffer-size                        = 32M
read-buffer                             = 16M
write-buffer                            = 16M
