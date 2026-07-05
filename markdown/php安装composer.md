# php安装composer


```
yum -y update

yum install php-cli php-zip wget unzip

php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"

php composer-setup.php --install-dir=/usr/local/bin --filename=composer

```

# php安装composer

```
//下载安装脚本 － composer-setup.php － 到当前目录   
php -r "copy('https://install.phpcomposer.com/installer', 'composer-setup.php');"

//执行安装过程    
php composer-setup.php

//删除安装脚本    
php -r "unlink('composer-setup.php');"

//将命令添加到全局    
sudo mv composer.phar /usr/local/bin/composer

//全局配置使用阿里云源    
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/

//取消全局配置阿里云源    
composer config -g --unset repos.packagist
```