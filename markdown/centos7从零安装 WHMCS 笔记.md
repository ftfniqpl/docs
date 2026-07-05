# CentOS 7 从零安装 WHMCS 笔记

记录下 WHMCS 安装步骤，服务器环境为 CentOS 7，从零部署 LNMP 网站环境，然后上传 WHMCS 安装，以及完成初始化配置和安全增强。

## 系统基本配置
```
# 设置东八区时间
timedatectl set-timezone 'Asia/Shanghai'

# 更改主机名
hostnamectl set-hostname webserver.localdomain

# 绑定主机名
cat > /etc/hosts << "EOF"
127.0.0.1   webserver.localdomain webserver localhost.localdomain localhost
::1         webserver.localdomain webserver localhost.localdomain localhost
EOF

# 更新软件包
yum -y update && reboot

# 安装常用软件
yum install -y wget unzip screen lrzsz
```

## 配置 LNMP 网站环境
LNMP 环境配置参考这篇文章 https://www.hostarr.com/install-lnmp-via-ius/

## 安装 Ioncube 插件
```
# 下载 Ioncube 并解压到当前目录
wget https://downloads.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz
tar -zxvf ioncube_loaders_lin_x86-64.tar.gz

# 复制插件文件到 PHP 扩展目录（选择与 PHP 版本相符或接近的版本）
# 可通过 php -i | grep extension_dir 查询当前 PHP 扩展目录路径
cp -Z /root/ioncube/ioncube_loader_lin_7.4.so /usr/lib64/php/modules

# 赋予插件文件可执行权限
chmod +x /usr/lib64/php/modules/ioncube_loader_lin_7.4.so

# 创建配置文件让 PHP 加载插件（可通过 php -i | grep 'additional .ini files' 查询当前 PHP 配置文件目录）
echo "zend_extension=/usr/lib64/php/modules/ioncube_loader_lin_7.4.so" > /etc/php.d/00-ioncube.ini

# 删除之前操作留下的多余文件
rm -rf /root/ioncube /root/ioncube_loaders_lin_x86-64.tar.gz

# 重启服务生效
nginx -s reload
systemctl restart php-fpm

# 验证 Ioncube 有无安装成功
php -v
```

## 下载 WHMCS 和中文语言包
```
WHMCS 安装包 https://s3.amazonaws.com/releases.whmcs.com/v2/pkgs/whmcs-8.1.3-release.1.zip

WHMCS 语言包 https://github.com/kaneawk/WHMCS-CN-translations
```

## 创建数据库、上传 WHMCS 程序及完成安装

数据库创建在之前 LNMP 教程里有介绍。下面是对网站目录的一些操作。
```
# 设置网站目录 SELinux 安全上下文为只读
semanage fcontext -a -t httpd_sys_content_t "/var/www/html(/.*)?"
restorecon -R -v /var/www/html

# 上传 WHMCS 文件后重命名配置文件名
mv /var/www/html/configuration.php.new /var/www/html/configuration.php

# 将网站目录用户/组改为 nginx
chown -R nginx:nginx /var/www/html

# 为必要文件/目录设置 SELinux 读写安全上下文
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/configuration.php"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/attachments(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/downloads(/.*)?"
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/templates_c(/.*)?"
restorecon -R -v /var/www/html
```
之后访问 https://example.com/install/install.php 开始安装。

安装完成后删除根目录下的 install 目录，并将 configuration.php 安全上下文改回只读，之后便可以登录管理员后台。
```
# 删除安装目录
rm -rf /var/www/html/install

# 将配置文件改回只读权限
semanage fcontext -a -t httpd_sys_content_t "/var/www/html/configuration.php"
restorecon -R -v /var/www/html
```

## WHMCS 安全和初始化配置

#### 保护可写目录

将所有可写目录移动到网站根目录上级，以避免 Web 访问。先创建要使用的新目录（如下）。
```
# 创建新的可写存储目录
mkdir -p /var/www/whmcs_writeable/attachments /var/www/whmcs_writeable/attachments/projects /var/www/whmcs_writeable/downloads /var/www/whmcs_writeable/templates_c

# 设置 SELinux 读写安全上下文
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/whmcs_writeable(/.*)?"
restorecon -R -v /var/www/whmcs_writeable

# 将目录所有者改为 nginx 用户/组
chown -R nginx:nginx /var/www/whmcs_writeable
```

然后转到 WHMCS 后台 -> 配置 -> 系统配置 -> 存储设置，先点“配置”选项卡添加新的本地存储目录（不用添加 templates_c 目录），再切回“设置”选项卡选择使用新的目录。

接着修改 configuration.php 配置文件，设置使用新的 templates_c 目录。
```
# 设置新的 templates_c 目录
sed -i "s|'templates_c'|'/var/www/whmcs_writeable/templates_c'|" /var/www/html/configuration.php

# 删除之前目录的 SELinux 读写安全上下文
semanage fcontext -d "/var/www/html/attachments(/.*)?"
semanage fcontext -d "/var/www/html/downloads(/.*)?"
semanage fcontext -d "/var/www/html/templates_c(/.*)?"
restorecon -R -v /var/www/html

# 再删除旧有目录
rm -rf /var/www/html/attachments /var/www/html/downloads /var/www/html/templates_c
```

#### 保护 WHMCS 配置文件

将 configuration.php 文件权限设为 400（仅允许文件所有者只读）。
```
chmod 400 /var/www/html/configuration.php
```
注：如果设置后遇到问题，可尝试设置 440，其次是 444。如果之后要更新 WHMCS 密钥，则先临时将文件权限设为 755，更新后再改回去。

