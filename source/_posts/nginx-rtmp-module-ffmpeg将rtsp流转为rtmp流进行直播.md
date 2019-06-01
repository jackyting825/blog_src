---
title: nginx-rtmp-module+ffmpeg将rtsp流转为rtmp流进行直播
date: 2019-05-31 16:53:11
tags:
  - nginx
  - rtmp
  - rtsp
  - ffmpeg
---

### RTSP、RTMP、HTTP协议比较

#### 共同点

1.都是用在应用层的协议

2.理论上这三种协议都可以做直播和点播，但直播一般用RTSP和RTMP点播用HTTP

#### 不同点

1.HTTP协议（HyperText Transfer Protocol，超文本传输协议)，是因特网上应用最为广泛的一种网络传输协议，所有的WWW文件都必须遵守这个标准,HTTP是一个基于TCP/IP通信协议来传递数据(HTML 文件, 图片文件, 查询结果等).所以HTTP不是流媒体协议，RTMP和RTSP是流媒体协议

2.RTMP是Real Time Messaging Protocol（实时消息传输协议）的首字母缩写。该协议基于TCP，是一个协议族，包括RTMP基本协议及RTMPT/RTMPS/RTMPE等多种变种。RTMP是一种设计用来进行实时数据通信的网络协议，主要用来在Flash/AIR平台和支持RTMP协议的流媒体/交互服务器之间进行音视频和数据通信,RTMP一般传输flv,f4v格式流.

3.RTSP（Real Time Streaming Protocol），RFC2326，实时流传输协议.RTSP以客户端方式工作，对流媒体提供播放、暂停、后退、前进等操作.RTSP传输的一般是TS、MP4格式的流，其传输一般需要2~3个通道，命令和数据通道分离。使用RTSP协议传输流媒体数据需要有专门的媒体播放器和媒体服务器，也就是需要支持RTSP协议的客户端和服务器。

### ffmpeg简介

FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。它包括了目前领先的音/视频编码库libavcodec.可以轻易地实现多种视频格式之间的相互转换，例如可以将摄录下的视频avi等转成现在视频网站所采用的flv格式

### nginx+nginx-rtmp-moudle安装

分别下载nginx和nginx-rtmp的源码然后进行编译即可.在此,为了方便我是直接使用的docker的tiangolo/nginx-rtmp镜像,docker安装参考上一篇初识docker文档

`docker pull tiangolo/nginx-rtmp` // 拉取nginx-rtmp镜像

`docker run -it --name nginx-rtmp tiangolo/nginx-rtmp -p 1935:1935` // 第一次运行容器,取个别名,后续可直接使用 `docker start nginx-rtmp`

使用`netstat -tunlp | grep 1935` 检测1935端口是否正在监听,正常情况是正在监听中

### ffmpeg安装

`sudo apt install ffmpeg` // 安装ffmpeg(我当前环境deepin,仓库里面自带ffmpeg包)

其他操作系统需要去官网下载对应的安装包即可或者按照官方文档添加对应系统的ppa进行安装即可.

### ffmpeg将rtsp转码为rtmp

使用ffmpeg命令,将rtsp转码为rtmp.ffmpeg参数项很多,未对其深究,直接参考网友的命令的.-i后面是rtsp流地址.

`ffmpeg -re  -rtsp_transport tcp -i "rtsp://184.72.239.149/vod/mp4://BigBuckBunny_175k.mov" -f flv -vcodec libx264 -vprofile baseline -acodec aac -ar 44100 -strict -2 -ac 1 -f flv -r 10 -s 1280x720 -q 10 "rtmp://127.0.0.1:1935/live/demo"`

    在执行转码命令过程中,可能会报信息类似 Past duration 0.999992 too large 的警告错误,经查询资料,是在-r参数后面
    指定的视频帧率参数导致的.rtsp://184.72.239.149/vod/mp4://BigBuckBunny_175k.mov这个地址是网络上的地址,
    刚开始copy网友命令的时候参数值为25,就会报以上错误.使用VLC media player播放其rtsp地址,
    发现其rtsp源本身视频就很模糊,所以但是后面直接改为10就不报错的.所以这个值得根据具体的情况进行调整

### 使用VLC media player播放转换后的rtmp地址

打开VLC media player播放器.在工具栏"媒体->打开网络串流"然后输入rtmp://127.0.0.1:1935/live/demo点击确定即可进行直播预览转换后的rtmp视频流

![](/images/rtmp-result.png)

**注:关于测试rtsp地址问题,上面的地址我测试的时候能够使用,但是不能保证以后能够一直正常使用,所以有网友图文讲解了使用VLC media player自制rtsp流.小伙伴的力量强大!其地址如下:https://blog.csdn.net/taoerit/article/details/51920018 
为了防止地址失效,我将页面截了一张完整图.图片如下:**

![](/images/build_push_rtsp.jpg)