# lvm添加新盘

### 步骤

```
fdisk /dev/vda
n # 新建分区
p # 主分区
回车 # 默认分区号
回车 # 默认开始位置
回车 # 默认结束位置
t # 指定分区格式
L # 列出可选格式
8e # 指定格式LVM
w # 保存操作

reboot

创建物理卷和分区
pvcreate /dev/sda3
vgextend centos /dev/sda3

我们查看物理卷情况
vgdisplay

按照实际大小酌情增加，比如我现在需增加40GB。
lvresize -L +40G /dev/mapper/centos-root

然后动态扩容分区的大小：
resize2fs /dev/mapper/centos-root
报错的话，执行命令
xfs_growfs /dev/mapper/centos-root

.最后，查看最终的结果
df -Th



swap扩容后格式化
swapoff /dev/mapper/centos-swap

lvresize -L + 10G /dev/mapper/centos-swap
mkswap /dev/mapper/centos-swap
swapon /dev/mapper/centos-swap



```

#### 1.查看当前硬盘及分区情况
```
[root@localhost ~]# fdisk -l

Disk /dev/sdb: 107.4 GB, 107374182400 bytes
255 heads, 63 sectors/track, 13054 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
```

其中`/dev/sdb`是我需要增加的硬盘

#### 2.初始化/dev/sdb为PV(physical volume)

命令如下:
```
[root@localhost ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
[root@localhost ~]# pvdisplay
  "/dev/sdb" is a new physical volume of "100.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb
  VG Name               
  PV Size               100.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               5d602Y-xPFg-RWj8-OUcS-H6M4-Rkn4-UXWofX
```

#### 3.PV加入至VG组。
```
[root@localhost ~]# vgcreate VGroup00 /dev/sdb
  Volume group "VGroup00" successfully created
[root@localhost ~]# vgdisplay
  --- Volume group ---
  VG Name               VGroup00
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               100.00 GiB
  PE Size               4.00 MiB
  Total PE              25599
  Alloc PE / Size       0 / 0   
  Free  PE / Size       25599 / 100.00 GiB
  VG UUID               dxt5j1-EM7w-C24F-y0Fm-ZbW4-6LfY-IbIfY2
```

#### 4.创建lv
```
[root@localhost ~]# lvcreate -l +100%free -n LVol00 VGroup00
  Logical volume "LVol00" created.
[root@localhost ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/VGroup00/LVol00
  LV Name                LVol00
  VG Name                VGroup00
  LV UUID                DVpdmE-JOJi-WvtU-gHVy-okP3-cw2s-Vfl1eh
  LV Write Access        read/write
  LV Creation host, time localhost.localdomain.localdomain, 2019-10-28 09:39:34 +0800
  LV Status              available
  # open                 0
  LV Size                100.00 GiB
  Current LE             25599
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
```
以上命令，把所有的空闲空间划至`/dev/VGroup00/LVol00`空间中。

#### 5.格式化逻辑分区
```
[root@localhost ~]# mkfs.ext4 /dev/VGroup00/LVol00
mke2fs 1.41.12 (17-May-2010)
文件系统标签=
操作系统:Linux
块大小=4096 (log=2)
分块大小=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
6553600 inodes, 26213376 blocks
1310668 blocks (5.00%) reserved for the super user
第一个数据块=0
Maximum filesystem blocks=4294967296
800 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000, 7962624, 11239424, 20480000, 23887872

正在写入inode表: 完成                            
Creating journal (32768 blocks): 完成
Writing superblocks and filesystem accounting information: 完成

This filesystem will be automatically checked every 26 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```

#### 6.挂载硬盘/data
```
[root@localhost ~]# mkdir -p /data
[root@localhost ~]# mount /dev/VGroup00/LVol00 /data
[root@localhost ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda3              29G   26G  2.4G  92% /
tmpfs                 3.9G  4.0K  3.9G   1% /dev/shm
/dev/sda1             477M  105M  347M  24% /boot
/dev/mapper/VGroup00-LVol00
                       99G   60M   94G   1% /data
```

#### 7.迁移zabbix的mysql数据库（附加操作）

原来mysql的数据目录在/var/lib/mysql。把它迁移至/data/mysql目录中。

##### 7.1 关闭相关服务
```
service zabbix-server stop
service httpd stop
service mysqld stop
```

##### 7.2 迁移目录
```
[root@localhost ~]# cd /var/lib/
[root@localhost lib]# mv mysql /data/

mkdir -p /var/lib/mysql
chown -R mysql:mysql mysql
```

##### 7.3 修改my.cnf
```
# Remove leading # and set to the amount of RAM for the most important data
# Remove leading # to turn on a very important data integrity option: logging
datadir=/data/mysql
```

##### 7.4 开启服务
```
service mysqld start
service httpd start
service zabbix-server start
```