#### 移动 Crons 目录和设置 Cron 作业

为防止 Web 访问 Crons 目录，需将其移动到其它目录。先创建目录（如下）。
```
# 创建新的只读存储目录
mkdir -p /var/www/whmcs_readable

# 设置 SELinux 只读安全上下文
semanage fcontext -a -t httpd_sys_content_t "/var/www/whmcs_readable(/.*)?"
restorecon -R -v /var/www/whmcs_readable

# 将目录所有者改为 nginx 用户/组
chown -R nginx:nginx /var/www/whmcs_readable
```
之后移动 Crons 目录，并在 Crons 配置文件指定新的存放目录。
```
# 移动 Crons 目录
mv -Z /var/www/html/crons /var/www/whmcs_readable

# 指定 WHMCS 安装目录
cat > /var/www/whmcs_readable/crons/config.php << "EOF"
<?php

$whmcspath = '/var/www/html/';

EOF

# 为新文件设置 nginx 用户/组
chown nginx:nginx /var/www/whmcs_readable/crons/config.php

# 指定 Crons 存放目录
echo -e "\n\$crons_dir = '/var/www/whmcs_readable/crons/';" >> /var/www/html/configuration.php
```
注：在移动 Crons 目录后，后续如果手动升级 WHMCS，需要将新的 Crons 目录文件手动覆盖旧文件。

最后，为使 WHMCS 可以执行自动化任务（例如生成续订发票等），我们需要设置 Cron 作业。
```
# 添加 Cron 计划任务
crontab -e

# 设置每 5 分钟运行 Cron 作业
*/5 * * * * /usr/bin/php -q /var/www/whmcs_readable/crons/cron.php
```
#### 更改 admin 目录名称
为避免被猜到后台登录地址，建议更改 admin 目录名称。
```
# 在配置文件指定新目录名称（名称仅限小写字母，数字，下划线、连字符）
echo -e "\n\$customadminpath = 'my_custom_folder_name';" >> /var/www/html/configuration.php

# 重命名目录名称
mv /var/www/html/admin /var/www/html/my_custom_folder_name
```
注：在更改 admin 目录名称后，如果更新了 WHMCS，需将新的 admin目录文件覆盖到新目录。

#### 限制 vendor 目录访问
该目录不应提供 Web 访问，需要添加拒绝访问指令。
```
# 编辑 Nginx 站点配置文件
vi /etc/nginx/conf.d/example.conf

# 在 server 位置内添加拒绝访问指令
server {
    ...
    location ^~ /vendor/ {
        deny all;
        return 403;
    }
    ...
}

# 重新加载 Nginx 服务生效
nginx -s reload
```
#### WHMCS Nginx 配置模板
最后附上适用 WHMCS8 的 Nginx 站点配置模板，以供参考（使用时建议删除中文注释）。
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

    # 将 PHP 请求传递给 FastCGI 处理，这里引用默认配置
    include /etc/nginx/default.d/php.conf;

    # 引用 Let’s Encrypt 配置，允许申请/续期证书时访问特定目录
    include /etc/nginx/snippets/letsencrypt-acme-challenge.conf;

    location ~ /announcements/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/announcements/$1;
    }

    location ~ /download/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/download$1;
    }

    location ~ /knowledgebase/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/knowledgebase/$1;
    }

    location ~ /store/ssl-certificates/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/store/ssl-certificates/$1;
    }

    location ~ /store/sitelock/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/store/sitelock/$1;
    }

    location ~ /store/website-builder/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/store/website-builder/$1;
    }

    location ~ /store/order/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/store/order/$1;
    }

    location ~ /cart/domain/renew/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/cart/domain/renew$1;
    }

    location ~ /account/paymentmethods/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/account/paymentmethods$1;
    }

    location ~ /password/reset/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/password/reset/$1;
    }

    location ~ /account/security/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/account/security$1;
    }

    location ~ /subscription?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=/subscription$1;
    }

    location ~ /auth/provider/google_signin/finalize/?(.*)$ {
        rewrite ^/(.*)$ /index.php?rp=auth/provider/google_signin/finalize$1;
    }

    # 设置脚本资源缓存时间
    location ~ .*\.(js|css)?$ {
        expires 12h;
    }

    # 设置图片资源缓存时间
    location ~ .*\.(png|jpg|jpeg|gif|bmp)$ {
        expires 30d;
    }

    # 拒绝访问模板文件
    location ~ .*\.(tpl|inc|cfg)?$ {
        deny all;
    }

    # 拒绝对隐藏文件访问
    location ~ /\. {
        deny all;
    }

    # 拒绝访问 vendor 目录
    location ^~ /vendor/ {
        deny all;
        return 403;
    }

    # 设置日志存儲位置
    access_log /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log warn;
}
```
注：里面的 Nginx 重写规则，设置后需在 WHMCS 其它选项 -> 系统相关 -> 系统清理 -> 清空模板缓存后方能生效。

#### 限制搜索引擎爬取
如需控制搜索引擎爬取内容，需在网站根目录创建 robots.txt 文件设置。譬如拒绝爬取全站内容。
```
cat > /var/www/html/robots.txt << "EOF"
User-agent: *
Disallow: /
EOF
```
#### 结束语
完成上述服务器配置后就可以开始使用了，接下来请配置 WHMCS 系统选项，支付网关，添加产品服务，工单支持设置，邮件发送服务，自动化任务等。