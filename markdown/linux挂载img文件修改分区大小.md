# linux挂载img文件修改分区大小

1. 挂载img镜像

   ```
   losetup -fP --show your_image.img
   ```

2. 扩容img文件本身大小 (如果需要扩大)

   ```
   # 在末尾增加1GB 
   dd if=/dev/zero bs=1M count=1024 >> your_image.img
   # 重新读取分区表
   losetup -c /dev/loop0
   ```

3. 调整分区大小(使用parted)

   ```
   parted /dev/loop0
   (parted) print
   # 记住要修改的分区编号 (例如 2)
   (parted) resizepart 2 100% # 将2号分区扩大到末尾
   (parted) quit
   ```

4. 调整文件系统大小

   ```
   对于ext文件系统
   e2fsck -f /dev/loop0p2 # 检查扩容的分区文件系统
   resize2fs /dev/loop0p2 # 扩容
   
   对于xfs文件系统
   mount /dev/loop0p2 /mnt
   xfs_growfs /mnt
   ```

5. 卸载镜像

   ```
   losetup -d /dev/loop0
   ```

   