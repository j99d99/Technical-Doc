#MHA+VIP实现MySQL的高可用
###环境说明:
系统:	centos6.5
mysql版本:	mysql-5.6.25

IP:
master-01:	172.16.88.151	(主)
slave-01:	172.16.88.152	(主备)
slave-02:	172.16.88.153	(从)
mha-manage:	172.16.88.154	(mha管理节点)
vip:		172.16.88.155	(虚拟IP)

###MySQL的安装和主从配置
#####mysql安装:	

```
[root@localhost ~]# wget http://cdn.mysql.com/archives/mysql-5.6/MySQL-server-5.6.25-1.el6.x86_64.rpm
[root@localhost ~]# wget http://cdn.mysql.com/archives/mysql-5.6/MySQL-client-5.6.25-1.el6.x86_64.rpm
[root@localhost ~]# rpm -vih MySQL-server-5.6.25-1.el6.x86_64.rpm 
[root@localhost ~]# rpm -vih MySQL-client-5.6.25-1.el6.x86_64.rpm
```
#####关于mysql-libs和mysql-server冲突解决办法
```
wget http://cdn.mysql.com/archives/mysql-5.6/MySQL-shared-compat-5.6.25-1.el6.x86_64.rpm
rpm -vih MySQL-shared-compat-5.6.25-1.el6.x86_64.rpm
```

#####MySQL的主从配置
master-01和slave-01以及slave-02配置为主从复制关系


#####1.配置master-01
[root@Mysql-master-01 ~]# vim /etc/my.cnf
server-id=1
log-bin=mysql-bin
binlog_format = mixed

#####2.配置slave-01
```
[root@Mysql-slave-01 ~]# vim /etc/my.cnf
server-id=2
log-bin=mysql-bin
binlog_format = mixed
log-slave-updates
relay_log_purge=0
```
#####3.重启mysql服务
#####4.添加mysql复制账号(每台都执行)
```
mysql> grant select,replication slave on *.* to 'slave'@'172.16.%' identified by 'test123';
Query OK, 0 rows affected (0.00 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
```

#####5.在mysql-master-01上操作
```
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

#####6.在mysql-slave-01上操作
```
mysql> stop slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> reset slave;
Query OK, 0 rows affected (0.00 sec)

mysql> change master to master_host='172.16.88.151',master_user='slave',master_password='test123',master_log_file='mysql-bin.000001',master_log_pos=120;
Query OK, 0 rows affected, 2 warnings (0.34 sec)

