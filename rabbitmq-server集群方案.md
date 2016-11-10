#rabbitmq集群方案手册


##测试环境
#####系统环境：Centos6.5
#####服务器ip:
    rabbitmq01:	192.168.122.29
    rabbitmq02:	192.168.122.192
    rabbitmq03:	192.168.122.210
#####软件版本:
    epel-release-latest-6.noarch.rpm
    rabbitmq-server-3.3.4-1.noarch.rpm


#####配置hosts
```
[root@rabbitmq01 ~]# vim /etc/hosts
192.168.122.29 rabbitmq01
192.168.122.192 rabbitmq02
192.168.122.210 rabbitmq03
```
#####一.安装epel源
```
#在集群各节点都操作,已rabbitmq01为例
[root@rabbitmq01 ~]# rpm -vih epel-release-latest-6.noarch.rpm
```

#####二.安装erlang
```
#在集群各节点都操作,已rabbitmq01为例
[root@rabbitmq01 ~]# yum install -y erlang
```

#####三.安装rabbitma-server
```
#在集群各节点都操作,已rabbitmq01为例
[root@rabbitmq01 ~]# rpm -vih rabbitmq-server-3.3.4-1.noarch.rpm 
```

#####四.修改数据和日志目录
```
#在集群各节点都操作,已rabbitmq01为例
[root@rabbitmq01 ~]# vim /etc/rabbitmq/rabbitmq-env.conf
#指定数据存放目录
RABBITMQ_MNESIA_BASE=/data/rabbitmq/mnesia
#指定日志目录
RABBITMQ_LOG_BASE=/data/weblogs/rabbitmq
[root@rabbitmq01 ~]# mkdir -p /data/rabbitmq 	#创建数据目录
[root@rabbitmq01 ~]# mkdir -p /data/weblogs/rabbitmq 	#创建日志目录
[root@rabbitmq01 ~]# chown -R rabbitmq:rabbitmq /data/weblogs/rabbitmq/
[root@rabbitmq01 ~]# chown -R rabbitmq:rabbitmq /data/rabbitmq/
```

####五.安装rabbit的web管理界面插件
```
#在集群各节点都操作,已rabbitmq01为例
[root@rabbitmq01 ~]# rabbitmq-plugins list 	#查看web管理界面插件是否安装
[ ] amqp_client                       3.3.4
[ ] cowboy                            0.5.0-rmq3.3.4-git4b93c2d
[ ] eldap                             3.3.4-gite309de4
[ ] mochiweb                          2.7.0-rmq3.3.4-git680dba8
[ ] rabbitmq_amqp1_0                  3.3.4
[ ] rabbitmq_auth_backend_ldap        3.3.4
[ ] rabbitmq_auth_mechanism_ssl       3.3.4
[ ] rabbitmq_consistent_hash_exchange 3.3.4
[ ] rabbitmq_federation               3.3.4
[ ] rabbitmq_federation_management    3.3.4
[ ] rabbitmq_management               3.3.4
[ ] rabbitmq_management_agent         3.3.4
[ ] rabbitmq_management_visualiser    3.3.4
[ ] rabbitmq_mqtt                     3.3.4
[ ] rabbitmq_shovel                   3.3.4
[ ] rabbitmq_shovel_management        3.3.4
[ ] rabbitmq_stomp                    3.3.4
[ ] rabbitmq_test                     3.3.4
[ ] rabbitmq_tracing                  3.3.4
[ ] rabbitmq_web_dispatch             3.3.4
[ ] rabbitmq_web_stomp                3.3.4
[ ] rabbitmq_web_stomp_examples       3.3.4
[ ] sockjs                            0.3.4-rmq3.3.4-git3132eb9
[ ] webmachine                        1.10.3-rmq3.3.4-gite9359c7
#安装web管理界面插件
[root@rabbitmq01 ~]# rabbitmq-plugins enable rabbitmq_management
The following plugins have been enabled:
  mochiweb
  webmachine
  rabbitmq_web_dispatch
  amqp_client
  rabbitmq_management_agent
  rabbitmq_management
Plugin configuration has changed. Restart RabbitMQ for changes to take effect.
```

