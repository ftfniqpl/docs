# firewall使用

## 一、管理端口
1. 列出 dmz 级别的被允许的进入端口
```
# firewall-cmd --zone=dmz --list-ports
```
2. 允许 tcp 端口 8080 至 dmz 级别
```
# firewall-cmd --zone=dmz --add-port=8080/tcp
```
3. 允许某范围的 udp 端口至 public 级别，并永久生效
```
# firewall-cmd --zone=public --add-port=5060-5059/udp --permanent
```
## 二、 网卡接口
1. 列出 public zone 所有网卡
```
# firewall-cmd --zone=public --list-interfaces
```
2. 将 eth0 添加至 public zone，永久
```
# firewall-cmd --zone=public --permanent --add-interface=eth0
```
3. eth0 存在与 public zone，将该网卡添加至 work zone，并将之从 public zone 中删除
```
# firewall-cmd --zone=work --permanent --change-interface=eth0
```
4. 删除 public zone 中的 eth0，永久
```
# firewall-cmd --zone=public --permanent --remove-interface=eth0
```
## 三、 管理服务
1. 添加 smtp 服务至 work zone
```
# firewall-cmd --zone=work --add-service=smtp
```
2. 移除 work zone 中的 smtp 服务
```
# firewall-cmd --zone=work --remove-service=smtp
```
## 四、 配置 external zone 中的 ip 地址伪装
1. 查看
```
# firewall-cmd --zone=external --query-masquerade
```
2. 打开伪装
```
# firewall-cmd --zone=external --add-masquerade
```
3. 关闭伪装
```
# firewall-cmd --zone=external --remove-masquerade
```
## 五、 配置 public zone 的端口转发
1. 要打开端口转发，则需要先
```
# firewall-cmd --zone=public --add-masquerade
```
2. 然后转发 tcp 22 端口至 3753
```
# firewall-cmd --zone=public --add-forward-port=port=22:proto=tcp:toport=3753
```
3. 转发 22 端口数据至另一个 ip 的相同端口上
```
# firewall-cmd --zone=public --add-forward-port=port=22:proto=tcp:toaddr=192.168.1.100
```
4. 转发 22 端口数据至另一 ip 的 2055 端口上
```
# firewall-cmd --zone=public --add-forward-port=port=22:proto=tcp:toport=2055:toaddr=192.168.1.100
```
## 六 、配置 public zone 的 icmp
1. 查看所有支持的 icmp 类型
```
# firewall-cmd --get-icmptypes
  destination-unreachable echo-reply echo-request parameter-problem redirect router-advertisement router-solicitation source-quench time-exceeded
```
2. 列出
```
# firewall-cmd --zone=public --list-icmp-blocks
```
3. 添加 echo-request 屏蔽
```
# firewall-cmd --zone=public --add-icmp-block=echo-request [--timeout=seconds]
```
4. 移除 echo-reply 屏蔽
```
# firewall-cmd --zone=public --remove-icmp-block=echo-reply
```
## 七、 IP 封禁 （这个是我们平时用得最多的）
```
# firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='222.222.222.222' reject"  单个IP
# firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='222.222.222.0/24' reject" IP段
# firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=192.168.1.2 port port=80  protocol=tcp  accept" 单个IP的某个端口
```
这个是我们用得最多的。封一个IP，和一个端口   reject 拒绝   accept 允许

当然，我们仍然可以通过 ipset 来封禁 ip

1. 封禁 ip
```
# firewall-cmd --permanent --zone=public --new-ipset=blacklist --type=hash:ip
# firewall-cmd --permanent --zone=public --ipset=blacklist --add-entry=222.222.222.222
```
2. 封禁网段
```
# firewall-cmd --permanent --zone=public --new-ipset=blacklist --type=hash:net
# firewall-cmd --permanent --zone=public --ipset=blacklist --add-entry=222.222.222.0/24
```
3. 倒入 ipset 规则
```
# firewall-cmd --permanent --zone=public --new-ipset-from-file=/path/blacklist.xml
```
4. 然后封禁 blacklist
```
# firewall-cmd --permanent --zone=public --add-rich-rule='rule source ipset=blacklist drop'
```
## 七、IP封禁和端口
```
# firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=192.168.1.2 port port=80  protocol=tcp  accept"
```
只对192.168.1.2这个IP只能允许80端口访问  （拒绝访问只需把  accept 换成 reject、删除该规则把 –add-rich-rule 改成 –remove-rich-rule即可）
```
# firewall-cmd --permanent --add-rich-rule="rule family=ipv4 source address=192.168.1.2/24 port port=80  protocol=tcp  accept"
```
只对192.168.1.2这个IP段只能允许80端口访问（拒绝访问只需把  accept 换成 reject、删除该规则把 –add-rich-rule 改成 –remove-rich-rule即可）

## 八、双网卡内网网卡不受防火墙限制
```
# firewall-cmd --permanent --zone=public --add-interface=eth1   
```
公网网卡–zone=public默认区域
```
# firewall-cmd --permanent --zone=trusted --add-interface=eth2
```
内网网卡–zone=trusted是受信任区域 可接受所有的网络连接

## 九、重新载入以生效
```
# firewall-cmd --reload
```
