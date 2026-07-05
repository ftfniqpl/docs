# linux大于2T硬盘分区并挂载


下面是简单步骤

```
1，将磁盘上原有的分区删除掉： 
进入：#parted /dev/sdb 
查看：（parted）p 
删除：（parted）rm 1 
（parted）rm 2

2,将磁盘格式变成gpt的格式（因为parted只能针对gpt格式的磁盘进行操作） 
转换：（parted） mklabel gpt 
设置单位为TB：（parted） unit MB（GB，TB） 
分区：（parted） mkpart primary 1 500 （分第一个主分区500MB） 
分区：（parted） mkpart primary 501 1000 (分第二个主分区500MB) 
分区：（parted） mkpart logical 1001 2000 (分第三个逻辑分区1000MB) （parted的逻辑分区不用先分扩展分区，直接一步到位） 
查看：（parted） p 
退出：（parted）quit ( parted分区自动保存，不用手动保存 )

3，格式化已经分好的区 
# mkfs -t ext4 /dev/sdb1 
4，挂载 
# mount /dev/sdb1 /mnt 
5,开机自动挂载： 
# echo “/dev/sdb1 /mnt ext4 defaults 0 0” >>/etc/fstab

```