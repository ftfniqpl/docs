# Centos7通过IUS安装LNMP环境

本文主要适用 CentOS 7，通过添加 IUS 软件源手动安装尽可能新的 LNMP 建站环境。对于新手，建议在全新系统环境下操作，并先运行 `yum -y update` 更新系统软件包，以及一些可选系统基础配置。

#### 前言

注 1：IUS 软件包名称带版本号，随着时间流逝，文中安装命令请相应修改，以确保安装最新版软件。

注 2：IUS 跟随上游软件支持周期，当某个版本结束支持后，IUS 会在一个月后将其移至存档库。如果要安装已结束支持的软件版本，请使用 `yum --enablerepo="ius-archive" install php70u-fpm` 安装命令（自行替换其中软件包名，下同）。如果要在存档库里搜索软件，则用 `yum --enablerepo="ius-archive" search all php70u` 命令。

注 3：为了方便，下文使用了 sed 命令修改配置文件。这种修改方式依赖匹配文本，如果文本有差别可能导致修改失败。建议在部署生产环境时用编辑器手动修改，或在修改后验证修改，以免漏掉配置。

##### 添加 IUS 软件源

IUS 里有许多软件包与 EPEL 存在依赖项，因此需要同时添加 EPEL 源。
```
# 添加 EPEL 软件源
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

# 添加 IUS 软件源
yum -y install https://repo.ius.io/ius-release-el7.rpm
```
添加后可以用下面命令检查软件源启用情况。

```
# 安装查询工具
yum -y install yum-utils

# 列出所有软件源
yum repolist all

# 列出已启用软件源
yum repolist enabled

# 列出已禁用软件源
yum repolist disabled
```

##### 安装 Nginx

EPEL 软件源里有 Nginx，并且版本不是太旧，这里直接从 EPEL 安装。
```
yum -y install nginx
```
安装后做以下基本配置。

修改 `/etc/nginx/nginx.conf` 配置文件。在 `http{...}` 内添加和修改下面红色参数，以关闭 Nginx 版本号输出，延长 fastcgi 进程超时时间（避免 504 Gateway Time-out 问题），以及设置合适 buffer 值。
```
http {
    ...
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   120s;
    keepalive_requests  1000;
    types_hash_max_size 4096;
    fastcgi_buffers 8 16k;
    fastcgi_buffer_size 16k;
    fastcgi_busy_buffers_size 24k;
    fastcgi_read_timeout 300s;
    server_tokens off;
    ...
}
```
继续修改这个配置文件，设置禁止以 IP 访问 Default Server。此外，还可以按实际需要添加连接数/请求数/连接传输速度限制和创建 CDN IP 列表配置文件。

修改 FastCGI 配置文件 SERVER_SOFTWARE 参数，删除版本号输出。
```
sed -i 's|nginx/$nginx_version|nginx|' /etc/nginx/fastcgi.conf
sed -i 's|nginx/$nginx_version|nginx|' /etc/nginx/fastcgi_params
```

`firewall-cmd --state` 检查是否安装启用防火墙。如果没有，用下面命令安装启用。

```
yum -y install firewalld
systemctl start firewalld
systemctl enable firewalld
```
防火墙放行 HTTP HTTPS 端口。
```
firewall-cmd --permanent --zone=public --add-service={http,https}
firewall-cmd --reload
```
创建一个 nginx 配置文件，方便后续申请 Let’s Encrypt 证书使用。
```
# 创建 nginx 片段配置存储目录
mkdir -p /etc/nginx/snippets

# 创建 Let’s Encrypt 证书申请目录验证配置文件
cat > /etc/nginx/snippets/letsencrypt-acme-challenge.conf << "EOF"
# Configuration file for Let's Encrypt ACME Challenge location
# Sources: https://community.letsencrypt.org/t/5622

location ^~ /.well-known/acme-challenge/ {
    default_type "text/plain";
    root         /var/www/letsencrypt;
}

location = /.well-known/acme-challenge/ {
    return       404;
}
EOF

# 创建 Let’s Encrypt 申请证书验证目录
mkdir -p /var/www/letsencrypt

# 安装 SELinux 环境管理工具
yum -y install policycoreutils-python

# 添加 SELinux 读写安全上下文
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/letsencrypt(/.*)?"
restorecon -R -v /var/www/letsencrypt

# 将目录所有者改为 nginx 用户/组
chown -R nginx:nginx /var/www/letsencrypt
```
创建开启 Gzip 压缩配置文件，以方便在站点配置文件里引用。

