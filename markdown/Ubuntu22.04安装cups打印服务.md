# Ubuntu 20.04 安装CUPS 共享打印服务

1. systemctl disable --now unattended-upgrades

2. sudo apt-get update -y

3. sudo apt remove cloud-init -y

4. sudo apt install vim net-tools -y

5. 修改网卡等待时间:

   1. vim /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service 
   2. 在Service段中添加 TimeoutStartSec=2sec
   3. systemctl daemon-reload

6. sudo apt install cups

7. vim /etc/cups/cupsd.conf

   1. 修改18行 Listen 为0.0.0.0:631
   2. 修改22行 Browsing Off 为Browsing On
   3. 在29行下面增加 ServerAlias *
   4. 在所有的在`Order Allow,Deny`下面增加 `Allow all`

8. 修改pip为国内源

   1. 在/home/linux下创建.pip文件夹与pip.conf文件

   2. mkdir -p /home/linux/.pip

   3. vim /home/linux/.pip/pip.conf

      [global]

      index-url = https://mirrors.aliyun.com/pypi/simple/

      [install]

      trusted-host=mirrors.aliyun.com

   4.  mkdir -p /root/.pip

   5. cp /home/linux/.pip/pip.conf /root/.pip/pip.conf

9. 编译hplip准备

   1. 添加 sudo apt install python-is-python3 libavahi-core-dev
   2. sudo apt install hplip hplip-data printer-driver-hpcups  -y
   3. sudo apt-get install --assume-yes python3-pip
   4. sudo pip3 install --upgrade pip
   5. sudo apt-get install --assume-yes libleptonica-dev
   6. sudo apt-get install --assume-yes tesseract-ocr tesseract-ocr-all
   7. sudo apt-get install --assume-yes libtesseract-dev
   8. sudo -H pip3 install tesserocr
   9. sudo -H pip3 install opencv-python  
   10. sudo -H pip3 install PyPDF2
   11. sudo -H pip3 install imutil
   12. sudo -H pip3 install scikit-image
   13. sudo -H pip3 install scipy
   14. sudo apt install ocrmypdf
   15. 在非root用户下执行 ./hplip-3.22.04.run

   ![一些安装选项](./assets/Ubuntu22.04%E5%AE%89%E8%A3%85cups%E6%89%93%E5%8D%B0%E6%9C%8D%E5%8A%A1/6673adda2c3e1.png) 

   ![继续一些配置](./assets/Ubuntu22.04%E5%AE%89%E8%A3%85cups%E6%89%93%E5%8D%B0%E6%9C%8D%E5%8A%A1/6673addb57a99.png) 

10. 然后安装 hp-plugin:  

​	hp-plugin -i

11. 运行lsusb查看 系统检测到的usb设备

12. 最后 将linux用户添加到lpadmin用户组中:

    sudo usermod -aG lpadmin $USER

13. 设置开机自启cups

    sudo systemctl enable cups



# Zerotier安装

```bash
curl -s https://install.zerotier.com | sudo bash
```

```
systemctl enable zerotier-one
zerotier-cli join 93afae5963a766fc
mkdir /var/lib/zerotier-one/moons.d
wget https://rowsea.com/000000b5b91d9c08.moon
mv 000000b5b91d9c08.moon /var/lib/zerotier-one/moons.d/
zerotier-cli orbit b5b91d9c08 b5b91d9c08
systemctl restart zerotier-one
```



# 安装软raid

apt install mdadm

检查磁盘状态

lsblk

fdisk -l

确认哪些磁盘未被使用，例如 `/dev/sdb /dev/sdc /dev/sdd`

清除磁盘旧配置

sudo mdadm --zero-superblock /dev/sd[b-d]

创建raid阵列

RAID5实例

```none
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sd[b-d]
```

RAID 10 示例（4块盘）：

```none
sudo mdadm --create --verbose /dev/md0 --level=10 --raid-devices=4 /dev/sd[b-e]
```

监控阵列同步状态

```none
watch cat /proc/mdstat
```

创建文件系统并挂载

```none
sudo mkfs.ext4 /dev/md0   // 使用磁盘配额 必须使用xfs文件系统 mkfs.xfs -f /dev/md0
sudo mkdir -p /mnt/raid
sudo mount /dev/md0 /mnt/raid
```