###配置rabbitmq远程web访问
```
#在集群各节点都操作,已rabbitmq01为例
#拷贝rabbit配置文件到/etc/rabbitmq目录
[root@rabbitmq01 ~]# cp /usr/share/doc/rabbitmq-server-3.3.4/rabbitmq.config.example /etc/rabbitmq/rabbitmq.config
[root@rabbitmq01 ~]# vim /etc/rabbitmq/rabbitmq.config
...
[
 {rabbit,
  [%%
  {loopback_users, []}	#开启远程访问，默认账户guest,密码guest
  ]}
].
#启动服务
[root@rabbitmq01 ~]# service rabbitmq-server start
```

#####六.设置节点cookie，保证服务器直接cookie一致
```
#rabbitmq启动后会在/var/lib/rabbitmq/目录下生成.erlang.cookie文件，集群节点的此文件需保持一致
#在rabitmq01上
[root@rabbitmq01 ~]# service rabbitmq-server stop

#在rabitmq02上
[root@rabbitmq02 ~]# service rabbitmq-server stop

#在rabitmq03上
[root@rabbitmq03 ~]# service rabbitmq-server stop

#在rabitmq01上
[root@rabbitmq01 ~]# cd /var/lib/rabbitmq/
[root@rabbitmq01 rabbitmq]# scp .erlang.cookie  root@rabbitmq02:/var/lib/rabbitmq/ 	#拷贝cookie文件到rabbitmq02
[root@rabbitmq01 rabbitmq]# scp .erlang.cookie  root@rabbitmq03:/var/lib/rabbitmq/ 	#拷贝cookie文件到rabbitmq03

#使用如下命令启动rabbitmq
#在rabitmq01上
[root@rabbitmq01 ~]# rabbitmq-server -detached

#在rabitmq02上
[root@rabbitmq02 ~]# rabbitmq-server -detached

#在rabitmq03上
[root@rabbitmq03 ~]# rabbitmq-server -detached

```



#####七.创建集群
将rabbitmq02与rabbitmq01组建集群
```
#在rabitmq01上
#查看状态,此时显示只有rabbitmq01一个节点
[root@rabbitmq01 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq01 ...
[{nodes,[{disc,[rabbit@rabbitmq01]}]},
 {running_nodes,[rabbit@rabbitmq01]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.
#在rabitmq02上
#停止集群服务
[root@rabbitmq02 ~]# rabbitmqctl stop_app
Stopping node rabbit@rabbitmq02 ...
...done.
#将rabbitmq02加入到rabbitmq01集群，默认是disc磁盘节点，加--ram参数指定为内存节点
[root@rabbitmq02 ~]# rabbitmqctl join_cluster rabbit@rabbitmq01
Clustering node rabbit@rabbitmq02 with rabbit@rabbitmq01 ...
...done.
#启动集群服务
[root@rabbitmq02 ~]# rabbitmqctl start_app
Starting node rabbit@rabbitmq02 ...
...done.

#在rabbitmq01上
#查看状态，对比加入前有区别，发现此时多了一个rabbitmq02节点
[root@rabbitmq01 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq01 ...
[{nodes,[{disc,[rabbit@rabbitmq01,rabbit@rabbitmq02]}]},
 {running_nodes,[rabbit@rabbitmq02,rabbit@rabbitmq01]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.

#将rabbitmq03加入到rabbitmq01集群节点
#在rabbitmq03上
#停止集群服务
[root@rabbitmq03 ~]# rabbitmqctl stop_app
Stopping node rabbit@rabbitmq03 ...
...done.
#加入到rabbitmq01的集群节点，会自动与rabbitmq02节点组件成集群
[root@rabbitmq03 ~]# rabbitmqctl join_cluster rabbit@rabbitmq01
Clustering node rabbit@rabbitmq03 with rabbit@rabbitmq01 ...
...done.
#启动集群服务
[root@rabbitmq03 ~]# rabbitmqctl start_app
Starting node rabbit@rabbitmq03 ...
...done.

#在rabbitmq01上
#查看状态，对比加入前有区别，发现此时节点数为rabbitmq02,rabbitmq03也成了集群
[root@rabbitmq01 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq01 ...
[{nodes,[{disc,[rabbit@rabbitmq01,rabbit@rabbitmq02,rabbit@rabbitmq03]}]},
 {running_nodes,[rabbit@rabbitmq03,rabbit@rabbitmq02,rabbit@rabbitmq01]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.
```