```
cat > /etc/nginx/snippets/enable-gzip-compression.conf << "EOF"
# Enable Gzip Compression
# Documentation: https://nginx.org/en/docs/http/ngx_http_gzip_module.html
gzip            on;
gzip_buffers    8 16k;
gzip_comp_level 5;
gzip_min_length 1000;
gzip_proxied    any;
gzip_types      text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;
gzip_vary       on;
EOF
```

修改 Nginx 服务 logrotate 日志分割配置文件 /etc/logrotate.d/nginx，调整日志分割参数，这里设置每天分割日志，保留最近 30 天日志文件。
```
/var/log/nginx/*.log {
    daily
    create
    dateext
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```
运行 Nginx 服务并设置开机启动。
```
systemctl start nginx
systemctl enable nginx
```
#### 安装 MariaDB

CentOS 默认软件源里的 MariaDB 版本太旧，这里从 IUS 源安装。
```
软件查询方法：先用 yum search 关键词 搜索相关软件，知道软件名后再安装，查看软件详情用 yum info 关键词 查看。如果仅限 IUS 源里搜索，则使用 yum --disablerepo="*" --enablerepo="ius" search all 关键词 命令。
```

```
yum -y install mariadb104-server
```
运行 MariaDB 并设置开机启动。
```
systemctl start mariadb
systemctl enable mariadb
```
运行初始化脚本设置数据库 root 密码，以及删除默认测试账户和数据库。
```
mysql_secure_installation
```
提示输入 root 密码，由于我们刚安装完没有设置，直接按 Enter 进入下一步。
```
Enter current password for root (enter for none):
```
按提示设置 MariaDB root 账户密码，选择 Y 回车后输入两遍密码完成设置。
```
Set root password? [Y/n]
```
接下来的几个选项都选Y，这将删除测试账户和数据库，以及禁用远程 root 登录并刷新权限表。
```
Remove anonymous users? [Y/n]
Disallow root login remotely? [Y/n]
Remove test database and access to it? [Y/n]
Reload privilege tables now? [Y/n]
```
至此，MariaDB 初始化安全配置完成。

#### 安装 PHP
PHP 也从 IUS 安装。下面命令安装了 FastCGI， MySQL 及其它常用 PHP 模块，过程中会将 PHP 核心软件包作为依赖项安装。
```
yum --enablerepo="ius-archive" -y install php74-fpm php74-mysqlnd php74-common php74-cli php74-gd php74-opcache php74-mbstring php74-json php74-xml php74-pecl-zip php74-bcmath php74-soap php74-xmlrpc php74-devel
```
更改 PHP 配置文件，调整一些常见参数值。
```
# 启用 cgi.fix_pathinfo 并将值设为 0（防范 PHP 解释器被欺骗执行恶意文件）
sed -i 's|^;cgi.fix_pathinfo.*|cgi.fix_pathinfo=0|' /etc/php.ini

# 将文件上传大小限制增加至 20M 和 80M
sed -i 's|^upload_max_filesize.*|upload_max_filesize = 20M|' /etc/php.ini
sed -i 's|^post_max_size.*|post_max_size = 80M|' /etc/php.ini

# 将每个脚本内存最大用量增加至 256M
sed -i 's|^memory_limit.*|memory_limit = 256M|' /etc/php.ini

# 设置记录错误日志类型
sed -i 's|^error_reporting.*|error_reporting = E_WARNING \& E_ERROR|' /etc/php.ini

# 定义日期函数默认时区
sed -i 's|^;date.timezone.*|date.timezone = Asia\/Shanghai|' /etc/php.ini

# 关闭 PHP 版本号输出
sed -i 's|^expose_php.*|expose_php = Off|' /etc/php.ini
```

