#  npm修改国内源安装electron



#### 解决办法

打开npm的配置文件

```
npm config edit
```

在空白处将下面几个配置添加上去,注意如果有原有的这几项配置，就修改

```
proxy=http://127.0.0.1:10807/
https-proxy=http://127.0.0.1:10807/
registry=https://registry.npmmirror.com
electron_mirror=https://cdn.npmmirror.com/binaries/electron/
electron_builder_binaries_mirror=https://npmmirror.com/mirrors/electron-builder-binaries/
```



# npm修改国内源

修改源地址为国内 NPM 镜像

```
npm config set registry https://registry.npmmirror.com
```

修改源地址为官方源

```
npm config set registry https://registry.npmjs.org/
```



npm安装提示'current user("nobody") does not have permission'的解决

```
npm -g config set user root
```

