# Ubuntu 24.04 安装FileManager服务

1. ```systemctl disable --now unattended-upgrades```

2. `sudo apt-get update -y`

3. `sudo apt remove cloud-init -y`

4. `sudo apt install vim tree net-tools iputils-ping software-properties-common -y`

5. 开启root登录,方便传代码

   ```
   修改/etc/ssh/ssd_config
   找到PermitRootLogin行, 修改为 PermitRootLogin yes
   systemctl daemon-reload
   设置root的密码: passwd root
   ```

6. 修改网卡等待时间:

   ```
   vim /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service 
   在Service段中添加 TimeoutStartSec=2sec
   systemctl daemon-reload
   ```

7. 修改pip为国内源

   ```
   1. 在/home/linux下创建.pip文件夹与pip.conf文件
   2. mkdir -p /home/linux/.pip
   3. vim /home/linux/.pip/pip.conf
   	[global]
   	index-url = https://mirrors.aliyun.com/pypi/simple/
   	[install]
   	trusted-host=mirrors.aliyun.com
   4. mkdir -p /root/.pip
   5. cp /home/linux/.pip/pip.conf /root/.pip/pip.conf
   ```

8. 安装zerotier

   ```
   curl -s https://install.zerotier.com | sudo bash
   
   systemctl enable zerotier-one
   zerotier-cli join 93afae5963a766fc
   mkdir /var/lib/zerotier-one/moons.d
   wget https://rowsea.com/000000b5b91d9c08.moon
   mv 000000b5b91d9c08.moon /var/lib/zerotier-one/moons.d/
   zerotier-cli orbit b5b91d9c08 b5b91d9c08
   systemctl restart zerotier-one
   ```

9. 安装软raid

   ```
   1. apt install mdadm quota
   2. 检查磁盘状态,确认哪些磁盘未被使用，例如 `/dev/sdb /dev/sdc /dev/sdd`
   	lsblk
   	fdisk -l
   3. 清除磁盘旧配置
   	sudo mdadm --zero-superblock /dev/sd[b-d]
   4. 创建raid阵列
   	sudo mdadm --create --verbose /dev/md0 --level=10 --raid-devices=4 /dev/sd[b-e]
   5. 创建文件系统并挂载
   	sudo mkfs.ext4 -O project,quota /dev/md0
   	sudo mkdir -p /mnt/raid
   	sudo mount /dev/md0 /mnt/raid
   6. 持久化配置
   	sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
   	sudo update-initramfs -u
   7. 添加到 `/etc/fstab`：
   	echo '/dev/md0 /mnt/raid ext4 defaults,nofail,discard,prjquota 0 0' | sudo tee -a /etc/fstab
   8.  开启磁盘路径配额
   	quotaon -P /mnt/raid
   9. 查看是否开启磁盘路径配额
   	quotaon -Ppv /mnt/raid
   ```

10. 安装smartctl

   ```
   apt install g++ gcc make -y
   tar -zxvf smartmontools-7.5.tar.gz
   cd smartmontools-7.5
   ./configure --disable-dependency-tracking
   make && make install
   /usr/local/sbin/smartctl -a /dev/sda
   ```

11. 双网卡指定网段走指定网卡网关方法

    ```
    route add default gw 10.129.168.1 # 添加默认网关10.129.168.1
    route add -net 10.129.168.0/21 gw 10.129.168.1 eth0 # 10.129.168.0网段走eth0网卡
    # 删除路由 route del -net 10.129.168.0/21 gw 10.129.168.1
    win10添加双网卡路由
    route print #查看网卡编号(第一列)
    route add 10.129.168.0 mask 255.255.248.0 10.129.168.1 if <网卡编号> -p
    #删除路由 route delete 10.129.168.0
    ```

12. 安装samba

    ```
    apt install samba
    systemctl enable smbd
    mkdir -p /etc/samba/users/
    
    /etc/samba/smb.conf配置文件修改为以下内容:
    [global]
       workgroup = WORKGROUP
       server string = %h server (Samba, Ubuntu)
       log file = /var/log/samba/log.%m
       max log size = 1000
       logging = file
       security = user
       ;map to guest = bad user
       #netbios name = Share Server
       max connections = 1000
       dns proxy = no
       min protocol = NT1
       # 隐藏特殊文件,如socket, 设备文件, FIFO等
       hide special files = yes 
       # 隐藏没有读取权限的文件
       hide unreadable = yes
       ;vfs objects = trim_perms full_audit
       ;full_audit:prefix = %u
       ;full_audit:syslog = true
       ;full_audit:success = create_file mkdirat renameat mknodat unlinkat pwrite_recv pwrite_send pread_recv pread_send
       include = /etc/samba/users/%U.share.conf
       以下解决中文乱码
       dos charset = UTF-8
       unix charset = utf-8
       display charset = utf-8
    ```