#####八.移除集群节点
#####移除当前节点
```
#在rabbitmq01上，查看删除节点前集群状态
[root@rabbitmq01 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq01 ...
[{nodes,[{disc,[rabbit@rabbitmq01,rabbit@rabbitmq02,rabbit@rabbitmq03]}]},
 {running_nodes,[rabbit@rabbitmq03,rabbit@rabbitmq02,rabbit@rabbitmq01]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.

#将集群节点rabbitmq03上移除rabbitmq03节点
#关闭集群节点
[root@rabbitmq03 ~]# rabbitmqctl stop_app
Stopping node rabbit@rabbitmq03 ...
...done.
#移除集群
[root@rabbitmq03 ~]# rabbitmqctl reset
Resetting node rabbit@rabbitmq03 ...
...done.

#在rabbitmq01上查看，发现集群状态显示已经没有rabbitmq03节点，说明已经删除
[root@rabbitmq01 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq01 ...
[{nodes,[{disc,[rabbit@rabbitmq01,rabbit@rabbitmq02]}]},
 {running_nodes,[rabbit@rabbitmq02,rabbit@rabbitmq01]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.
```

#####移除远程节点
```
#前提是远程节点以下线
#例在rabbitmq02远程下线rabbitmq01
#关闭rabbitmq01,模拟rabbitmq01已经下线
[root@rabbitmq01 ~]# rabbitmqctl stop_app
#在rabbitmq02上
[root@rabbitmq02 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq02 ...
[{nodes,[{disc,[rabbit@rabbitmq01,rabbit@rabbitmq02]}]},
 {running_nodes,[rabbit@rabbitmq02]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.

#在rabbitmq02上
#移除远程节点前，先查看下当前集群状态
[root@rabbitmq02 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq02 ...
[{nodes,[{disc,[rabbit@rabbitmq01,rabbit@rabbitmq02]}]},  #节点信息显示rabbitmq01仍在集群当中
 {running_nodes,[rabbit@rabbitmq02]},   #因集群节点rabbitmq已经下线，所以运行节点只有rabbitmq02一个
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.
#移除rabbitmq01节点
[root@rabbitmq02 ~]# rabbitmqctl forget_cluster_node rabbit@rabbitmq01
Removing node rabbit@rabbitmq01 from cluster ...
...done.
#移除远程节点后，查看下当前集群状态
[root@rabbitmq02 ~]# rabbitmqctl cluster_status                       
[root@rabbitmq02 ~]# rabbitmqctl cluster_status                       
Cluster status of node rabbit@rabbitmq02 ...
[{nodes,[{disc,[rabbit@rabbitmq02]}]},  #节点信息显示只有rabbitmq02一个节点了
 {running_nodes,[rabbit@rabbitmq02]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.

#移除rabbitmq01后，下次启动rabbitmq01时有报错，解决如下
[root@rabbitmq01 ~]# rabbitmqctl start_app
Starting node rabbit@rabbit1 ...
Error: inconsistent_cluster: Node rabbit@rabbitmq01 thinks it's clustered with node rabbit@rabbitmq02, but rabbit@rabbit2 disagrees
[root@rabbitmq01 ~]# rabbitmqctl reset
Resetting node rabbit@rabbit1 ...done.
[root@rabbitmq01 ~]# rabbitmqctl start_app
Starting node rabbit@mcnulty ...
...done.

##还有一种报错，将节点移除集群后，无法启动。,需要删除/data/rabbitmq/下的数据目录才行
```


