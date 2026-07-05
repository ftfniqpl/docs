# openwrt配置nginx禁用uci管理


```
uci set nginx.global.uci_enable=false
uci commit nginx
/etc/init.d/nginx restart
cp /etc/nginx/uci.conf.template nginx.conf
```