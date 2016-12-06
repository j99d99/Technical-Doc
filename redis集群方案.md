#redis集群方案手册


##测试环境
#####系统环境：Centos6.5
#####服务器ip:
    redis01:	192.168.122.217
    redis02:	192.168.122.132
    redis03:	192.168.122.112
#####软件版本:
    redis-3.0.4.tar.gz


##redis实例的安装配置
####此处以redis01的安装为例，另外两台的安装与redis01步骤相同
#####一.redis安装
```
[root@redis01 ~]# wget http://download.redis.io/releases/redis-3.0.4.tar.gz  		#下载软件
[root@redis01 ~]# yum install -y gcc-c++		#安装gcc-c++依赖,初始最小化系统未安装，需安装
[root@redis01 ~]# tar zxvf redis-3.0.4.tar.gz
[root@redis01 ~]# cd redis-3.0.4
[root@redis01 redis-3.0.4]# make PREFIX=/usr/local/redis6379 install		#一台服务器安装两个redis,所以命名以端口区分
[root@redis01 redis-3.0.4]# make PREFIX=/usr/local/redis6380 install
```
#####二.redis的初始配置
```
[root@redis01 redis-3.0.4]# cd utils/
[root@redis01 utils]# sh install_server.sh 	##redis6379的初始化配置
Welcome to the redis service installer
This script will help you easily set up a running redis server

Please select the redis port for this instance: [6379] 	#默认6379端口,不需要修改，直接回车即可
Selecting default: 6379
Please select the redis config file name [/etc/redis/6379.conf] 	#默认，直接回车即可
Selected default - /etc/redis/6379.conf
Please select the redis log file name [/var/log/redis_6379.log] /data/weblogs/redis/redis_6379.log 	#修改redis6379的日志路径
Please select the data directory for this instance [/var/lib/redis/6379] /data/redis/6379 			#修改redis6379实例数据位置
Please select the redis executable path [] /usr/local/redis6379/bin/redis-server				#redis6379的执行文件目录
Selected config:
Port           : 6379
Config file    : /etc/redis/6379.conf
Log file       : /data/weblogs/redis/redis_6379.log
Data dir       : /data/redis/6379
Executable     : /usr/local/redis6379/bin/redis-server
Cli Executable : /usr/local/redis6379/bin/redis-cli
Is this ok? Then press ENTER to go on or Ctrl-C to abort.
Copied /tmp/6379.conf => /etc/init.d/redis_6379
Installing service...
Successfully added to chkconfig!
Successfully added to runlevels 345!
Starting Redis server...
Installation successful!

[root@redis01 utils]# sh install_server.sh 	##redis6380的初始化配置 
Welcome to the redis service installer
This script will help you easily set up a running redis server

Please select the redis port for this instance: [6379] 6380			##配置redis6380的端口为6380
Please select the redis config file name [/etc/redis/6380.conf] 	#配置文件路径，直接回车即可
Selected default - /etc/redis/6380.conf
Please select the redis log file name [/var/log/redis_6380.log] /data/weblogs/redis/redis_6380.log 		#修改redis6380的日志路径
Please select the data directory for this instance [/var/lib/redis/6380] /data/redis/6380				#修改redis6380实例数据位置
Please select the redis executable path [] /usr/local/redis6380/bin/redis-server  						#redis6380的执行文件目录
Selected config:
Port           : 6380
Config file    : /etc/redis/6380.conf
Log file       : /data/weblogs/redis/redis_6380.log
Data dir       : /data/redis/6380
Executable     : /usr/local/redis6380/bin/redis-server
Cli Executable : /usr/local/redis6380/bin/redis-cli
Is this ok? Then press ENTER to go on or Ctrl-C to abort.
Copied /tmp/6380.conf => /etc/init.d/redis_6380
Installing service...
Successfully added to chkconfig!
Successfully added to runlevels 345!
Starting Redis server...
Installation successful!
```

####三.开启redis持久化
修改配置文件参数，可使用如下命令修改
```
[root@redis01 ~]# cd
[root@redis01 ~]# sed -i 's/appendonly no/appendonly yes/g' /etc/redis/6379.conf
[root@redis01 ~]# sed -i 's/appendonly no/appendonly yes/g' /etc/redis/6380.conf
```
或着手动修改配置文件的"appendonly no"参数为"appendonly yes"