持久化配置

```none
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

添加到 `/etc/fstab`：

```none
echo '/dev/md0 /mnt/raid ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```

**性能测试与验证**

```none
sudo apt install fio
fio --name=randwrite --ioengine=libaio --rw=randwrite --bs=4k --size=1G --numjobs=4 --runtime=60 --group_reporting
```

**五、常见问题与排查方法**

问题一：阵列创建失败或卡住

- 原因：某些磁盘可能已有旧 RAID 签名
- 解决：使用 `–zero-superblock` 清除所有参与盘的 RAID 信息

问题二：重启后阵列未挂载

- 检查 `/etc/mdadm/mdadm.conf` 和 `/etc/fstab` 是否正确写入
- 检查 RAID 设备名是否变动（可使用 UUID 挂载）

问题三：磁盘掉线或降级模式

- 使用 `mdadm –detail /dev/md0` 查看状态

替换损坏盘后重新添加：

```none
sudo mdadm --add /dev/md0 /dev/sdX
```

问题四：性能不达预期

- 检查磁盘是否为 SMR 类型（不推荐用于 RAID）
- 检查是否开启写缓存（使用 `hdparm -W`）
- 优化文件系统挂载参数，如 `noatime,nobarrier`



# 安装SMARTCTL

```
apt install g++ gcc make -y
tar -zxvf smartmontools-7.5.tar.gz
cd smartmontools-7.5
./configure --disable-dependency-tracking
make && make install
/usr/local/sbin/smartctl -a /dev/sda
```

# 重装系统后恢复RAID1硬盘阵列

1. 查看是否有阵列存在

   cat /proc/mdstat

   显示我们发现有一个md0,由sdb和sdc组成的

2. 安装mdadm

   apt-get install -y mdadm

3. 导入

   mdadm --assemble --scan

4. 恢复配置并保存

   mdadm --detail --scan >> /etc/mdadm/mdadm.conf

   update-initramfs -u

   显示如下:

   root@ubuntu: ~# update-initramfs -u

   update-initramfs: Generating /boot/initrd.img-4.19.0-10-amd64

5. 删除/etc/mdadm/mdadm.conf类似下面这个

   ARRAY /dev/md/0 

   这一行都要删除，然后重启一下

6. 然后就可以挂载，并设置开机自动挂载了



# 安装Samba

apt install samba

mkdir /mnt/raid/share && chmod 777 /mnt/raid/share

vim /etc/samba/smb.conf

```
[share]
comment = 共享目录
path = /mnt/raid/share
browseable = yes
writable = yes
valid users = linux
create mode = 0644
directory mode = 0775
```

#添加用户

smbpasswd -a linux



systemctl enable smbd

systemctl restart smbd



Centos下开启firewall

```
firewall-cmd --zone=public --add-port=139/tcp --permanent
firewall-cmd --zone=public --add-port=445/tcp --permanent
firewall-cmd --zone=public --add-port=137/udp --permanent
firewall-cmd --zone=public --add-port=138/udp --permanent
```





```
curl -sSL https://raw.githubusercontent.com/lyarinet/Samba-Manager/main/auto_install.sh | sudo bash
```



# Samba服务器（多用户组、多用户有不同的访问权限）

1. 首先服务器采用用户验证的方式，每个用户可以访问自己的宿主目录，并且只有该用户能访问宿主目录，并具有完全的权限，而其他人不能看到你的宿主目录。
2. 建立一个caiwu的文件夹，希望caiwu组和lingdao组的人能看到，network02也可以访问，但只有caiwu01有写的权限。
3. 建立一个lindao的目录，只有领导组的人可以访问并读写，还有network02也可以访问，但外人看不到那个目录
4. 建立一个文件交换目录exchange，所有人都能读写，包括guest用户，但每个人不能删除别人的文件。
5. 建立一个公共的只读文件夹public，所有人只读这个文件夹的内容。

​    前期的工作

​    建立3个组：

```
groupadd caiwu
groupadd network
groupadd lingdao
```

​    添加用户并加入相关的组当中：

```
useradd caiwu01 -g caiwu
useradd caiwu02 -g caiwu
useradd network01 -g network
useradd network02 -g network
useradd lingdao01 -g lingdao
useradd lingdao02 -g lingdao
```