更改 PHP-FPM 配置文件，调整一些常见参数值。
```
# 将用户/组改为 nginx
sed -i 's|^user =.*|user = nginx|' /etc/php-fpm.d/www.conf
sed -i 's|^group =.*|group = nginx|' /etc/php-fpm.d/www.conf

# 将通信方式改为 UNIX Socket
sed -i '/listen = 127.0.0.1:9000/ s/^/;/' /etc/php-fpm.d/www.conf
sed -i '/;listen = \/run\/php-fpm\/www.sock/ s/^;//' /etc/php-fpm.d/www.conf

# 为 UNIX Socket 添加 Nginx 用户权限（需 POSIX ACL 支持，CentOS 7 默认启用。如不支持请设置 listen.owner、listen.group 和 listen.mode 参数，以确保 Nginx 有权限读写 UNIX Socket）
sed -i '/;listen.acl_users = nginx/ s/^;//' /etc/php-fpm.d/www.conf

# 将工作进程 stdout 和 stderr 重定向到错误日志
sed -i '/;catch_workers_output = yes/ s/^;//' /etc/php-fpm.d/www.conf

# 设置日志级别记录错误
sed -i 's|^;log_level.*|log_level = error|' /etc/php-fpm.conf

# 将 php-fpm 日志目录用户/组改为 nginx
chown -R nginx:nginx /var/log/php-fpm
```
创建 FastCGI 默认接收方式配置文件（CentOS 8 的 php-fpm 版本安装后自带这个文件）。
```
cat > /etc/nginx/conf.d/php-fpm.conf << "EOF"
# PHP-FPM FastCGI server
# network or unix domain socket configuration

upstream php-fpm {
        server unix:/run/php-fpm/www.sock;
}
EOF
```
创建 PHP 脚本传递到 FastCGI 的默认配置文件（CentOS 8 的 php-fpm 版本安装后自带这个文件）。
```
cat > /etc/nginx/default.d/php.conf << "EOF"
# pass the PHP scripts to FastCGI server
#
# See conf.d/php-fpm.conf for socket configuration
#
index index.php index.html index.htm;

location ~ \.(php|phar)(/.*)?$ {
    fastcgi_split_path_info ^(.+\.(?:php|phar))(/.*)$;

    fastcgi_intercept_errors on;
    fastcgi_index  index.php;
    include        fastcgi_params;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    fastcgi_param  PATH_INFO $fastcgi_path_info;
    fastcgi_pass   php-fpm;
}
EOF
```
设置 PHP 会话存储目录等（这里新建目录使用是为避免日后软件更新后需重新设置目录权限）。
```
# 设置使用新的目录
sed -i 's|/var/lib/php/fpm/session|/var/lib/custom-php-fpm/session|' /etc/php-fpm.d/www.conf
sed -i 's|/var/lib/php/fpm/wsdlcache|/var/lib/custom-php-fpm/wsdlcache|' /etc/php-fpm.d/www.conf
sed -i 's|/var/lib/php/fpm/opcache|/var/lib/custom-php-fpm/opcache|' /etc/php-fpm.d/www.conf

# 创建目录并修改用户组为 nginx，以及为目录设置合适权限
mkdir -p /var/lib/custom-php-fpm/session /var/lib/custom-php-fpm/wsdlcache /var/lib/custom-php-fpm/opcache
chgrp -R nginx /var/lib/custom-php-fpm/session /var/lib/custom-php-fpm/wsdlcache /var/lib/custom-php-fpm/opcache
chmod -R 0755 /var/lib/custom-php-fpm
chmod -R 0770 /var/lib/custom-php-fpm/session /var/lib/custom-php-fpm/wsdlcache /var/lib/custom-php-fpm/opcache

# 设置合适的 SELinux 安全上下文
semanage fcontext -a -t httpd_var_lib_t "/var/lib/custom-php-fpm(/.*)?"
semanage fcontext -a -t httpd_var_run_t "/var/lib/custom-php-fpm/session(/.*)?"
semanage fcontext -a -t httpd_var_run_t "/var/lib/custom-php-fpm/wsdlcache(/.*)?"
restorecon -R -v /var/lib/custom-php-fpm
```
启动 php-fpm 服务并设置开机自启。
```
systemctl start php-fpm
systemctl enable php-fpm
```