####四.重启redis服务并确认有生成持久化文件
```
[root@redis01 ~]# service redis_6379 restart 
Stopping ...
Redis stopped
Starting Redis server...
[root@redis01 ~]# service redis_6380 restart
Stopping ...
Redis stopped
Starting Redis server...
[root@redis01 ~]# ll /data/redis/6379/
appendonly.aof  dump.rdb        
[root@redis01 ~]# ll /data/redis/6380/
appendonly.aof  dump.rdb
```

##redis集群配置
每台redis服务器都需要修改，此处以redis01为例，其他服务器操作步骤相同
####一.修改redis配置文件

```
#修改redis6379
[root@redis01 ~]# sed -i 's/# cluster-enabled yes/cluster-enabled yes/g' /etc/redis/6379.conf
[root@redis01 ~]# sed -i 's/# cluster-config-file nodes-6379.conf/cluster-config-file nodes-6379.conf/g' /etc/redis/6379.conf
[root@redis01 ~]# sed -i 's/# cluster-node-timeout 15000/cluster-node-timeout 5000/g' /etc/redis/6379.conf

#修改redis6380
[root@redis01 ~]# sed -i 's/# cluster-enabled yes/cluster-enabled yes/g' /etc/redis/6380.conf
[root@redis01 ~]# sed -i 's/# cluster-config-file nodes-6379.conf/cluster-config-file nodes-6380.conf/g' /etc/redis/6380.conf
[root@redis01 ~]# sed -i 's/# cluster-node-timeout 15000/cluster-node-timeout 5000/g' /etc/redis/6380.conf

#重启redis服务
[root@redis01 ~]# service redis_6379 restart 
Stopping ...
Redis stopped
Starting Redis server...
[root@redis01 ~]# service redis_6380 restart
Stopping ...
Redis stopped
Starting Redis server...
```

####二.安装集群需要的相关组件
```
[root@redis01 ~]# yum -y  install zlib ruby rubygems rubygems-devel
[root@redis01 ~]# gem install redis  --version 3.0.4
```

