---
title: redis集群安装
date: 2017-12-29 10:00:28
tags: redis
categories: redis
---

# redis Cluster 集群安装

## 集群部署

测试部署方式，一台测试机多实例启动部署。

安装redis

```
$ wget http://download.redis.io/releases/redis-3.2.8.tar.gz
$ tar xzf redis-3.2.8.tar.gz
$ cd redis-3.2.8
$ yum groupinstall -y "Development Tools"
$ make && make install
```

修改配置文件 redis.conf

```yaml
#redis.conf默认配置
daemonize yes   #后台运行开启
pidfile /var/run/redis/redis.pid  #多实例情况下需修改，指定pid文件的路径 通过绝对路径指明文件存放的位置 自行创建相关的文件目录 例如redis_6380.pid
port 6379　　　　　　　　#多实例情况下需要修改,修改端口号 例如6380
tcp-backlog 511
bind 0.0.0.0　　　　　　
timeout 0
tcp-keepalive 0
loglevel notice
logfile /var/log/redis/redis.log　　　　　　#多实例情况下需要修改，例如6380.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb　　#多实例情况下需要修改，修改dump日志文件路径 果不修改dump文件那么每次的日志文件都是公用的 例如dump.6380.rdb
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly yes  #启用二进制日志
appendfilename "appendonly.aof"　　#多实例情况下需要修改,例如 appendonly_6380.aof
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10

#################自定义配置
#系统配置
#vim /etc/sysctl.conf
#vm.overcommit_memory = 1

aof-rewrite-incremental-fsync yes
maxmemory 4096mb
maxmemory-policy allkeys-lru
dir /opt/redis/data　　　　　　#多实例情况下需要修改，例如/data/6380

#集群配置
cluster-enabled yes   #启用集群
cluster-config-file /opt/redis/6380/nodes.conf   #多实例情况下需要修改，例如/6380/
cluster-node-timeout 5000   #集群超时时间


#从ping主间隔默认10秒
#复制超时时间
#repl-timeout 60

#远距离主从
#config set client-output-buffer-limit "slave 536870912 536870912 0"
#config set repl-backlog-size 209715200
```

### 拷贝redis.conf文件到文件夹中

cp redis.conf 7000/redis-7000.conf

mkdir 7000 7001 7002 7003 7004 7005

### 将配置文件分别拷贝到7001-7008中，需要修改端口号即可.

在vim下执行以下命令可以先将文件中的全部7000修改为7001

:%s/7000/7001/g 注：代表将当前文本的所有的7000替换成7001

分别将7002-7008的配置文件进行修改

.创建shell脚本文件启动多个redis服务从7000-7008

```
#!/bin/sh
redis-server 7000/redis-7000.conf &
redis-server 7001/redis-7001.conf &
redis-server 7002/redis-7002.conf &
redis-server 7003/redis-7003.conf &
redis-server 7004/redis-7004.conf &
redis-server 7005/redis-7005.conf &
redis-server 7006/redis-7006.conf &
redis-server 7007/redis-7007.conf &
redis-server 7008/redis-7008.conf &
```

## 创建redis-cluster

通过redis-trib.rb来创建redis集群，

1、安装的ruby，通过yum 源下载安装的ruby可能版本过低，导致:

输入命令 “ gem install redis “ 出现 “ ERROR: Error installing redis redis requires Ruby version >= 2.2.2. “

所以要安装ruby的RVM管理工具获取ruby的最新包

使用curl安装rvm ，输入命令 “ curl -L get.rvm.io | bash -s stable “ 进行安装，这个时候可能会出现

```
GPG signature verification failed for ‘/usr/local/rvm/archives/rvm-1.29.3.tgz‘ - ‘https://github.com/rvm/rvm/releases/download/1.29.3/1.29.3.tar.gz.asc‘! Try to install GPG v2 and then fetch the public key:
```

这时候要输入密钥 `gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3`

然后在执行

```
curl -L get.rvm.io | bash -s stable
source /usr/local/rvm/scripts/rvm
```

查看rvm中管理的所有ruby版本，
输入命令 `rvm list known`进行查询，查询出来如下