​    然后我们使用smbpasswd -a caiwu01的命令为6个帐户分别添加到samba用户中

```
mkdir /home/samba
mkdir /home/samba/caiwu
mkdir /home/samba/lingdao
mkdir /home/samba/exchange
mkdir /home/samba/public
```

​    为了避免麻烦可以在这里把上面所有的文件夹的权限都设置成777，通过samba灵活的权限管理来设置上面的5点要求。

​    以下是smb.conf的配置文件

```
[global]
workgroup = bmit #我的网络工作组
server string = Frank's Samba File Server #我的服务器名描述
security = user #使用用户验证机制

encrypt passwords = yes
smb passwd file = /etc/samba/smbpasswd #使用加密密码机制，在win95和winnt使用的是明文

其他的基本上可以按照默认的来。

[homes] #homes段满足第1条件
comment = Home Directories
browseable = no
writable = yes
valid users = %S
create mode = 0664
directory mode = 0775

[caiwu] #caiwu段满足我们的第2要求
comment = caiwu
path = /home/samba/caiwu
public = no
valid users = @caiwu,@lingdao,network02
write list = caiwu01
printable = no

[lingdao] #lingdao段能满足我们的第3要求
comment = lingdao
path = /home/samba/lingdao
public = no
browseable = no
valid users = @lingdao,network02
printable = no

[exchage]
comment = Exchange File Directory
path = /home/samba/exchange
public = yes
writable = yes

#exchange段基本能满足我们的第4要求，但不能满足每个人不能删除别人的文件这个条件，即使里设置了mask也是没用，其实这个条件只要unix设置一个粘着位就行
chmod -R 1777 /home/samba/exchange
```

​    注意这里权限是1777，类似的系统目录/tmp也具有相同的权限，这个权限能实现每个人能自由写文件，但不能删除别人的文件这个要求

```
[public]
comment = Read Only Public
path = /home/samba/public
public = yes
read only = yes

#这个public段能满足第5要求。
```

​    到此为止设置已经能实现共享文件要求，记得重启服务

```
#/etc/rc.d/init.d/smb restart
```

​    如果大家没有winodws，不妨先用samba的cilent端命令来测试一下

​    命令的用法举几个例子

```
smbclient -L 服务器ip -N
```

​    guest帐户查询服务器的samba共享情况，可以检验一下是否lingdao目录时候能被guest帐户看到，应该是看不到的，当然也可以以某个用户的名义查看

```
smbclient -L 服务器ip -U caiwu01
```

​    系统会提示密码，只要输入smb密码就行。

```
smbclient //服务器ip/caiwu -U caiwu01

#以caiwu01用户的名义登录caiwu目录

smbmount //服务器ip/caiwu /mnt/caiwu -o username=caiwu01

#把服务器的财务目录映射到本地的/mnt/caiwu目录。
```

​    测试

```
smbclient -L //localhost/share
```

​    或者　

```
smbclient-L \\127.0.0.1 -Umyname
```

​    这时输入的密码就是你刚才设置的samba密码使用

1. windows用户

​    在我的电脑地址栏里输入\192.168.1.1访问；也可windows+R输入\192.168.1.1；
登录后可以右击映射到本地驱动器。

```
net use * /delete
```

1. Linux

- 使用smbclient

```
#smbclient//192.168.1.1/Normal -U user%passwd
```

- 挂载到某个目录使用

```
#mkdir/mnt/share
#mount -o username=youruser,password=passwd //192.168.1.1/Normal  /mnt/share
```

​    设置开机挂载将如下命令写入/etc/fstab

```
//192.168.1.1/share  /mnt/ml45  cifs  defaults,auto,username=youruser,password=passwd 0 0
```

​    然后#mount -a





# 安装NTP Server



修改时区： timedatectl set-timezone Asia/Shanghai

ubuntu设置24h显示: localectl set-locale LC_TIME="en_GB.UTF-8"

关闭systemd-timesyncd服务，防止与ntpd冲突

```
sudo systemctl stop systemd-timesyncd 
sudo systemctl disable systemd-timesyncd 
sudo apt install -y ntp 
sudo systemctl enable ntp --now
```

