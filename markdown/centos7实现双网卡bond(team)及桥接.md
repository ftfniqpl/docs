# centos7实现双网卡bond及桥接


1. 介绍   
    将多块网卡绑定同一IP地址对外提供服务，可以实现高可用或者负载均衡。直接给两块网卡设置同一IP地址是不可以的。通过bonding，虚拟一块网卡对外提供连接，物理网卡的被修改为相同的MAC地址    
    
2. Bonding工作模式    
   Linux网卡绑定共七种模式，分别是如下模式：    
- mode=0  round-robin轮询策略(Round-robin policy)    
- mode=1  active-backup主备策略(Active-backup policy)    
- mode=2  load balancing (xor)异或策略(XOR policy)    
- mode=3  fault-tolerance (broadcast)广播策略(Broadcast policy)    
- mode=4  lacp IEEE 802.3ad 动态链路聚合(IEEE 802.3ad Dynamic link aggregation)    
- mode=5  transmit load balancing适配器传输负载均衡(Adaptive transmit load balancing)    
- mode=6  adaptive load balancing适配器负载均衡(Adaptive load balancing)    

1.    round-robin轮询策略
```
cat /proc/net/bonding/bond0    
Bonding Mode: load balancing (round-robin)    
该模式下，链路处于负载均衡状态，数据以轮询方式向每条链路发送报文，基于per packet方式发送。即每条链路各一个数据包。这模式好处在于增加了带宽，同时支持容错能力，当有链路出问题，会把流量切换到正常的链路上。该模式下，交换机端需要配置聚合口，在cisco交换机上叫port channel
```

2.active-backup主备策略
```
该模式拓扑图与上图相同
cat /proc/net/bonding/bond0
Bonding Mode: fault-tolerance (active-backup)
在该模式下，一个端口处于主状态，一个处于备状态，所有流量都在主链路上发出和接收，备链路不会有任何流量。当主端口down掉时，备端口接管主状态。同时可以设置primary网卡，若primary网卡出现故障，切换至备网卡，primary网卡回复后，流量自动回切。这种模式接入不需要交换机端支持。
```

3.load balancing (xor)异或策略
```
该模式拓扑图与上图相同
cat /proc/net/bonding/bond0
Bonding Mode: load balancing (xor)
在该模式下，通过源和目标mac做hash因子来做xor算法来选择链路，这样就使得到达特定对端的流量总是从同一个接口上发出。和balance-rr一样，交换机端口需要能配置为“port channel”。
值得注意的是，若选择这种模式，如果所有流量源和目标mac都固定了，例如使用“网关模式”，即所有对外的数据传输均固定走一个网关，那么根据该模式的描述，分发算法算出的线路就一直是同一条，另外一条链路不会有任何数据流，那么这种模式就没有多少意义了。
```

4.fault-tolerance (broadcast)广播策略
```
cat /proc/net/bonding/bond0
Bonding Mode: fault-tolerance (broadcast)
这种模式的特点是一个报文会复制两份往bond下的两个接口分别发送出去。当有对端交换机失效，我们感觉不到任何丢包。这个模式也需要交换机配置聚合口。

从这个ping信息可以看到，这种模式的特点是，同一个报文服务器会复制两份分别往两条线路发送，导致回复两份重复报文，虽然这种模式不能起到增加网络带宽的效果，反而给网络增加负担，但对于一些需要高可用的环境下，例如RAC的心跳网络，还是有一定价值的。
```

5.lacp IEEE 802.3ad 动态链路聚合
```
该模式拓扑结构与主备模式相同
cat /proc/net/bonding/bond0
Bonding Mode: IEEE 802.3ad Dynamic link aggregation
该模式是基于IEEE 802.3ad Dynamic link aggregation（动态链接聚合）协议，针对该协议的介绍，在公众号之前的文章中有所涉及。
在该模式下，操作系统和交换机都会创建一个聚合组，在同一聚合组下的网口共享同样的速率和双工设定。操作系统根据802.3ad 协议将多个slave 网卡绑定在一个聚合组下。聚合组向外发送数据选择哪一块儿网卡是基于传输hash 策略，该策略可以通过xmit_hash_policy 选项从缺省的XOR 策略改变到其他策略。
该模式的必要条件：
- ethtool 支持获取每个slave 的速率和双工设定；
- 交换机支持IEEE 802.3ad Dynamic link aggregation。
大多数交换机需要经过特定配置才能支持802.3ad 模式。
```

6.transmit load balancing适配器传输负载均衡
```
cat /proc/net/bonding/bond0
Bonding Mode: transmit load balancing
这种模式相较load balancing (xor)异或策略及LACP模式的hash策略相对智能，会主动根据对端的MAC地址上的流量，智能的分配流量从哪个网卡发出。但不足之处在于，仍使用一块网卡接收数据。存在的问题与load balancing (xor)也是一样的一样，如果对端MAC地址是唯一的，那么策略就会失效。这个模式下bond成员使用各自的mac，而不是上面几种模式是使用bond0接口的mac。无需交换机支持
```