mysql> start slave;
Query OK, 0 rows affected (0.04 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.16.88.151
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 120
               Relay_Log_File: Mysql-slave-01-relay-bin.000002
                Relay_Log_Pos: 283
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
.....
1 row in set (0.00 sec)
```
当看到如下两个参数状态为Yes,即表示主从配置成功
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```
######7.slave-02配置同slave-01,当server_id不能相同

###MHA安装配置
#####一.免密钥配置
在每台之间都互通免秘钥登录
```
[root@localhost ~]# ssh-keygen -t rsa
[root@localhost ~]# ssh-copy-id -i /root/.ssh/id_rsa.pub root@$ip
#$ip指的是每台的ip
```

#####二.MHA的安装
1.软件下载
```
wget http://www.mysql.gr.jp/frame/modules/bwiki/index.php?plugin=attach&pcmd=open&file=mha4mysql-node-0.56-0.el6.noarch.rpm&refer=matsunobu
wget http://www.mysql.gr.jp/frame/modules/bwiki/index.php?plugin=attach&pcmd=open&file=mha4mysql-manager-0.56-0.el6.noarch.rpm&refer=matsunobu
```

2.安装epel源
```
rpm -vih http://ftp.linux.ncsu.edu/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```

3.NODE节点安装
```
yum install perl-DBD-MySQL
rpm -Uvh mha4mysql-node-0.56-0.el6.noarch.rpm 
```
4.MANAGE节点安装
```
yum install perl-DBD-MySQL
yum install perl-Config-Tiny
yum install perl-Log-Dispatch
yum install perl-Time-HiRes
rpm -Uvh mha4mysql-node-0.56-0.el6.noarch.rpm
rpm -Uvh mha4mysql-manager-0.56-0.el6.noarch.rpm
```

#####三.MHA的配置
1.数据库添加管理账户
```
grant all privileges on *.* to 'mha'@'172.16.%' identified by 'mha123';
```
2.mha-mamage节点配置
```
[root@MHA-manage ~]# mkdir -p /masterha/app1
[root@MHA-manage ~]# mkdir /etc/masterha
[root@MHA-manage ~]# vim /etc/masterha/app1.cnf
[server default]
user = mha
password = mha123
manager_workdir = /masterha/app1
manager_log = /masterha/app1/manager.log
remote_workdir = /masterha/app1
ssh_user = root
repl_user = slave
repl_password = test123
ping_interval = 1

[server1]
hostname = 172.16.88.151
master_binlog_dir = /var/lib/mysql
candidate_master = 1

[server2]
hostname = 172.16.88.152 
master_binlog_dir = /var/lib/mysql
candidate_master = 1

[server3]
hostname = 172.16.88.153
master_binlog_dir = /var/lib/mysql
```

3.验证ssh信任登录是否成功
```
[root@MHA-manage ~]# masterha_check_ssh --conf=/etc/masterha/app1.cnf
.......
Mon Oct 17 09:48:46 2016 - [debug] 
Mon Oct 17 09:47:05 2016 - [debug]  Connecting via SSH from root@172.16.88.152(172.16.88.152:22) to root@172.16.88.151(172.16.88.151:22)..
Mon Oct 17 09:47:56 2016 - [debug]   ok.
Mon Oct 17 09:47:56 2016 - [debug]  Connecting via SSH from root@172.16.88.152(172.16.88.152:22) to root@172.16.88.153(172.16.88.153:22)..
Mon Oct 17 09:48:46 2016 - [debug]   ok.
Mon Oct 17 09:48:46 2016 - [info] All SSH connection tests passed successfully.
[root@MHA-manage ~]# 
```

4.验证mysql复制是否成功
```
[root@MHA-manage ~]# masterha_check_repl --conf=/etc/masterha/app1.cnf 
.......
172.16.88.151(172.16.88.151:3306) (current master)
 +--172.16.88.152(172.16.88.152:3306)
 +--172.16.88.153(172.16.88.153:3306)

Mon Oct 17 11:10:42 2016 - [info] Checking replication health on 172.16.88.152..
Mon Oct 17 11:10:42 2016 - [info]  ok.
Mon Oct 17 11:10:42 2016 - [info] Checking replication health on 172.16.88.153..
Mon Oct 17 11:10:42 2016 - [info]  ok.
Mon Oct 17 11:10:42 2016 - [warning] master_ip_failover_script is not defined.
Mon Oct 17 11:10:42 2016 - [warning] shutdown_script is not defined.
Mon Oct 17 11:10:42 2016 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
```


#####四.配置通过脚本对VIP进行切换
mha-manager节点上进行配置
1.添加配置
```
[root@MHA-manage app1]# vim /etc/masterha/app1.cnf
[server default]
....
master_ip_failover_script=/etc/masterha/master_ip_failover
```
2.给master添加VIP
```
[root@MHA-manage app1]# ssh 172.16.88.151 "/sbin/ifconfig eth0:0 172.16.88.155 netmask 255.255.240.0 up"
```

3.添加master_ip_failover切换脚本
```
[root@MHA-manage app1]# vim /etc/masterha/master_ip_failover 
  
#!/usr/bin/env perl  

use strict;
use warnings FATAL => 'all';

use Getopt::Long;

my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);

my $vip = '172.16.88.155/23';
my $key = '0';
my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();

sub main {

    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

    if ( $command eq "stop" || $command eq "stopssh" ) {

        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {

        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}

sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip   
            --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
```

#####五.一些命令和测试

启动监听服务
```
[root@MHA-manage ~]# nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /masterha/app1/manager.log 2>&1 & 
```

检查状态
```
[root@MHA-manage ~]# masterha_check_status --conf=/etc/masterha/app1.cnf
app1 (pid:1311) is running(0:PING_OK), master:172.16.88.151
```
关闭服务
```
[root@MHA-manage ~]# masterha_stop --conf=/etc/masterha/app1.cnf 
Stopped app1 successfully.
```

测试说明
模拟mysql服务宕机后vip自动切换
1.启动监听
2.查看vip在哪台服务器
3.关闭master
4.查看vip是否有变化
以下是测试过程

```
[root@MHA-manage ~]# nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /manterha/app1/manager.log 2>&1 & 
[1] 6213
[root@MHA-manage ~]# masterha_check_status --conf=/etc/masterha/app1.cnf
app1 (pid:6213) is running(0:PING_OK), master:172.16.88.151
[root@MHA-manage ~]# ssh 172.16.88.151 "ip addr"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:14:6c:06 brd ff:ff:ff:ff:ff:ff
    inet 172.16.88.151/23 brd 172.16.89.255 scope global eth0
    inet 172.16.88.155/23 brd 172.16.89.255 scope global secondary eth0:0
    inet6 fe80::5054:ff:fe14:6c06/64 scope link 
       valid_lft forever preferred_lft forever
[root@MHA-manage ~]# ssh 172.16.88.152 "ip addr" 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:a1:4a:02 brd ff:ff:ff:ff:ff:ff
    inet 172.16.88.152/23 brd 172.16.89.255 scope global eth0
    inet6 fe80::5054:ff:fea1:4a02/64 scope link 
       valid_lft forever preferred_lft forever
[root@MHA-manage ~]# ssh 172.16.88.151 "service mysql stop"
Shutting down MySQL..... SUCCESS! 
[root@MHA-manage ~]# ssh 172.16.88.151 "ip addr"           
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:14:6c:06 brd ff:ff:ff:ff:ff:ff
    inet 172.16.88.151/23 brd 172.16.89.255 scope global eth0
    inet6 fe80::5054:ff:fe14:6c06/64 scope link 
       valid_lft forever preferred_lft forever
[1]+  Done                    nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /manterha/app1/manager.log 2>&1
[root@MHA-manage ~]# ssh 172.16.88.152 "ip addr"           
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:a1:4a:02 brd ff:ff:ff:ff:ff:ff
    inet 172.16.88.152/23 brd 172.16.89.255 scope global eth0
    inet 172.16.88.155/23 brd 172.16.89.255 scope global secondary eth0:0
    inet6 fe80::5054:ff:fea1:4a02/64 scope link 
       valid_lft forever preferred_lft forever
```
从上可以看出,当master的mysql服务关闭后,VIP已经切换到172.16.88.152上


对于MHA,当master宕机后,MHA会将slave中的一台提升为master,旧的master将被剔除,后续恢复只能当做新master的slave使用,在mha剔除旧master的同时,会将/etc/masterha/app1.cnf配置文件内关于旧master的配置段也剔除,同时MHA监听服务也会自动关闭,下次开启监听服务前,需保证/mansterha/app1/目录下的app1.failover.complete和saved_master_binlog_from_172.16.88.151_3306_20161018133124.binlog文件删除


由于每个slave上设置了参数relay_log_purge=0，所以slave节点需要定期删除中继日志，建议每个slave节点删除中继日志的时间错开
例:
```
[root@Mysql-slave-01 ~]# corntab -e
0 5 * * *  /usr/bin/purge_relay_logs --user=$user--password=$passwd --port=3306 --disable_relay_log_purge >> /var/lib/mysql/purge_relay.log  2>&1
```
```
[root@Mysql-slave-02 ~]# corntab -e
0 5 * * *  /usr/bin/purge_relay_logs --user=$user--password=$passwd --port=3306 --disable_relay_log_purge >> /var/lib/mysql/purge_relay.log  2>&1
```