# linux 清空 last和lastb及history

```
清除登陆系统成功的记录
echo "">/var/log/wtmp  #清空last 此文件默认打开时乱码，可查到ip等信息

清除登陆系统失败的记录
echo "">/var/log/btmp  #清空lastb 此文件默认打开时乱码 ， 可查到 登陆失败 信息
 
清除历史执行命令
history -c  # 清空历史执行命令
```