修改/etc/ntp.conf

```
# 1) 记录频偏
driftfile /var/lib/ntp/ntp.drift
leapfile  /usr/share/zoneinfo/leap-seconds.list

# 2) 上游时间源：至少 3~4 个，分布不同机构/地域
pool 0.cn.pool.ntp.org iburst
pool 1.cn.pool.ntp.org iburst
pool 2.cn.pool.ntp.org iburst
pool 3.cn.pool.ntp.org iburst
# 如需更稳，可替换为企业可信源或本地区权威源

# 3) 访问控制（默认收紧）
restrict -4 default kod notrap nomodify nopeer noquery
restrict -6 default kod notrap nomodify nopeer noquery

# 允许本机与本地网段查询/同步（示例：10.0.0.0/24）
restrict 127.0.0.1
restrict ::1
restrict 10.0.0.0 mask 255.0.0.0 nomodify notrap

# 4) 接口控制（可选：仅监听内网口）
# interface ignore wildcard
# interface listen eth0

# 5) 异常容错（可选）：上游全不可达时有序参考
# tos orphan 10
```



```
sudo systemctl restart ntp
sudo ufw allow 123/udp
sudo ufw reload
```

运行状态检查

```
# 进程/端口
sudo systemctl status ntp
sudo ss -lunp | grep :123

# 同步状态与上游列表
ntpq -p -n

# 粗粒度健康度
ntpstat
```



# RAID完整操作

1. mdadm操作 使用本地回环创建3个块设备

   ```
   $ truncate -s 1G md0.img md1.img md2.img
   $ sudo losetup -f --show md0.img
   /dev/loop0
   $ sudo losetup -f --show md1.img
   /dev/loop1
   $ sudo losetup -f --show md2.img
   /dev/loop2
   ```

2.  创建RAID,先以RAID 0为例，创建一个跨3块盘的RAID 0

   ```
   $ sudo mdadm --create /dev/md0 --level=0 --raid-devices=3 /dev/loop0 /dev/loop1 /dev/loop2
   mdadm: Defaulting to version 1.2 metadata
   mdadm: array /dev/md0 started.
   ```

   得到的 `/dev/md0` 就是创建完成的 RAID 块设备，它的大小是约 3G，可以使用 `mdadm --detail` 查看详细信息。

   ```
   $ sudo mdadm --detail /dev/md0
   /dev/md0:
              Version : 1.2
        Creation Time : Wed Feb 21 00:54:02 2024
           Raid Level : raid0
           Array Size : 3139584 (2.99 GiB 3.21 GB)
         Raid Devices : 3
        Total Devices : 3
          Persistence : Superblock is persistent
   
          Update Time : Wed Feb 21 00:54:02 2024
                State : clean 
       Active Devices : 3
      Working Devices : 3
       Failed Devices : 0
        Spare Devices : 0
   
               Layout : -unknown-
           Chunk Size : 512K
   
   Consistency Policy : none
   
                 Name : hostname:0  (local to host hostname)
                 UUID : 119661ca:ff0a76b7:28909e39:18af76dc
               Events : 0
   
       Number   Major   Minor   RaidDevice State
          0       7        0        0      active sync   /dev/loop0
          1       7        1        1      active sync   /dev/loop1
          2       7        2        2      active sync   /dev/loop2
   ```

   对应的状态也可以通过 `/proc/mdstat` 查看：

   ```
   $ cat /proc/mdstat
   Personalities : [raid6] [raid5] [raid4] [raid1] [raid0] 
   md0 : active raid0 loop2[2] loop1[1] loop0[0]
         3139584 blocks super 1.2 512k chunks
   
   unused devices: <none>
   ```

   使用 `--stop` 选项停止 RAID：

   ```
   $ sudo mdadm --stop /dev/md0
   mdadm: stopped /dev/md0
   $ ls -lha /dev/md0
   ls: cannot access '/dev/md0': No such file or directory
   ```

   在 stop 之后，如果需要再恢复，可以使用 `--assemble` 选项：

   ```
   $ sudo mdadm --assemble /dev/md0 /dev/loop0 /dev/loop1 /dev/loop2
   mdadm: /dev/md0 has been started with 3 drives.
   $ # 或者使用 --scan 参数：
   $ sudo mdadm --assemble --scan
   mdadm: /dev/md/0 has been started with 3 drives.
   ```

   将这个 RAID 0 拆掉，然后试试组建 RAID 1、RAID 5：

   ```
   $ # RAID 1
   $ sudo mdadm --stop /dev/md0
   mdadm: stopped /dev/md0
   $ sudo mdadm --misc --zero-superblock /dev/loop0 /dev/loop1 /dev/loop2
   $ sudo mdadm --create /dev/md0 --level=1 --raid-devices=3 /dev/loop0 /dev/loop1 /dev/loop2
   mdadm: Note: this array has metadata at the start and
       may not be suitable as a boot device.  If you plan to
       store '/boot' on this device please ensure that
       your boot-loader understands md/v1.x metadata, or use
       --metadata=0.90
   Continue creating array? y
   mdadm: Defaulting to version 1.2 metadata
   mdadm: array /dev/md0 started.
   $ # RAID 5
   $ sudo mdadm --stop /dev/md0
   mdadm: stopped /dev/md0
   $ sudo mdadm --misc --zero-superblock /dev/loop0 /dev/loop1 /dev/loop2
   $ sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/loop0 /dev/loop1 /dev/loop2
   mdadm: Defaulting to version 1.2 metadata
   mdadm: array /dev/md0 started.
   ```

   如果在重新创建 mdadm 阵列之前不清空 superblock，会输出类似以下的警告信息：

   ```
   mdadm: /dev/loop0 appears to be part of a raid array:
          level=raid0 devices=3 ctime=Wed Feb 21 00:54:02 2024
   mdadm: /dev/loop1 appears to be part of a raid array:
          level=raid0 devices=3 ctime=Wed Feb 21 00:54:02 2024
   mdadm: /dev/loop2 appears to be part of a raid array:
          level=raid0 devices=3 ctime=Wed Feb 21 00:54:02 2024
   ```

