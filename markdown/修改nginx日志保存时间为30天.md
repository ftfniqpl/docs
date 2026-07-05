# 修改nginx日志保存时间为30天

在 /etc/logrotate.d目录下找到nginx的文件并打开

将 rotate的值修改为30，保存并退出

然后执行`logrotate /etc/logrotate.d/nginx` 使其生效