---
title: 编译安装nginx添加rtmp模块
date: 2019-06-01 11:30:17
tags:
  - nginx
  - rtmp
---

    摘要:nginx源码编译添加rtmp模块实现视频推流服务器
    环境:deepin linux 15.10.1(基于debian9)

### 安装依赖库

执行命令前,最好执行一次sudo apt update更新下仓库

`sudo apt install autoconf automake`

`sudo apt install libpcre3 libpcre3-dev`

`sudo apt install openssl`

`sudo apt install libssl-dev`

### 下载nginx和nginx-rtmp-module源码

进入到一个目录(可以自己新建),然后clone nginx和rtmp模块的源码,地址可在github上面查找对应的仓库,然后进行clone操作(需要安装git)

`git clone https://github.com/nginx/nginx.git --depth=1` // clone nginx源码,指定克隆深度depth为1即表示只克隆最近一次commit(clone时间大幅缩短)

`git clone https://github.com/arut/nginx-rtmp-module.git --depth=1` // clone nginx-rtmp-module源码

进入到nginx源码目录,有一个auto文件夹，里面有一个名为configure的文件.通过命令参数调用该文件,生成MakeFile

`cd nginx` // 进入到nginx源码目录

`./auto/configure --prefix=/opt/nginx --with-http_ssl_module --with-http_v2_module --with-http_flv_module --with-http_mp4_module --add-module=../nginx-rtmp-module/`

`ls -al` // 查看当前目录(nginx)下的文件,会有一个产生的MakeFile文件

### 编译和安装

当前目录还是位于上一步的nginx目录

`make` // 编译

`sudo make install` // 安装

### 查看结果

`ls -l /opt/nginx/` // 查看opt目录下nginx目录的内容

`sudo /opt/nginx/sbin/nginx` // 启动nginx服务

浏览器打开localhost,正常就能打开nginx默认的首页面

### nginx 推流配置

`sudo vim /opt/nginx/conf/nginx.conf`

    rtmp {
      server {
          listen 1935;
          application rtmplive_demo {
              live on;
              max_connections 1024;
          }
          application hlsvideo {
              live on;
              hls on;
              hls_path /home/bz/Desktop/video/hlsvideo; # 推流存放文件夹,自定义
              hls_fragment 1s;
          }
      }
    }

    location ^~ /hlsvideo {
      types {
        application/vnd.apple.mpegurl    m3u8;
        video/mp2t ts;
      }
      root /home/bz/Desktop/video; # 此处不能写/home/bz/Desktop/video/hlsvideo,因为路径中带了一层hlsvideo了,如果写上hlsvideo会导致读取m3u8文件404
      add_header Cache-Control    no-cache;
    }

`sudo /opt/nginx/sbin/nginx -t` // 测试配置文件是否ok

`sudo /opt/nginx/sbin/nginx -s reload`

测试rtmp推流

`ffmpeg -re -i ./龙珠超.布罗利.mp4 -vcodec libx264 -vprofile baseline -acodec aac -ar 44100 -strict -2 -ac 1 -f flv -s 1280x720 -q 10 rtmp://192.168.100.31:1935/rtmplive_demo/longzhuchao`

    注:rtmp://192.168.100.31:1935/rtmplive_demo/longzhuchao rtmp流地址,其中rtmplive_demo必须和nginx.conf中
    application中的rtmplive_demo名称必须一致,否则导致推流不成功

打开VLC Media Player测试

在工具栏"媒体->打开网络串流"然后输入rtmp://192.168.100.31:1935/rtmplive_demo/longzhuchao点击确定即可进行直播预览转换后的rtmp视频流.效果如图

![](/images/push_rtmp_res.png)


测试HLS推流

`ffmpeg -re -i ./龙珠超.布罗利.mp4 -vcodec libx264 -vprofile baseline -acodec aac -ar 44100 -strict -2 -ac 1 -f flv -s 1280x720 -q 10 rtmp://192.168.100.31:1935/hlsvideo/longzhuchao`

    注:rtmp://192.168.100.31:1935/hlsvideo/longzhuchao,其中hlsvideo必须和nginx.conf中
    application中hlsvideo名称必须一致,否则导致推流不成功

打开VLC Media Player测试

HLS测试地址是http协议的.访问路径是nginx中http节点下server节点配置的.此处是http://192.168.100.31/hlsvideo/longzhuchao.m3u8

在工具栏"媒体->打开网络串流"然后输入http://192.168.100.31/hlsvideo/longzhuchao.m3u8点击确定即可进行直播预览转换后的rtmp视频流.效果如图

![](/images/m3u8-res.png)