3. 重建操作

   与 LVM 一章类似，这里展示在一块盘丢失（损坏）情况下的操作：

   ```
   $ sudo mdadm --stop /dev/md0
   mdadm: stopped /dev/md0
   $ sudo losetup -D
   $ sudo losetup -f --show md0.img
   /dev/loop0
   $ sudo losetup -f --show md1.img
   /dev/loop1
   $ # 此时 /dev/md0 已经自动构建，并且处于丢失一块盘的状态
   $ sudo mdadm --detail /dev/md0
   /dev/md0:
              Version : 1.2
        Creation Time : Thu Feb 22 13:05:14 2024
           Raid Level : raid5
           Array Size : 2093056 (2044.00 MiB 2143.29 MB)
        Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
         Raid Devices : 3
        Total Devices : 2
          Persistence : Superblock is persistent
   
          Update Time : Thu Feb 22 13:05:19 2024
                State : clean, degraded 
       Active Devices : 2
      Working Devices : 2
       Failed Devices : 0
        Spare Devices : 0
   
               Layout : left-symmetric
           Chunk Size : 512K
   
   Consistency Policy : resync
   
                 Name : hostname:0  (local to host hostname)
                 UUID : 3e4455f5:65e4251c:b21c1c81:13b44a8b
               Events : 18
   
       Number   Major   Minor   RaidDevice State
          0       7        0        0      active sync   /dev/loop0
          1       7        1        1      active sync   /dev/loop1
          -       0        0        2      removed
   $ # 添加新的盘
   $ truncate -s 1G md3.img
   $ sudo losetup /dev/loop3 md3.img
   $ sudo mdadm --add /dev/md0 /dev/loop3
   $ sudo mdadm --detail /dev/md0
   /dev/md0:
              Version : 1.2
        Creation Time : Thu Feb 22 13:05:14 2024
           Raid Level : raid5
           Array Size : 2093056 (2044.00 MiB 2143.29 MB)
        Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
         Raid Devices : 3
        Total Devices : 3
          Persistence : Superblock is persistent
   
          Update Time : Thu Feb 22 13:52:32 2024
                State : clean, degraded, recovering 
       Active Devices : 2
      Working Devices : 3
       Failed Devices : 0
        Spare Devices : 1
   
               Layout : left-symmetric
           Chunk Size : 512K
   
   Consistency Policy : resync
   
       Rebuild Status : 35% complete
   
                 Name : hostname:0  (local to host hostname)
                 UUID : 3e4455f5:65e4251c:b21c1c81:13b44a8b
               Events : 26
   
       Number   Major   Minor   RaidDevice State
          0       7        0        0      active sync   /dev/loop0
          1       7        1        1      active sync   /dev/loop1
          3       7        3        2      spare rebuilding   /dev/loop3
   ```

   添加新盘后，可以看到重建操作会自动开始。

