---
title: vue-cli项目在Linux/macos下修改端口为80的问题
date: 2019-12-26 09:57:14
tags: 
  - vue
  - nodejs
---

    摘要:一直在写vue项目,但是最近需要将端口更改为80,在Linux/macos下无论是修改vue.config.js了面devServer的port为80还是在脚本命令后加启动参数--port 80,
    最后启动成功后的端口都是1024


猜测:在Linux/macos下,端口范围是有一定的权限要求的.同事在Windows下直接修改为80是完全可以的.

此时,需要将npm命令用root的身份运行,才能有足够的权限来使用80端口.因为本机安装nodejs的时候,直接是安装到当前用户下的.使用sudo命令的时候,查找的命令目录是/usr/bin/,可以在该目录下建立对应的npm命令软链接,使root能够执行npm命令

    cd /usr/bin/  # 进入到目录
    which npm  # 查看当前npm安装路径
    sudo ln -s /opt/node-v10.16.3-linux-x64/bin/npm # 建立软链接
    sudo ln -s /opt/node-v10.16.3-linux-x64/bin/node  # 建立软链接
    sudo npm -v # 检测root下命令是否可用

执行完毕后,使用sudo npm run serve即可正常在80端口下启动.