7.adaptive load balancing适配器负载均衡
```
该模式拓扑结构与上图一致。
cat /proc/net/bonding/bond0
Bonding Mode: adaptive load balancing
该模式除了balance-tlb适配器传输负载均衡模式的功能外，同时加上针对IPV4流量接收的负载均衡。接收负载均衡是通过ARP协商实现的。在进行ARP协商的过程中，bond模块将对端和本地的mac地址进行绑定，这样从同一端发出的数据，在本地也会一直使用同一块网卡来接收。若是网络上发出的广播包，则由不同网卡轮询的方式来进行接收。通过这种方式实现了接收的负载均衡。该模式同样无需交换机支持。
```


3.配置bonding    
    3.1 默认情况下bonding模块没有被加载，需先加载bonding模块   

```
[root@c2 ~]# lsmod |grep bonding
[root@c2 ~]# modprobe bonding
[root@c2 ~]# lsmod |grep bonding
bonding               152656  0 
```

    3.2 新建bond网口的配置文件

```
[root@c2 ~]# cat /etc/sysconfig/network-scripts/ifcfg-bond0 
DEVICE=bond0
NAME=bond0
TYPE=Bond
BONDING_MASTER=yes
IPADDR=192.168.0.100
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DNS1=8.8.8.8
ONBOOT=yes
BOOTPROTO=none
BONDING_OPTS="mode=0 miimon=100"
```

    3.3 网口eth0、eth1的配置文件
    
```
[root@c2 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
NAME="eth0"
DEVICE="eth0"
ONBOOT=yes
BOOTPROTO=none
TYPE=Ethernet
MASTER=bond0
SLAVE=yes
```

```
[root@c2 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth1
NAME="eth1"
DEVICE="eth1"
ONBOOT=yes
BOOTPROTO=none
TYPE=Ethernet
MASTER=bond0
SLAVE=yes
```
        
    3.4 br0配置
    
```
[root@c2 network-scripts]# cat ifcfg-br0
TYPE=Bridge
BOOTPROTO=static
PEEDNS=yes
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=192.168.0.100
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DNS1=8.8.8.8
USERCTL=no
```

    3.4 重新加载网络配置

```
[root@c2 ~]# nmcli connection reload        
[root@c2 ~]# systemctl restart network.service
```
    
    3.5 查看bonding状态
    
```
[root@c2 ~]# cat /proc/net/bonding/bond0 
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: adaptive load balancing
Primary Slave: None
Currently Active Slave: eth1
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eth1
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:0c:29:ba:03:9e
Slave queue ID: 0
```

3.6 删除网卡

```
nmcli con show
nmcli con delete team0
systemctl restart network
```



# 双网卡配置team绑定

需要在生产环境下要部署kvm虚拟化生产环境，因为网络采用的是叶脊拓扑，需要对多个网卡进行绑定，常用绑定技术为bond和team，team是centos7新支持的，性能和稳定性都高于bond,因此决定采用team对多个网卡进行绑定。需要绑定的两块万兆卡名称为ens7f0和ens7f1配置步骤如下：

1、 配置ens7f0
```
[root@localhost network-scripts]# cat ifcfg-ens7f0
NAME=ens7f0
DEVICE=ens7f0
ONBOOT=yes
DEVICETYPE=TeamPort
TEAM_MASTER=team0
TEAM_PORT_CONFIG='{"prio":9}'
```

2、 配置ens7f1
```
[root@localhost network-scripts]# cat ifcfg-ens7f1
NAME=ens7f1
DEVICE=ens7f1
DEVICETYPE=TeamPort
ONBOOT=yes
TEAM_MASTER=team0
TEAM_PORT_CONFIG={"prio": 10,"sticky": true}
```

3、 配置team0
```
[root@localhost network-scripts]# cat ifcfg-team0
DEVICE=team0
NAME=team0
DEVICETYPE=Team
TEAM_CONFIG='{"runner": {"name": "activebackup"},"link_watch": {"name": "ethtool","delay_up": 2500,"delay_down": 1000}}'
ONBOOT=yes
BRIDGE=br0
```

4、 配置br0
```
[root@localhost network-scripts]# cat ifcfg-br0
TYPE=Bridge
BOOTPROTO=static
NAME=br0
DEVICE=br0
ONBOOT=yes
IPADDR=192.168.217.12
NETMASK=255.255.255.0
GATEWAY=192.168.217.254
DNS1=119.29.29.29
DNS2=8.8.8.8
```

5、 查看网卡状态
```
[root@localhost network-scripts]# teamdctl team0 state view
setup:
  runner: activebackup
ports:
  ens7f0
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
  ens7f1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
        down count: 0
runner:
  active port: ens7f0
```
  
至此，整个team端口组配置完毕，实测无任何问题