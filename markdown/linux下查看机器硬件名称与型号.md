# linux下查看机器硬件名称与型号


```
查看机箱型号
dmidecode | grep "Product Name"

yum install smartmontools
#查看硬盘型号
smartctl --all /dev/sda

#查看cpu型号
cat /proc/cpuinfo

# 查看内存型号及插槽
dmidecode | grep -A16 "Memory Device$"


```