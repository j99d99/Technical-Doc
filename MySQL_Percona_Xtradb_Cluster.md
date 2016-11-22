#MySQL集群方案-Percona_Xtradb_Cluster

####测试环境:
	Centos 6.5 x86_64
####服务器IP:
	mysql-pxc01		192.168.122.207
	mysql-pxc02		192.168.122.150
	mysql-pxc03		192.168.122.131

####软件下载

	#percona-repository源
	wget http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
	#epel源
	wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm



####安装
1.卸载原有的mysql-libs依赖包
```
rpm -e `rpm -qa|grep mysql-libs` --nodeps
```
2.安装epel源并安装socat
```
rpm -vih https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
```
3.安装percona-repository源
```
rpm -vih http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
```
4.安装pxc
```
yum install -y Percona-XtraDB-Cluster-full-57
```
5.配置pxc
```
#mysql-pxc01上
vim /etc/my.cnf
[mysqld]
...
datadir=/var/lib/mysql
user=mysql
wsrep_provider=/usr/lib64/libgalera_smm.so
# Cluster connection URL contains the IPs of node#1, node#2 and node#3
wsrep_cluster_address=gcomm://192.168.122.207,192.168.122.150,192.168.122.131
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
# Node 1 address
wsrep_node_address=192.168.122.207
wsrep_sst_method=xtrabackup-v2
wsrep_cluster_name=my_centos_cluster
wsrep_sst_auth="sstuser:s3cret3s"


授权用户
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 's3cret3s';
GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
FLUSH PRIVILEGES;

#使用如下命令启动mysql
/etc/init.d/mysql bootstrap-pxc

#查看数据库状态
show status like 'wsrep%';
```
```
#mysql-pxc02上
vim /etc/my.cnf
[mysqld]
...
datadir=/var/lib/mysql
user=mysql

wsrep_provider=/usr/lib64/libgalera_smm.so
# Cluster connection URL contains the IPs of node#1, node#2 and node#3
wsrep_cluster_address=gcomm://192.168.122.207,192.168.122.150,192.168.122.131
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
# Node 2 address
wsrep_node_address=192.168.122.150
wsrep_sst_method=xtrabackup-v2
wsrep_cluster_name=my_centos_cluster
wsrep_sst_auth="sstuser:s3cret3s"

#启动mysql
/etc/init.d/mysql start

#查看数据库状态
show status like 'wsrep%';
```
```
#mysql-pxc03上
vim /etc/my.cnf
[mysqld]
...
datadir=/var/lib/mysql
user=mysql

wsrep_provider=/usr/lib64/libgalera_smm.so
# Cluster connection URL contains the IPs of node#1, node#2 and node#3
wsrep_cluster_address=gcomm://192.168.122.207,192.168.122.150,192.168.122.131
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
# Node 3 address
wsrep_node_address=192.168.122.131
wsrep_sst_method=xtrabackup-v2
wsrep_cluster_name=my_centos_cluster
wsrep_sst_auth="sstuser:s3cret3s"

#启动mysql
/etc/init.d/mysql start

#查看数据库状态
show status like 'wsrep%';
```