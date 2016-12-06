#ftps安装文档
#####1.安装vsftpd服务
```
yum install -y vsftpd
```
#####2.关闭selinux和防火墙

#####3.新建ftp账号
```
useradd -s /sbin/nologin -d /data/ftpstest vsftptest
```
#####4.配置vsftpd
```
#1.建立登录用户列表文件
cd /etc/vsftpd
vim vsftp.user
#添加刚刚新建的ftp账号
vsftpjiuyun

#2.生成证书
mkdir /etc/vsftpd/ssl -p
mkdir /etc/vsftpd/ssl/private -p
ssl证书生成
openssl req -x509 -nodes -days 720 -newkey rsa:2048 -keyout /etc/vsftpd/ssl/private/vsftpd.key -out /etc/vsftpd/ssl/vsftpd.pem
Country Name (2 letter code) [XX]:CN 	#国家
State or Province Name (full name) []:Zhejiang   #省份
Locality Name (eg, city) [Default City]:Hangzhou 	#城市
Organization Name (eg, company) [Default Company Ltd]:RD 	#公司
Organizational Unit Name (eg, section) []: 	#直接回车
Common Name (eg, your name or your server's hostname) []:rdtest.com 	#主机名
Email Address []:mjin@erongdu.com 	#邮箱

#3.修改vsftpd.conf配置文件
vim vsftpd.conf
修改
anonymous_enable=YES
为
anonymous_enable=NO
修改
listen=NO
为
listen=YES
#如果没有则添加如下内容
pam_service_name=vsftpd
userlist_enable=YES
chroot_local_user=YES
userlist_deny=NO
userlist_file=/etc/vsftpd/vsftp.user
tcp_wrappers=YES

ssl_enable=YES
allow_anon_ssl=NO
ssl_sslv2=YES
ssl_sslv3=YES
ssl_tlsv1=YES
require_ssl_reuse=NO
force_local_logins_ssl=YES
force_local_data_ssl=YES
rsa_cert_file=/etc/vsftpd/ssl/vsftpd.pem
rsa_private_key_file=/etc/vsftpd/ssl/private/vsftpd.key
#4.启动vsftpd服务
systemctl enable vsftpd
systemctl start vsftpd
```