#### LNMP 运行环境测试
至此，LNMP 环境已搭建完成。下面试着安装一个 WordPress 看能否运行。

###### 创建网站目录和设置合适权限
```
# 创建网站目录
mkdir -p /var/www/example

# 下载 WordPress 程序并解压到目录
wget -P /var/www/example https://wordpress.org/latest.tar.gz
tar -zxvf /var/www/example/latest.tar.gz --strip-components 1 -C /var/www/example
rm -f /var/www/example/latest.tar.gz

# 设置 SELinux 读写安全上下文，以允许 httpd 进程读写网站目录/文件
# WordPress 完整功能运行有不少目录/文件需要读写，细分设置比较麻烦，这里采用递归设置
# 对于不需要读写的目录/文件，可以设置 httpd_sys_content_t 只读安全上下文，这样安全性更好
# 如果要删除设置的安全上下文，将下面 -a 参数改为 -d 并去掉 -t 和后面指定的安全上下文，然后再运行
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/example(/.*)?"
restorecon -R -v /var/www/example

# 允许 HTTPD 脚本和模块连接网络（不设置 WordPress 将无法在线更新和安装主题/插件）
setsebool -P httpd_can_network_connect 1
# 允许 HTTPD 对可写可执行内存分配（可解决 php-fpm 可写可执行内存分配拒绝错误，通常非必须）
setsebool -P httpd_execmem 1

# 将目录所有者改为 nginx 用户/组
chown -R nginx:nginx /var/www/example
```

###### 申请 Let’s Encrypt 免费证书

设置好域名解析后，创建网站配置文件（内容如下，这里简单使域名可访问就行）。
```
cat > /etc/nginx/conf.d/example.conf << "EOF"
server {
    listen      80;
    listen      [::]:80;
    server_name example.com www.example.com;
    root        /var/www/example;
    include     /etc/nginx/snippets/letsencrypt-acme-challenge.conf;
}
EOF
```
刷新 Nginx 服务，使配置生效。
```
nginx -s reload
```
安装 ACME 客户端申请 SSL 证书，这里用热门的 acme.sh。
```
# 安装后断开 SSH 连接重新登录
curl https://get.acme.sh | sh

# 设置默认申请 Let’s Encrypt 证书
# 2021-10-01 更新：建议更换使用 ZeroSSL 证书，旧设备兼容性较好
acme.sh --set-default-ca --server letsencrypt
```
申请域名 SSL 证书。
```
acme.sh --issue -d example.com -d www.example.com -w /var/www/letsencrypt
```
将证书/密钥安装到相关目录，并重启 Nginx 服务。
```
acme.sh --install-cert -d example.com -d www.example.com \
--key-file       /etc/pki/tls/private/example.com.key \
--fullchain-file /etc/pki/tls/certs/example.com.cer \
--reloadcmd      "systemctl force-reload nginx"
```
生成 DH 会话密钥。这与上面的域名密钥不同，它可以多个站点共用。
```
openssl dhparam -out /etc/pki/tls/certs/dhparam.pem 2048
```

