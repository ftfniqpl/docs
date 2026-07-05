# iterm2导入配置文件

将导出的 json文件放到 `~/Library/Application Support/iTerm2/DynamicProfiles`下即可



# iterm2自动登录并解决vim错乱

1. 安装sshpass

```
    git clone https://git.oschina.net/ftfniqpl/ubuntu1604install.git
    cd ubuntu1604install/mac
    tar -xvf sshpass-1.06.tar.gz  
    cd sshpass  
    ./configure  
    su - make install
```

2. 

```
/usr/local/bin/sshpass -p 密码 ssh -p 22 用户名@IP
```

3. 在item2的profile中使用command即可


# iterm2 ssh时提示:cannot change locale (UTF-8) No such file or directory

```
vim /etc/ssh/ssh_config找到

Host *
	SendEnv LANG LC_*

注释掉,重新登录即可
```