4. 完整性检查

   这里展示三盘 RAID 1 下进行检查与修复的场景：

   ```
   $ sudo mdadm --stop /dev/md0
   mdadm: stopped /dev/md0
   $ sudo mdadm --misc --zero-superblock /dev/loop0 /dev/loop1 /dev/loop3
   $ sudo mdadm --create /dev/md0 --level=1 --raid-devices=3 /dev/loop0 /dev/loop1 /dev/loop3
   mdadm: Note: this array has metadata at the start and
       may not be suitable as a boot device.  If you plan to
       store '/boot' on this device please ensure that
       your boot-loader understands md/v1.x metadata, or use
       --metadata=0.90
   Continue creating array? y
   mdadm: Defaulting to version 1.2 metadata
   mdadm: array /dev/md0 started.
   $ # 向其中一块盘写入垃圾数据
   $ sudo dd if=/dev/urandom of=/dev/loop0 bs=1M count=1 oseek=100
   1+0 records in
   1+0 records out
   1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.00406889 s, 258 MB/s
   $ sudo bash -c 'echo check > /sys/block/md0/md/sync_action'
   $ # 在 /proc/mdstat 中可以看到检查的进度
   $ cat /proc/mdstat
   Personalities : [raid6] [raid5] [raid4] [raid1] [raid0] 
   md0 : active raid1 loop3[2] loop1[1] loop0[0]
         1046528 blocks super 1.2 [3/3] [UUU]
         [===================>.]  check = 95.3% (998016/1046528) finish=0.0min speed=249504K/sec
   
   unused devices: <none>
   $ # 由于我们这里盘很小，所以检查很快就结束了
   $ cat /sys/block/md0/md/mismatch_cnt  # 获取不一致的块数
   4096
   $ # 由于我们有足够多的副本，可以尝试修复
   $ sudo bash -c 'echo repair > /sys/block/md0/md/sync_action'
   $ # 在 /proc/mdstat 中也可以看到修复的进度
   $ cat /proc/mdstat
   Personalities : [raid6] [raid5] [raid4] [raid1] [raid0] 
   md0 : active raid1 loop3[2] loop1[1] loop0[0]
         1046528 blocks super 1.2 [3/3] [UUU]
         [===========>.........]  resync = 57.4% (601984/1046528) finish=0.0min speed=300992K/sec
   
   unused devices: <none>
   $ # 修复完成后，mismatch_cnt 中仍然记录的是不一致的数量
   $ # 再执行一次 check 后 mismatch_cnt 值会变为 0
   ```