####三.创建集群
```
#在redis源码解压目录，只需在三台中的一台操作即可,此处在redis01操作
[root@redis01 ~]# cd redis-3.0.4
[root@redis01 redis-3.0.4]# ./src/redis-trib.rb create --replicas 1 192.168.122.217:6379 192.168.122.217:6380 192.168.122.132:6379 192.168.122.132:6380 192.168.122.112:6379 192.168.122.112:6380
>>> Creating cluster
Connecting to node 192.168.122.217:6379: OK
Connecting to node 192.168.122.217:6380: OK
Connecting to node 192.168.122.132:6379: OK
Connecting to node 192.168.122.132:6380: OK
Connecting to node 192.168.122.112:6379: OK
Connecting to node 192.168.122.112:6380: OK
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
192.168.122.112:6379
192.168.122.217:6379
192.168.122.132:6379
Adding replica 192.168.122.217:6380 to 192.168.122.112:6379
Adding replica 192.168.122.112:6380 to 192.168.122.217:6379
Adding replica 192.168.122.132:6380 to 192.168.122.132:6379
M: 6343d46d255182e0e116dfad536028702e776251 192.168.122.217:6379
   slots:5461-10922 (5462 slots) master
S: 28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380
   replicates c18bae19c335c2bfad4c4628d562bd937bd608ec
M: 3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379
   slots:10923-16383 (5461 slots) master
S: 5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380
   replicates 3bde122a2d531d562f932657a67ae0461479cbd1
M: c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379
   slots:0-5460 (5461 slots) master
S: c67e2adf957cb6f123103c677fb998135ca61a5b 192.168.122.112:6380
   replicates 6343d46d255182e0e116dfad536028702e776251
Can I set the above configuration? (type 'yes' to accept): yes	#确认集群节点信息
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join....
>>> Performing Cluster Check (using node 192.168.122.217:6379)
M: 6343d46d255182e0e116dfad536028702e776251 192.168.122.217:6379
   slots:5461-10922 (5462 slots) master
M: 28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380
   slots: (0 slots) master
   replicates c18bae19c335c2bfad4c4628d562bd937bd608ec
M: 3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379
   slots:10923-16383 (5461 slots) master
M: 5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380
   slots: (0 slots) master
   replicates 3bde122a2d531d562f932657a67ae0461479cbd1
M: c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379
   slots:0-5460 (5461 slots) master
M: c67e2adf957cb6f123103c677fb998135ca61a5b 192.168.122.112:6380
   slots: (0 slots) master
   replicates 6343d46d255182e0e116dfad536028702e776251
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
redis集群创建成功

####查看集群节点
```
[root@redis01 redis-3.0.4]# /usr/local/redis6379/bin/redis-cli -c -p 6379 	#登录集群
127.0.0.1:6379> cluster nodes 		#查看集群节点信息
6343d46d255182e0e116dfad536028702e776251 192.168.122.217:6379 myself,master - 0 0 1 connected 5461-10922
c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379 master - 0 1478586739489 5 connected 0-5460
c67e2adf957cb6f123103c677fb998135ca61a5b 192.168.122.112:6380 slave 6343d46d255182e0e116dfad536028702e776251 0 1478586740491 6 connected
28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380 slave c18bae19c335c2bfad4c4628d562bd937bd608ec 0 1478586739991 5 connected
3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379 master - 0 1478586739489 3 connected 10923-16383
5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380 slave 3bde122a2d531d562f932657a67ae0461479cbd1 0 1478586740491 4 connected
127.0.0.1:6379> cluster info	#查看集群信息
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_sent:10661
cluster_stats_messages_received:10661
```

####集群节点状态检查
```
[root@redis01 redis-3.0.4]# ./src/redis-trib.rb check 192.168.122.217:6379
Connecting to node 192.168.122.217:6379: OK
Connecting to node 192.168.122.112:6379: OK
Connecting to node 192.168.122.112:6380: OK
Connecting to node 192.168.122.217:6380: OK
Connecting to node 192.168.122.132:6379: OK
Connecting to node 192.168.122.132:6380: OK
>>> Performing Cluster Check (using node 192.168.122.217:6379)
M: 6343d46d255182e0e116dfad536028702e776251 192.168.122.217:6379
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
M: c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: c67e2adf957cb6f123103c677fb998135ca61a5b 192.168.122.112:6380
   slots: (0 slots) slave
   replicates 6343d46d255182e0e116dfad536028702e776251
S: 28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380
   slots: (0 slots) slave
   replicates c18bae19c335c2bfad4c4628d562bd937bd608ec
M: 3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: 5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380
   slots: (0 slots) slave
   replicates 3bde122a2d531d562f932657a67ae0461479cbd1
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
#####集群的关闭和重新开启

关闭redis集群，关闭redis集群所有节点的服务即可
开启redis集群，开启redis集群所有节点的服务即可

####集群节点的增加和删除以及分片(数据槽slot重新分配)
####集群节点的删除
#####从节点的删除
```
#删除从节点直接删除即可，使用命令
#删除参考命令解释:	 redis-trib.rb del-node "del_slave_ip:port" "slave_node_id"
[root@redis01 redis-3.0.4]# ./src/redis-trib.rb del-node 192.168.122.112:6380 c67e2adf957cb6f123103c677fb998135ca61a5b	#删除192.168.122.112:6380这个从节点
>>> Removing node c67e2adf957cb6f123103c677fb998135ca61a5b from cluster 192.168.122.112:6380
Connecting to node 192.168.122.112:6380: OK
Connecting to node 192.168.122.217:6380: OK
Connecting to node 192.168.122.217:6379: OK
Connecting to node 192.168.122.132:6379: OK
Connecting to node 192.168.122.112:6379: OK
Connecting to node 192.168.122.132:6380: OK
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
```

