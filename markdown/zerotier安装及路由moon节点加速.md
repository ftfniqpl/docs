# ZeroTier安装及路由moon节点加速

```
zerotier定义了几个专业名词:
PLANET 行星级的服务器，zerotier各地的根服务器，有日本、新加坡等地。
MOON 卫星级服务器，用户自建的私有根服务器，起到中转加速的作用。
LEAF 相当于各个枝叶，就是每台连接到该网络的机器节点
```

`充当MOON的机子最好有公网IP`

#### **移动网络对zerotier默认的PLANET节点不友好,需要自建moon节点进行通信**


```
Windows: C:\ProgramData\ZeroTier\One
Macintosh: /Library/Application Support/ZeroTier/One (在 Terminal 中应为 /Library/Application\ Support/ZeroTier/One)
Linux: /var/lib/zerotier-one
FreeBSD/OpenBSD: /var/db/zerotier-one

zerotier-idtool initmoon identity.public >>moon.json //生成moon.json文件
```

修改生成的`moon.json`文件 将"stableEndpoints": [ ] 修改下，放上分配的ip地址及zerotier的端口

如下:

```
{
 "id": "22f6cf7653",
 "objtype": "world",
 "roots": [
  {
   "identity": "22f6cf7653:0:03577cdd2129d3f0e5d689e54a7c5a221d50e8b0497c745a85301276e8aadf6eae10fa185913753b61f61926aa22cb6e72b285eb0f40c4ef1eea414edd5e969a",
   "stableEndpoints": ["192.168.192.88/9993"]
  }
 ],
 "signingKey": "4224275d0a577dbf5a3e459f1b48c121b9dc943bf163329855a8cfcaf0519e36676878aaa1984c6a6552115fe1ff766945611de0373b25a5dead4682c5c063f7",
 "signingKey_SECRET": "5227bd8e8341ef1fea2a0c9f13090f3f64c54daf72fea4040532b6ee170fa751841f5001df64e5d849e2b933391a0c056a4dc4a0fea1212d32f214012f13dc74",
 "updatesMustBeSignedBy": "4224275d0a577dbf5a3e459f1b48c121b9dc943bf163329855a8cfcaf0519e36676878aaa1984c6a6552115fe1ff766945611de0373b25a5dead4682c5c063f7",
 "worldType": "moon"
}
```


修改完保存，执行：

```
zerotier-idtool genmoon moon.json
//显示结果
# wrote 00000022f6cf7653.moon (signed world with timestamp 1550806005779)
//目录下生成了00000022f6cf7653.moon的文件
mkdir moons.d //新建文件夹
mv 00000022f6cf7653.moon moons.d //移动到moons.d文件夹
killall -9 zerotier-one //重启
zerotier-one -d
```

将生成的`00000022f6cf7653.moon`复制到其它机器的` /var/lib/zerotier-one/moons.d`目录下,并重启zerotier服务

或者使用`zerotier-cli orbit 22f6cf7653 22f6cf7653` 将MOON节点加入到客户端节点上

重启zerotier后查看 `zerotier-cli listpeers`

注意，阿里云机器需要在安全组 开启 9993的udp协议

如需要使用TCP进行中继:

Linux设备路径: /var/lib/zerotier-one/local.conf

Windows设备: c:\ProgramData\ZeroTier\One\local.conf

Mac设备: /Library/Application Support/ZeroTier/One/local.conf

添加以下配置内容来强制TCP中继:

```
{
	"settings": {
		"forceTcpRelay": true
	}
}
```

保存后重启zerotier服务即可