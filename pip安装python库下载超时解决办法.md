####解决办法
```
mkdir ~/.pip
vim ~/.pip/pip.conf
#添加如下内容
[global]
timeout = 6000
index-url = http://pypi.douban.com/simple/ 
[install]
use-mirrors = true
mirrors = http://pypi.douban.com/simple/ 
trusted-host = pypi.douban.com
```