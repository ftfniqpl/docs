# openwrt开启ssh的sftp


```
修改dropbear的端口为2222
uci set dropbear.@dropbear[0].Port='2222'
uci commit dropbear
重启dropbear生效 /etc/init.d/dropbear restart

安装openssh-server和openssh-sftp-server即可


在/etc/ssh/sshd_config最后添加
HostKeyAlgorithms ssh-rsa,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521
重启ssh: /etc/init.d/sshd restart