#####九.修改集群节点类型disc为ram类型
```
[root@rabbitmq02 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq02 ...
[{nodes,[{disc,[rabbit@rabbitmq01,rabbit@rabbitmq02]}]},
 {running_nodes,[rabbit@rabbitmq01,rabbit@rabbitmq02]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.
[root@rabbitmq02 ~]# rabbitmqctl stop_app
Stopping node rabbit@rabbitmq02 ...
...done.
#修改为ram节点
[root@rabbitmq02 ~]# rabbitmqctl change_cluster_node_type ram
Turning rabbit@rabbitmq02 into a ram node ...
...done.
[root@rabbitmq02 ~]# rabbitmqctl start_app
Starting node rabbit@rabbitmq02 ...
...done.
[root@rabbitmq02 ~]# rabbitmqctl cluster_status
Cluster status of node rabbit@rabbitmq02 ...
[{nodes,[{disc,[rabbit@rabbitmq01]},{ram,[rabbit@rabbitmq02]}]},
 {running_nodes,[rabbit@rabbitmq01,rabbit@rabbitmq02]},
 {cluster_name,<<"rabbit@rabbitmq01">>},
 {partitions,[]}]
...done.
```
#####十.常用命令
```
启动集群节点
rabbitmqctl start_app
关闭集群节点
rabbitmqctl stop_app
查看集群状态
rabbitmqctl cluster_status

#关闭当前集群节点
rabbitmqctl stop
#启动当前集群节点
rabbitmq-server -detached

#添加用户，#rabbitmqctl add_user <username> <password>
rabbitmqctl add_user admin admin@123                                                                            
# 设置用户角色 
# rabbitmqctl set_user_tags admin administrator 
#授权用户权限
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```


#####十一.集群数据测试
######第一步，在rabbitmq03上编写测试脚本
```
[root@rabbitmq03 ~]# vim rabbitmq_test.py 
#!/usr/bin/env python
import os
import pika
xx=raw_input("Please:")
connection = pika.BlockingConnection(pika.ConnectionParameters('192.168.122.210'))
channel = connection.channel()
channel.queue_declare(queue='HHello')
channel.basic_publish(exchange='', routing_key='HHello', body=xx)
print " [x] Sent 'Hello World!'"
connection.close()

#插入两台测试数据，此时消息未被处理
[root@rabbitmq03 ~]# python rabbitmq_test.py 
Please:xxx
 [x] Sent 'Hello World!'
[root@rabbitmq03 ~]# python rabbitmq_test.py 
Please:ccc
 [x] Sent 'Hello World!'
```

######第二步,在rabbitmq02上，查看消息队列信息
```
[root@rabbitmq02 ~]# rabbitmqctl list_queues
Listing queues ...
HHello  2   #显示有两台未处理的消息
...done.
```

######第三步,在rabbitmq01上，编写处理消息脚本
```
[root@rabbitmq01 ~]# vim rabbitmq_recive.py 
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('192.168.122.29'))
channel = connection.channel()
channel.queue_declare(queue='HHello')


def callback(ch, method, properties, body):
    print " [x] Received %r" % (body,)

channel.basic_consume(callback, queue='HHello', no_ack=True)

print ' [*] Waiting for messages. To exit press CTRL+C'
channel.start_consuming()
[root@rabbitmq01 ~]# python rabbitmq_recive.py  #开启处理程序
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'xxx'   #处理两台消息
 [x] Received 'ccc
```

######第四步,在rabbitmq02上查看刚刚的消息是否被处理
```
[root@rabbitmq02 ~]# rabbitmqctl list_queues
Listing queues ...
HHello  0   #此时显示消息数为0，说明消息已经被处理
...done.
```

######通过以上测试可以从不同的3台节点看到数据的互通，测试了集群


