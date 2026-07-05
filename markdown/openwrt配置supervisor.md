# openwrt配置supervisor


```
opkg update
mv /usr/lib/libpcap.so.1 /usr/lib/libpcap.so.1.1
opkg install curl tmux tcpdump libwolfssl node-npm
wget https://bootstrap.pypa.io/get-pip.py
python3 get-pip.py install
/usr/bin/python3 -m pip install --upgrade pip
pip install supervisor
supervisor安装完成后，会在/usr/bin/目录生成3个可执行文件
```

```
$ ls -lh /usr/bin/*supervisor*
-rwxr-xr-x    1 root     root         218 Aug  8 14:46 /usr/bin/echo_supervisord_conf
-rwxr-xr-x    1 root     root         223 Aug  8 14:46 /usr/bin/supervisorctl
-rwxr-xr-x    1 root     root         221 Aug  8 14:46 /usr/bin/supervisord
```

2. 生成supervisor配置文件
```
mkdir -p /etc/supervisor/conf.d
cd /etc/supervisor/
/usr/bin/echo_supervisord_conf > ./supervisord.conf
```

supervisor目录结构：
```
root@OpenWrt:/etc/supervisor# tree
.
├── conf.d
└── supervisord.conf
```

3. 修改supervisor配置文件
```
#配置外部文件加载路径
[include]
files = /etc/supervisor/conf.d/*.conf
配置supervisor开机启动脚本

cat /etc/init.d/supervisord

#!/bin/sh /etc/rc.common
# Start/stop/restart supervisor in OpenWrt.

START=91
USE_PROCD=0
PROG=/usr/bin/supervisord
DAEMON=${PROG}
# Location of the pid file
PIDFILE=/tmp/supervisord.pid
# Config of supervisor
CONFIG=/etc/supervisor/supervisord.conf
start_service() {
    # $DAEMON -c $CONFIG -j $PIDFILE
    procd_open_instance
    procd_set_param command $PROG -c $CONFIG -j $PIDFILE
    procd_set_param respawn
    procd_close_instance
    touch $CONFIG
}
stop_service() {
    kill $(cat $PIDFILE)
}
```

4. 自启supervisor
```
chmod +x  /etc/init.d/supervisord
/etc/init.d/supervisord enable
```