###### 修改 Nginx 网站配置文件
```
vi /etc/nginx/conf.d/example.conf
```
内容如下（为方便理解添加了中文注释，使用时建议删除）。
```
# HTTP 重定向到 HTTPS
server {
    listen      80;
    listen      [::]:80;
    server_name example.com www.example.com;
    return 301  https://www.example.com$request_uri;
}

# 根域名重定向到 WWW
server {
    listen      443 ssl http2;
    listen      [::]:443 ssl http2;
    server_name example.com;
    return 301  https://www.example.com$request_uri;

    # 证书文件，依序为 证书链+域名证书，域名证书密钥，DH 会话密钥
    ssl_certificate           /etc/pki/tls/certs/example.com.cer;
    ssl_certificate_key       /etc/pki/tls/private/example.com.key;
    ssl_dhparam               /etc/pki/tls/certs/dhparam.pem;

    # SSL 会话，协议版本，加密算法等参数
    ssl_buffer_size           4k;
    ssl_session_timeout       10m;
    ssl_session_cache         shared:SSL:10m;
    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers               ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_stapling              on;
    ssl_stapling_verify       on;
    ssl_trusted_certificate   /etc/pki/tls/certs/example.com.cer;
}

# WWW 主站点
server {
    listen      443 ssl http2;
    listen      [::]:443 ssl http2;
    server_name www.example.com;
    root        /var/www/example;
    index       index.html index.htm index.php;

    # 证书文件，依序为 证书链+域名证书，域名证书密钥，DH 会话密钥
    ssl_certificate           /etc/pki/tls/certs/example.com.cer;
    ssl_certificate_key       /etc/pki/tls/private/example.com.key;
    ssl_dhparam               /etc/pki/tls/certs/dhparam.pem;

    # SSL 会话，协议版本，加密算法等参数
    ssl_buffer_size           4k;
    ssl_session_timeout       10m;
    ssl_session_cache         shared:SSL:10m;
    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers               ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_stapling              on;
    ssl_stapling_verify       on;
    ssl_trusted_certificate   /etc/pki/tls/certs/example.com.cer;

    # 检查访问文件/目录是否存在，否则重定向 /index.php，若 URL 带参数则附加
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    # 可选：WordPress 程序 Google XML Sitemaps 插件伪静态重写规则
    rewrite ^/sitemap(-+([a-zA-Z0-9_-]+))?\.xml$ "/index.php?xml_sitemap=params=$2" last;
    rewrite ^/sitemap(-+([a-zA-Z0-9_-]+))?\.xml\.gz$ "/index.php?xml_sitemap=params=$2;zip=true" last;
    rewrite ^/sitemap(-+([a-zA-Z0-9_-]+))?\.html$ "/index.php?xml_sitemap=params=$2;html=true" last;
    rewrite ^/sitemap(-+([a-zA-Z0-9_-]+))?\.html.gz$ "/index.php?xml_sitemap=params=$2;html=true;zip=true" last;

    # 将 PHP 请求传递给 FastCGI 处理，这里引用默认配置
    include /etc/nginx/default.d/php.conf;

    # 引用 Let’s Encrypt 配置，允许申请/续期证书时访问特定目录
    include /etc/nginx/snippets/letsencrypt-acme-challenge.conf;

    # 引用 Gzip 配置启用压缩
    include /etc/nginx/snippets/enable-gzip-compression.conf;

    # 设置图片资源缓存时间
    location ~ .*\.(png|jpg|jpeg|gif|bmp)$ {
        expires 30d;
    }

    # 设置其它资源缓存时间
    location ~ .*\.(js|css)?$ {
        expires 12h;
    }

    # 拒绝对隐藏文件访问
    location ~ /\. {
        deny all;
    }

    # 拒绝对指定目录 PHP 文件访问
    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    # 设置日志存儲位置
    access_log /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log warn;
}
```
刷新 Nginx 服务，使配置生效。
```
nginx -s reload
```

###### 创建网站数据库和帐户
使用数据库 root 帐号登录 SQL Shell，过程中输入之前设置的密码。
```
mysql -u root -p
```
创建一个数据库，名称可随意（不区分大小写），譬如 testdb。
```
CREATE DATABASE testdb;
```
创建数据库帐户，例如用户名 testuser，密码 password，并赋予 testdb 数据库管理权限给创建的帐户。
```
GRANT ALL ON testdb.* TO 'testuser'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;
```
刷新数据库权限表，之后用 exit 退出。
```
FLUSH PRIVILEGES;
```

###### 初始化网站程序安装
访问网站域名完成 WordPress 安装。过程中输入之前设置的数据库信息，设置网站标题，管理员帐号等。到此，一个基于 LNMP 环境运行的 WordPress 网站就搭建安装完成了。

如果你在搭建后访问出错，先看下 Nginx 日志文件有没有错误，例如查看最近 30 条错误记录。
```
tail -30 /var/log/nginx/example.com.error.log
```
或者查看 SELinux 有没有拒绝日志。
```
# 查看 SELinux 最近拒绝哪些操作
ausearch -m AVC,USER_AVC,SELINUX_ERR,USER_SELINUX_ERR -ts today

# 查看错误详细信息（如果找不到命令，需安装 setroubleshoot 软件包）
sealert -l "*"
```