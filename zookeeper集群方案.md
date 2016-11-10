#zookeeper集群方案手册


##测试环境
#####系统环境：Centos6.5
#####服务器ip:
    zookeeper01:	192.168.122.217
    zookeeper02:	192.168.122.132
    zookeeper03:	192.168.122.112
#####软件版本:
	jdk-7u80-linux-x64.rpm
    zookeeper-3.4.6.tar.gz
	官方下载地址:http://www.apache.org/dyn/closer.cgi/zookeeper/

####安装jdk
s三台服务器均需要安装
```
[root@zookeeper01 ~]# rpm -vih jdk-7u80-linux-x64.rpm
```
####zookeeper的安装配置
在zookeeper01上
```
[root@zookeeper01 ~]# tar zxvf zookeeper-3.4.6.tar.gz 
[root@zookeeper01 ~]# mv zookeeper-3.4.6 /usr/local/zookeeper-3.4.6
[root@zookeeper01 ~]# vim /etc/profile
......
#设置zookeeper环境变量
#ZOOKEEPER  
ZOOKEEPER=/usr/local/zookeeper-3.4.6 
PATH=$PATH:$ZOOKEEPER/bin
[root@zookeeper01 ~]# source /etc/profile
[root@zookeeper01 ~]# cp /usr/local/zookeeper-3.4.6/conf/zoo_sample.cfg /usr/local/zookeeper-3.4.6/conf/zoo.cfg 	#拷贝zookeeper的配置文件
[root@zookeeper01 ~]# cd /usr/local/zookeeper-3.4.6/conf
[root@zookeeper01 conf]# sed -i 's#dataDir=/tmp/zookeeper#dataDir=/data/zookeeper#g' zoo.cfg 	#修改data目录
[root@zookeeper01 conf]# vim zoo.cfg
......
#server.A=B：C：D：A表示第几号服务器；B 是这个服务器的 ip 地址；C 与集群中的 Leader 服务器交换信息的端口；D 表示Leader 服务器挂了，执行选举时服务器相互通信的端口
server.1=192.168.122.217:2888:3888 
server.2=192.168.122.132:2888:3888
server.3=192.168.122.112:2888:3888
[root@zookeeper01 conf]# mkdir -p /data/zookeeper
[root@zookeeper01 ~]# cd /data/zookeeper/
[root@zookeeper01 zookeeper]# vim myid 	#常见myid文件
1	#此处的1对应server.1
#启动zookeeper
[root@zookeeper01 zookeeper]# zkServer.sh start
```

在zookeeper02上
```
[root@zookeeper02 ~]# tar zxvf zookeeper-3.4.6.tar.gz 
[root@zookeeper02 ~]# mv zookeeper-3.4.6 /usr/local/zookeeper-3.4.6
[root@zookeeper02 ~]# vim /etc/profile
......
#设置zookeeper环境变量
#ZOOKEEPER  
ZOOKEEPER=/usr/local/zookeeper-3.4.6 
PATH=$PATH:$ZOOKEEPER/bin
[root@zookeeper02 ~]# source /etc/profile
[root@zookeeper02 ~]# cp /usr/local/zookeeper-3.4.6/conf/zoo_sample.cfg /usr/local/zookeeper-3.4.6/conf/zoo.cfg 	#拷贝zookeeper的配置文件
[root@zookeeper02 ~]# cd /usr/local/zookeeper-3.4.6/conf
[root@zookeeper02 conf]# sed -i 's#dataDir=/tmp/zookeeper#dataDir=/data/zookeeper#g' zoo.cfg 	#修改data目录
[root@zookeeper02 conf]# vim zoo.cfg
......
#server.A=B：C：D：A表示第几号服务器；B 是这个服务器的 ip 地址；C 与集群中的 Leader 服务器交换信息的端口；D 表示Leader 服务器挂了，执行选举时服务器相互通信的端口
server.1=192.168.122.217:2888:3888 
server.2=192.168.122.132:2888:3888
server.3=192.168.122.112:2888:3888
[root@zookeeper02 conf]# mkdir -p /data/zookeeper
[root@zookeeper02 ~]# cd /data/zookeeper/
[root@zookeeper02 zookeeper]# vim myid 	#常见myid文件
2	#此处的2对应server.2
#启动zookeeper
[root@zookeeper02 zookeeper]# zkServer.sh start
```

在zookeeper03上
```
[root@zookeeper03 ~]# tar zxvf zookeeper-3.4.6.tar.gz 
[root@zookeeper03 ~]# mv zookeeper-3.4.6 /usr/local/zookeeper-3.4.6
[root@zookeeper03 ~]# vim /etc/profile
......
#设置zookeeper环境变量
#ZOOKEEPER  
ZOOKEEPER=/usr/local/zookeeper-3.4.6 
PATH=$PATH:$ZOOKEEPER/bin
[root@zookeeper03 ~]# source /etc/profile
[root@zookeeper03 ~]# cp /usr/local/zookeeper-3.4.6/conf/zoo_sample.cfg /usr/local/zookeeper-3.4.6/conf/zoo.cfg 	#拷贝zookeeper的配置文件
[root@zookeeper03 ~]# cd /usr/local/zookeeper-3.4.6/conf
[root@zookeeper03 conf]# sed -i 's#dataDir=/tmp/zookeeper#dataDir=/data/zookeeper#g' zoo.cfg 	#修改data目录
[root@zookeeper03 conf]# vim zoo.cfg
......
#server.A=B：C：D：A表示第几号服务器；B 是这个服务器的 ip 地址；C 与集群中的 Leader 服务器交换信息的端口；D 表示Leader 服务器挂了，执行选举时服务器相互通信的端口
server.1=192.168.122.217:2888:3888 
server.2=192.168.122.132:2888:3888
server.3=192.168.122.112:2888:3888
[root@zookeeper03 conf]# mkdir -p /data/zookeeper
[root@zookeeper03 ~]# cd /data/zookeeper/
[root@zookeeper03 zookeeper]# vim myid 	#常见myid文件
3	#此处的3对应server.3
#启动zookeeper
[root@zookeeper03 zookeeper]# zkServer.sh start
```


#查看zookeeper状态
```
#zookeeper01
[root@zookeeper01 zookeeper]# zkServer.sh status
JMX enabled by default
Using config: /usr/local/zookeeper-3.4.6/bin/../conf/zoo.cfg
Mode: leader 	#区别另外两台

#zookeeper02
[root@zookeeper02 zookeeper]# zkServer.sh status
JMX enabled by default
Using config: /usr/local/zookeeper-3.4.6/bin/../conf/zoo.cfg
Mode: follower

#zookeeper03
[root@zookeeper03 zookeeper]# zkServer.sh status
JMX enabled by default
Using config: /usr/local/zookeeper-3.4.6/bin/../conf/zoo.cfg
Mode: follower
```