```
# MRI Rubies
[ruby-]1.8.6[-p420]
[ruby-]1.8.7[-head] # security released on head
[ruby-]1.9.1[-p431]
[ruby-]1.9.2[-p330]
[ruby-]1.9.3[-p551]
[ruby-]2.0.0[-p648]
[ruby-]2.1[.10]
[ruby-]2.2[.7]
[ruby-]2.3[.4]
[ruby-]2.4[.1]
ruby-head

# for forks use: rvm install ruby-head-<name> --url https://github.com/github/ruby.git --branch 2.2

# JRuby
jruby-1.6[.8]
jruby-1.7[.27]
jruby[-9.1.13.0]
jruby-head

# Rubinius
rbx-1[.4.3]
rbx-2.3[.0]
rbx-2.4[.1]
rbx-2[.5.8]
rbx-3[.84]
rbx-head

# Opal
opal

# Minimalistic ruby implementation - ISO 30170:2012
mruby-1.0.0
mruby-1.1.0
mruby-1.2.0
mruby-1[.3.0]
mruby[-head]

# Ruby Enterprise Edition
ree-1.8.6
ree[-1.8.7][-2012.02]

# Topaz
topaz

# MagLev
maglev[-head]
maglev-1.0.0

# Mac OS X Snow Leopard Or Newer
macruby-0.10
macruby-0.11
macruby[-0.12]
macruby-nightly
```

选择一个你喜欢的版本进行安装，但首先提醒一下，你所选择的版本不能低于 “ 2.0.0 “ 就可以了，输入命令 “ rvm install 2.3.4 “ 进行安装，如下

```
[root@VM_0_12_centos ~]# rvm install 2.3.4
Searching for binary rubies, this might take some time.
Found remote file https://rvm_io.global.ssl.fastly.net/binaries/centos/7/x86_64/ruby-2.3.4.tar.bz2
Checking requirements for centos.
Installing requirements for centos.
Installing required packages: libffi-devel, readline-devel, sqlite-devel, zlib-devel, libyaml-devel, openssl-devel.............
Requirements installation successful.
ruby-2.3.4 - #configure
ruby-2.3.4 - #download
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 25.2M  100 25.2M    0     0   461k      0  0:00:55  0:00:55 --:--:--  225k
No checksum for downloaded archive, recording checksum in user configuration.
ruby-2.3.4 - #validate archive
ruby-2.3.4 - #extract
ruby-2.3.4 - #validate binary
ruby-2.3.4 - #setup
ruby-2.3.4 - #gemset created /usr/local/rvm/gems/ruby-2.3.4@global
ruby-2.3.4 - #importing gemset /usr/local/rvm/gemsets/global.gems..............................
ruby-2.3.4 - #generating global wrappers........
ruby-2.3.4 - #gemset created /usr/local/rvm/gems/ruby-2.3.4
ruby-2.3.4 - #importing gemsetfile /usr/local/rvm/gemsets/default.gems evaluated to empty gem list
ruby-2.3.4 - #generating default wrappers........
```

redis-trib.rb命令与redis-cli命令放置在同一个目录中，可全路径执行或者创建别名。

redis-trib.rb create –replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:70005

–replicas 1 表示一主一从，

#### 脚本使用帮助

- 查看脚本帮助

```
$ ruby redis-trib.rb help
Usage: redis-trib <command> <options> <arguments ...>

  create          host1:port1 ... hostN:portN
                  --replicas <arg>
  check           host:port
  info            host:port
  fix             host:port
                  --timeout <arg>
  reshard         host:port
                  --from <arg>
                  --to <arg>
                  --slots <arg>
                  --yes
                  --timeout <arg>
                  --pipeline <arg>
  rebalance       host:port
                  --weight <arg>
                  --auto-weights
                  --use-empty-masters
                  --timeout <arg>
                  --simulate
                  --pipeline <arg>
                  --threshold <arg>
  add-node        new_host:new_port existing_host:existing_port
                  --slave
                  --master-id <arg>
  del-node        host:port node_id
  set-timeout     host:port milliseconds
  call            host:port command arg arg .. arg
  import          host:port
                  --from <arg>
                  --copy
                  --replace
  help            (show this help)

For check, fix, reshard, del-node, set-timeout you can specify the host and port of any working node in the cluster.
```

- 各选项详解

```
1、create：创建集群
2、check：检查集群
3、info：查看集群信息
4、fix：修复集群
5、reshard：在线迁移slot
6、rebalance：平衡集群节点slot数量
7、add-node：将新节点加入集群
8、del-node：从集群中删除节点
9、set-timeout：设置集群节点间心跳连接的超时时间
10、call：在集群全部节点上执行命令
11、import：将外部redis数据导入集群
```