13. 安装supervisor,开启python服务

    ```
    apt install python3-pip python3-flask supervisor -y
    systemc	enable supervisor
    
    修改vim /etc/supervisor/conf.d/pysmb.conf
    [program:pysmb]
    directory=/mnt/raid/www
    command=/usr/bin/python3 pysmb.py
    process_name=pysmb
    autostart=true
    autorestart=true
    startretries=10
    redirect_stderr=true
    stdout_logfile=/tmp/pysmb.log
    
    重新加载supervisorctl reload
    查看supervisorctl status
    ```

14. 安装PHP7.4

    ```
    sudo add-apt-repository ppa:ondrej/php
    sudo apt update
    sudo apt install -y php7.4 php7.4-cli php7.4-fpm php7.4-mysql php7.4-xml php7.4-curl php7.4-zip php7.4-mbstring php7.4-gd php7.4-iconv
    
    修改/etc/php/7.4/php.ini中 以下字段的值
    upload_max_filesize = 20000M
    max_execution_time = 60
    memory_limit = 256M
    doc_root = /mnt/raid/www
    
    修改/etc/php/7.4/php-fpm.d/www.conf中以下字段的值
    listen = /dev/shm/php-fpm.sock
    
    systemctl enable php7.4-fpm
    ```

15. 安装nginx

    ```
    apt install nginx
    在/etc/nginx/nginx.conf中 include /etc/nginx/conf.d/*.conf上方添加以下字段
    charset 'utf-8';
    server_tokens off;
    
    注释 include /etc/nginx/sites-enabled/*;
    
    在/etc/nginx/conf.d/下新建file.conf文件,添加以下内容
    server {
        server_name default_server;
        listen 80;
    
        root /mnt/raid/www;
    
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        resolver 8.8.8.8 ipv6=off;
        client_max_body_size 20000M;
        client_body_buffer_size 128k;
    
        # log files
        access_log /var/log/nginx/file_access.log;
        error_log /var/log/nginx/file_error.log;
    
        index index.php index.html;
    
        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }
    
        location /favicon.ico {
            root /mnt/raid/www;
        }
    
        location ~ \.(php|phar)(/.*)?$ {
    	    try_files $uri =404;
            fastcgi_split_path_info ^(.+\.(?:php|phar))(/.*)$;
    
            fastcgi_intercept_errors on;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_pass unix:/dev/shm/php-fpm.sock;
        }
    
        location /static {
            root /mnt/raid/www;
        }
    }
    
    systemctl enable nginx
    ```

16. 上传代码

    ```
    mkdir -p /mnt/raid/www
    开始上传代码
    chown -R www-data:root /mnt/raid/www
    ```

17. 测试

    ```
    1. 查看python服务是否启动
    2. 查看nginx, samba等服务
    ```

18. 安装ttyd服务

    ```
    mv ttyd.x86_64 /usr/local/bin/ttyd
    chmod -R 755 /usr/local/bin/ttyd 
    /usr/local/bin/ttyd -W -p 8082 login
    
    vim /etc/systemd/system/ttyd.service
    [Unit]
    Description=ttyd - Web-based Terminal
    After=network.target
    
    [Service]
    User=root # 以登录的用户身份运行，或者指定 root
    # 如果是使用 root 权限，则使用以下这一行，通常不需要额外参数
    ExecStart=/usr/local/bin/ttyd -W -p 7681 login # 如果是bash就直接登录
    # 如果需要指定具体的用户和工作目录，可以使用下面这一行（示例）
    # ExecStart=/usr/local/bin/ttyd -W -i 127.0.0.1 -p 7681 -U -l /home/user -c user:password bash
    Restart=always
    RestartSec=3
    
    [Install]
    WantedBy=multi-user.target
    
    systemctl daemon-reload
    systemctl enable ttyd
    systemctl start ttyd
    ```

19. nginx配置

    ```
    location /ssh {
    	#auth_basic "Restricted Access";
    	#auth_basic_user_file /etc/nginx/.htpasswd; 
        proxy_pass http://127.0.0.1:7681/;
    	proxy_set_header Host $host;
    	proxy_set_header X-Forwarded-Proto $scheme;
    	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    	proxy_http_version 1.1;
    	proxy_set_header Upgrade $http_upgrade;
    	proxy_set_header Connection "upgrade";
    }
    ```

20. 双网卡添加持久化路由

    ```
    vim /etc/systemd/system/static-route.service
    
    [Unit]
    Description=Static Route Configuration
    After=network.target
    
    [Service]
    Type=oneshot
    ExecStart=/sbin/ip route add 10.0.0.0/8 via 10.122.44.1 dev enp2s0
    RemainAfterExit=yes
    
    [Install]
    WantedBy=multi-user.target
    
    systemctl enable static-route
    systemctl start static-route
    ```

21. 

