# mac下部署IntelliJIDEALicenseServer

#### 1.编辑一个 com.phpdragon.IntelliJIDEALicenseServerDarwinAmd64.plist 文件,文件名称自己定义 

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>KeepAlive</key>
        <false/>
        <key>RunAtLoad</key>
        <true/>
        <key>Label</key>
        <string>com.phpdragon.IntelliJIDEALicenseServerDarwinAmd64</string>
        <key>ProgramArguments</key>
            <array>
                <string>/Users/phpdragon/Server/IntelliJIDEALicenseServer/IntelliJIDEALicenseServer_darwin_amd64</string>
                <string>-l</string>
                <string>0.0.0.0(绑定的IP地址)</string>
                <string>-p</string>
                <string>8888(大于1024的端口号)</string>
                <string>-u</string>
                <string>phpdragon(显示被激活的用户名)</string>
            </array>
    </dict>
</plist>
```

#### 2. /Users/phpdragon/Server/IntelliJIDEALicenseServer/IntelliJIDEALicenseServer_darwin_amd64 -h   #查看启动参数


#### 3. 编辑完毕之后拷贝 com.phpdragon.IntelliJIDEALicenseServerDarwinAmd64.plist 文件至 ~/Library/LaunchAgents/ 目录下。

#### 4 1. 加载配置好的.plist文件:
```
launchctl load  ~/Library/LaunchAgents/com.phpdragon.IntelliJIDEALicenseServer.plist
```

#### 4 2.查看配置是否加载生效：
```
launchctl list | grep IntelliJIDEALicenseServer
```

#### 4 3.使用 launchctl 命令管理你的服务，使用 -h 参数来获取参数说明帮助。
```
launchctl start com.phpdragon.IntelliJIDEALicenseServerDarwinAmd64  #启动服务
launchctl stop com.phpdragon.IntelliJIDEALicenseServerDarwinAmd64  #停止服务

launchctl list | grep IntelliJIDEALicenseServer | awk '{print $3}' | xargs launchctl start
launchctl list | grep IntelliJIDEALicenseServer | awk '{print $3}' | xargs launchctl stop
```

#### 4 4.检查服务是否启动:
```
telnet 127.0.0.1 8888

Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.

```