#####主节点的删除
######步骤:
1.删除主节点的从节点(参考删除从节点)
2.去掉主节点分配的slot(删除主节点为192.168.122.217:6379的主节点为例)
```
[root@redis01 redis-3.0.4]# ./src/redis-trib.rb reshard 192.168.122.217:6379
...	#省略号表示省略无关内容
How many slots do you want to move (from 1 to 16384)? 16384 		#被移除的数据槽,可根据cluster nodes 查看
...
What is the receiving node ID? 28cca88d789477a1bae4abfbe20c187bb54583cf 		#接收被删除节点数据的主节点
...
Source node #1:6343d46d255182e0e116dfad536028702e776251 #被删除主节点的node-id  
Source node #2:done 		#结束标志，开始执行
...
Do you want to proceed with the proposed reshard plan (yes/no)? yes #删除要被删除节点的slot后，重新分配
[root@redis01 redis-3.0.4]# /usr/local/redis6379/bin/redis-cli -c -p 6379 cluster nodes 	#查看节点信息
28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380 master - 0 1478593786321 7 connected 0-10922
6343d46d255182e0e116dfad536028702e776251 192.168.122.217:6379 myself,master - 0 0 1 connected		#此时要被删除的主节点的slot信息没有了
c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379 slave 28cca88d789477a1bae4abfbe20c187bb54583cf 0 1478593786823 7 connected
3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379 master - 0 1478593785820 3 connected 10923-16383
5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380 slave 3bde122a2d531d562f932657a67ae0461479cbd1 0 1478593787322 4 connected

```
3.删除主节点(此时删除主节点跟删除从节点操作一样)




