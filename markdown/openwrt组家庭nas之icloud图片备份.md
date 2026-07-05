# openwrt组家庭nas之icloud图片备份

```
docker create \		#不要改
--name icloudpd \		#可以改，只是容器的名字，部署多个容器时，必须要改，不能一致
--hostname icloudpd_boredazfcuk \		#不用改
--network bridge \		#openwrt环境下，用bridge桥接即可
--restart=always \		#不用改
--env user=root \		#openwrt环境下，默认只有root用户，作者建议与宿主系统用户名一致，避免权限问题
--env user_id=0 \		#openwrt环境下，root用户的id是0
--env group=root \		#openwrt环境下，root用户的用户组也是root
--env group_id=0 \		#openwrt环境下，root组id也是0
--env apple_id="xxxxxxx@icloud.com" \		#填写你的iCloud账号
--env authentication_type=2FA \		#身份验证类型，如果你启用了两步验证，则为2FA,否则为Web
--env folder_structure={:%Y%m} \		#下载目录的文件夹结构，我设置的是/年月结构	
--env auto_delete=True \		#扫描iCloud中“最近删除”的照片和视频，并在本地下载目录中也删除对应的照片和视频。（如果您在iCloud中恢复照片，它将再次下载）
--env synchronisation_interval=86400 \		#同步间隔时间，可设置为以下时段：21600（6 小时）、43200（12 小时）、86400（24 小时）、129600（36 小时）、172800（48 小时）和 604800（7 天）。默认为 86400 秒。设置较短的同步周期时要小心，Apple倾向于限制过于频繁地访问其服务器的连接。设置小于 12 小时的值将显示警告，因为 Apple 可能会限制您。我设置的是每12小时同步一次。
--env icloud_china=True \		#如果设置了这个变量，它将使用云上贵州icloud.com.cn，而不是icloud.com 作为下载源。推荐设置上这个参数。
--env skip_check=True \    #设置是否跳过已下载检查（增量同步），默认为False（不跳过），也就是默认为每次启动只同步新增内容，如果要重新完全同步，可设置变量skip_check的值为True
--env photo_size=original \     #默认为原始尺寸, 可选original(原始尺寸), medium(中等质量), thumb(缩略图)
--env convert_heic_to_jpeg=True \     #设置此变量则在下载时将heic文件转换为jpeg,同时保留原始文件
--env jpeg_quality=100 \     # 图片质量, 数字0(最低质量)到100(最高质量), 默认90
--env TZ=Asia/Shanghai \		#设置容器时间为东八区时间
--env download_path=/home/root/iCloud \		#容器中的目录，此变量设置上比较好，不设定的话，默认也是这个
--volume /mnt/sdb1/config/.icloudpd_config:/config \		#配置文件挂载路径。冒号前面是openwrt环境的绝对路径，冒号后面是映射到容器里的路径，你需要修改的就是冒号前面的。
--volume /mnt/sdb1/photos:/home/root/iCloud \		#照片下载路径。冒号前面是openwrt环境的绝对路径，冒号后面是映射到容器里的路径，你需要修改的就是冒号前面的。
# boredazfcuk/icloudpd		#不要改
miniers/icloudpd
```

```
cd /mnt/sdb1/photos
touch .mounted

查看错误信息
docker logs -f --tail 100 容器名
```

```
docker exec -it icloudpd sync-icloud.sh --Initialise



php预览图:https://files.photo.gallery

sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

安装php环境:
opkg update
opkg install imagemagick
opkg install php8 php8-cli php8-fpm php8-mod-dom php8-mod-exif php8-mod-fileinfo php8-mod-filter php8-mod-gd php8-mod-session php8-mod-mbstring php8-mod-zip php8-mod-curl php8-mod-opcache php8-pecl-imagick
```


docker 安装git 服务器

```

docker run -d --privileged=true --restart=always --network bridge --name=wikitten -p 10800:80 -v /mnt/sdb1/wiki:/data wikitten:latest

docker run -d --privileged=true --restart=always --network bridge --name photos -p 10810:80 -v /mnt/sdb1/photos:/data photos:latest
```