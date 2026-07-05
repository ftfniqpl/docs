# mysql5.7及8.0远程授权

```
修改/etc/my.conf, 添加
bind-address=0.0.0.0


你想使用powerall从任何主机连接到mysql服务器的话。 
    GRANT ALL PRIVILEGES ON *.* TO '用户名'@'授权的IP' identified by '123345'  WITH GRANT OPTION ; 
修改密码
    update user set password=PASSWORD('123456') where user='root';
    flush tables;
    flush privileges;


mysql 8.0 的远程授权   
    CREATE USER 'root'@'%' IDENTIFIED BY 'root';
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
```

# mysql5.7修改root密码

```
1. 通过mysql免密码登陆
在其配置文件/etc/my.cnf中加入skip-grant-tables=1即可
重启mysql，使用mysql命令即可进入
use mysql
update user set authentication_string = password("123456") where user="root";
flush privileges;

# 或者将root密码清空 update user set authentication_string = NULL where user = 'root';

然后将/etc/my.cnf中的skip-grant-tables=1注释掉，重启mysql服务即可。

在此要注意的是，之前版本密码修改字段为password，在5.7版本之后字段为authentication_string

#查看临时密码:     
grep 'temporary password' /var/log/mysqld.log

#mysql 5.7 之后修改密码:      
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';

5.7之后的密码校验 可以关闭
在mysqld下面加入“validate-password=0”,然后重启mysql即可。

如果提示错误:You must reset your password using ALTER USER statement before executing this statement

使用这个命令 alter user user() identified by 'root';

```

# mysql 8.0 修改默认字符   

```
[mysql]
# 设置mysql客户端默认编码
default-character-set=utf8mb4

[mysqld]
# 允许最大连接
max_connections=200
# 服务端默认utf8编码
character-set-server=utf8mb4
# 默认存储
default-storage-engine=INNODB

[client]
#设置客户端编码
default-character-set=utf8mb4

#重启后查看
show variables like 'character%'
```

# mysql8.0 修改认证协议 支持旧版本客户端   

```
1. 查看mysql使用的认证协议
    use mysql;
    select user,host,plugin,authentication_string from user;
2. 修改my.cnf文件，添加
    [mysqld]
    default_authentication_plugin=mysql_native_password
3. 重启mysql
    brew services restart mysql
4. 修改以前的帐号的认证方式
    ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'passwd';
    FLUSH PRIVILEGES;
```