####集群节点的增加
主节点的增加
```
#新添加的主节点相当于其他在用主节点下面的slave
[root@redis01 redis-3.0.4]# ./src/redis-trib.rb add-node 192.168.122.217:6379 192.168.122.217:6380
>>> Adding node 192.168.122.217:6379 to cluster 192.168.122.217:6380
Connecting to node 192.168.122.217:6380: OK
Connecting to node 192.168.122.132:6379: OK
Connecting to node 192.168.122.112:6379: OK
Connecting to node 192.168.122.132:6380: OK
>>> Performing Cluster Check (using node 192.168.122.217:6380)
M: 28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380
   slots:0-10922 (10923 slots) master
   1 additional replica(s)
M: 3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379
   slots: (0 slots) slave
   replicates 28cca88d789477a1bae4abfbe20c187bb54583cf
S: 5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380
   slots: (0 slots) slave
   replicates 3bde122a2d531d562f932657a67ae0461479cbd1
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
Connecting to node 192.168.122.217:6379: OK
[ERR] Node 192.168.122.217:6379 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0.	#此处报错原因是因为前面测试删除节点，不是使用全新安装的redis服务,相关目录还保留残留文件,需手动删除
#以下是解决办法
[root@redis01 redis-3.0.4]# cd /data/redis/6379/
[root@redis01 6379]# rm -rf nodes-6379.conf 	#删除原有节点信息文件
[root@redis01 redis-3.0.4]# service redis_6379 restart 	#重启redis服务
Stopping ...
Waiting for Redis to shutdown ...
Redis stopped
Starting Redis server...
[root@redis01 redis-3.0.4]# ./src/redis-trib.rb add-node 192.168.122.217:6379 192.168.122.217:6380 	#重新添加新节点
>>> Adding node 192.168.122.217:6379 to cluster 192.168.122.217:6380
Connecting to node 192.168.122.217:6380: OK
Connecting to node 192.168.122.132:6379: OK
Connecting to node 192.168.122.112:6379: OK
Connecting to node 192.168.122.132:6380: OK
>>> Performing Cluster Check (using node 192.168.122.217:6380)
M: 28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380
   slots:0-10922 (10923 slots) master
   1 additional replica(s)
M: 3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379
   slots: (0 slots) slave
   replicates 28cca88d789477a1bae4abfbe20c187bb54583cf
S: 5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380
   slots: (0 slots) slave
   replicates 3bde122a2d531d562f932657a67ae0461479cbd1
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
Connecting to node 192.168.122.217:6379: OK
>>> Send CLUSTER MEET to node 192.168.122.217:6379 to make it join the cluster.
[OK] New node added correctly.
[root@redis01 redis-3.0.4]# /usr/local/redis6379/bin/redis-cli -c -p 6380 cluster nodes            #查看集群节点信息
3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379 master - 0 1478594694846 3 connected 10923-16383
c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379 slave 28cca88d789477a1bae4abfbe20c187bb54583cf 0 1478594693844 7 connected
e0fab3c8d591f84535c46841c16c9c287c6d73e2 192.168.122.217:6379 master - 0 1478594692841 0 connected  #发现添加成功
28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380 myself,master - 0 0 7 connected 0-10922
5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380 slave 3bde122a2d531d562f932657a67ae0461479cbd1 0 1478594693342 4 connected
#添加成功的节点并没有真正在集群中使用，集群的slot槽需要重新分配
[root@redis01 redis-3.0.4]# ./src/redis-trib.rb reshard 192.168.122.217:6379
Connecting to node 192.168.122.217:6379: OK
Connecting to node 192.168.122.217:6380: OK
Connecting to node 192.168.122.112:6379: OK
Connecting to node 192.168.122.132:6380: OK
Connecting to node 192.168.122.132:6379: OK
>>> Performing Cluster Check (using node 192.168.122.217:6379)
...... 		#s省略号表示省略无关信息
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 5600  		#设置solt，设置每个节点的solt区间范围
What is the receiving node ID? e0fab3c8d591f84535c46841c16c9c287c6d73e2 #此处填新添加的节点的nodeID
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:all #表示全部重新分配
....
Do you want to proceed with the proposed reshard plan (yes/no)? yes 	#确认重启分配
[root@redis01 redis-3.0.4]# /usr/local/redis6379/bin/redis-cli -c -p 6380 cluster nodes #查看节点信息，发现新添加的主节点也有slot
3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379 master - 0 1478595533541 9 connected 1914-5599
c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379 slave 28cca88d789477a1bae4abfbe20c187bb54583cf 0 1478595533041 10 connected
e0fab3c8d591f84535c46841c16c9c287c6d73e2 192.168.122.217:6379 master - 0 1478595533541 8 connected 9286-16383
28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380 myself,master - 0 0 10 connected 0-1913 5600-9285
5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380 slave 3bde122a2d531d562f932657a67ae0461479cbd1 0 1478595532539 9 connected
```
从节点的增加
#添加从节点
```
#命令参考:redis-trib.rb add-node --slave --master-id "master_node_id" "new_slave" "集群已有任一集群节点"
[root@redis01 redis-3.0.4]# redis-trib.rb add-node --slave --master-id e0fab3c8d591f84535c46841c16c9c287c6d73e2 192.168.122.112:6380 192.168.122.132:6380
[root@redis01 redis-3.0.4]# /usr/local/redis6379/bin/redis-cli -c -p 6380 cluster nodes 	#查看节点信息
c67e2adf957cb6f123103c677fb998135ca61a5b :0 slave,noaddr e0fab3c8d591f84535c46841c16c9c287c6d73e2 1478596060068 1478596057963 8 disconnected
3bde122a2d531d562f932657a67ae0461479cbd1 192.168.122.132:6379 master - 0 1478596780704 9 connected 1914-5599
400667d62c9dd1581f5c5161d9d52b91e1ee137d 192.168.122.112:6380 slave e0fab3c8d591f84535c46841c16c9c287c6d73e2 0 1478596781206 8 connected 	#添加从节点成功
c18bae19c335c2bfad4c4628d562bd937bd608ec 192.168.122.112:6379 slave 28cca88d789477a1bae4abfbe20c187bb54583cf 0 1478596781706 10 connected
e0fab3c8d591f84535c46841c16c9c287c6d73e2 192.168.122.217:6379 master - 0 1478596780203 8 connected 9286-16383
28cca88d789477a1bae4abfbe20c187bb54583cf 192.168.122.217:6380 myself,master - 0 0 10 connected 0-1913 5600-9285
5f25c4e5e0891530e54570f51cf9de8015c08d9e 192.168.122.132:6380 slave 3bde122a2d531d562f932657a67ae0461479cbd1 0 1478596780203 9 connected
```