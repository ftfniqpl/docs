# centos7使用ius源安装mysql及python3

Jasmy is unstoppable. Striked into Third Target. I'm getting many Dms that you're enjoying this amazing run on Bitcoin and Jasmy. Keep sending me DMs, your profits are that gives me feels of satisfaction.

### centos下通用第三方库IUS
```
curl https://setup.ius.io | sh
```


 2. 在mysql server配置文件上增加    

```
[mysqld]
server_id=1
log-bin=master
binlog_format=row
```

 3. 配置mysql 增加maxwell用户     

```
mysql> CREATE USER 'maxwell'@'%' IDENTIFIED BY 'XXXXXX';
mysql> GRANT ALL ON maxwell.* TO 'maxwell'@'%';
mysql> GRANT SELECT, REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'maxwell'@'%';
```

 4. 安装nginx , redis      
```
yum install nginx redis java-1.8.0-openjdk
systemctl enable nginx redis
systemctl start nginx redis
```
     
 5.  安装python3.6 
```
yum install -y https://repo.ius.io/ius-release-el7.rpm
# http://npm.taobao.org/mirrors/python/  这个地址可以下载 镜像包
yum install -y python36u python36u-libs python36u-devel python36u-pip
yum install gcc-c++
yum install python-setuptools -y
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
python2 get-pip.py install
pip2 install pip -U
pip2 install pbr
pip2 install virtualenvwrapper
pip3.6 install pip -U
pip3.6 install virtualenvwrapper
pip3.6 install redis paho-mqtt
```

6. 安装supervisor
```
yum install supervisor
修改 /etc/supervisord.conf最后一行的.ini 为conf
systemctl enable supervisord
```


7. 安装mysql客户端

```
rpm -ivh https://repo.mysql.com/mysql57-community-release-el7-11.noarch.rpm
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
yum search mysql-community
```


如果pip安装出现超时的问题

```
在pip后面添加 -i https://mirrors.aliyun.com/pypi/simple/
```