5. 监控 SMART信息

   阅读 SMART 信息可以帮助了解硬盘的健康状态。在服务器上，如果使用硬件 RAID，使用 `smartctl` 需要添加额外的参数来从 RAID 控制器获取真实的磁盘信息，例如下面的例子：

   ```
   $ sudo smartctl --scan
   /dev/sda -d scsi # /dev/sda, SCSI device
   /dev/sdb -d scsi # /dev/sdb, SCSI device
   /dev/sdc -d scsi # /dev/sdc, SCSI device
   /dev/sdd -d scsi # /dev/sdd, SCSI device
   /dev/bus/4 -d megaraid,8 # /dev/bus/4 [megaraid_disk_08], SCSI device
   /dev/bus/4 -d megaraid,9 # /dev/bus/4 [megaraid_disk_09], SCSI device
   /dev/bus/4 -d megaraid,10 # /dev/bus/4 [megaraid_disk_10], SCSI device
   /dev/bus/4 -d megaraid,11 # /dev/bus/4 [megaraid_disk_11], SCSI device
   /dev/bus/4 -d megaraid,12 # /dev/bus/4 [megaraid_disk_12], SCSI device
   /dev/bus/4 -d megaraid,13 # /dev/bus/4 [megaraid_disk_13], SCSI device
   /dev/bus/4 -d megaraid,14 # /dev/bus/4 [megaraid_disk_14], SCSI device
   /dev/bus/4 -d megaraid,15 # /dev/bus/4 [megaraid_disk_15], SCSI device
   /dev/bus/0 -d megaraid,8 # /dev/bus/0 [megaraid_disk_08], SCSI device
   /dev/bus/0 -d megaraid,9 # /dev/bus/0 [megaraid_disk_09], SCSI device
   /dev/bus/0 -d megaraid,10 # /dev/bus/0 [megaraid_disk_10], SCSI device
   /dev/bus/0 -d megaraid,11 # /dev/bus/0 [megaraid_disk_11], SCSI device
   /dev/bus/0 -d megaraid,12 # /dev/bus/0 [megaraid_disk_12], SCSI device
   /dev/bus/0 -d megaraid,13 # /dev/bus/0 [megaraid_disk_13], SCSI device
   $ sudo smartctl -a /dev/sdd  # 直接查询只能看到没有意义的控制器信息
   smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.10.0-21-amd64] (local build)
   Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org
   
   === START OF INFORMATION SECTION ===
   Vendor:               AVAGO
   Product:              MR9361-8i
   Revision:             4.68
   Compliance:           SPC-3
   User Capacity:        1,919,816,826,880 bytes [1.91 TB]
   Logical block size:   512 bytes
   Physical block size:  4096 bytes
   Logical Unit id:      0x600605b00f17786026223e2d33c1767b
   Serial number:        007b76c1332d3e22266078170fb00506
   Device type:          disk
   Local Time is:        Sun Feb 11 18:40:45 2024 CST
   SMART support is:     Unavailable - device lacks SMART capability.
   
   === START OF READ SMART DATA SECTION ===
   Current Drive Temperature:     0 C
   Drive Trip Temperature:        0 C
   
   Error Counter logging not supported
   
   Device does not support Self Test logging
   $ sudo smartctl -a /dev/bus/4 -d megaraid,8  # 添加参数可以看到真实的磁盘信息
   （内容省略）
   ```

   `smartctl -a` 的输出主要分为两个部分：information section 和 smart data section。

   Information section 展示硬盘的基本信息，包括型号、序列号、容量、固件版本等。

   Smart data section 则展示了硬盘的 SMART 信息。其中**自检信息**与**错误记录**均会显示，其他的部分视硬盘类型而定。

   SMART 还支持让磁盘自检，对应的参数为 `smartctl -t`，可以选择不同的测试参数（例如 short、long 等）

   安装 smartmontools 之后，可以启用 smartd 服务（smartd.service 或 smartmontools.service）。 该服务会每隔一段时间检查硬盘的 SMART 信息，并在发现问题时发送邮件通知管理员（在正确配置的情况下）。 默认 Debian 提供的 `/etc/smartd.conf` 的有效内容如下：

   ```
   DEVICESCAN -d removable -n standby -m root -M exec /usr/share/smartmontools/smartd-runner
   ```

   `/usr/share/smartmontools/smartd-runner` 会调用 `/etc/smartmontools/run.d/` 下的文件， 其中默认提供的 `10mail` 会使用系统的 `mail` 命令发送邮件。

   在配置中添加 `-M test` 可以在启动时发送测试邮件

6. 紧急救援

   对于 RAID 0 类型的配置（包括 RAID 10/01），`mdadm` 工具支持拼接多个（没有 mdadm superblock 的）块设备的内容组成 RAID，参考命令如下：

   ```
   sudo mdadm --build --assume-clean -c 128 --level=0 --raid-devices=8 --size=195364584 /dev/md0 /dev/mapper/disk1 /dev/mapper/disk2 /dev/mapper/disk3 /dev/mapper/disk4 /dev/mapper/disk5 /dev/mapper/disk6 /dev/mapper/disk7 /dev/mapper/disk8
   ```

   







# Ubuntu24.04 安装PHP7.4

```
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install -y php7.4 php7.4-cli php7.4-fpm php7.4-mysql php7.4-xml php7.4-curl php7.4-zip php7.4-mbstring php7.4-gd
```
