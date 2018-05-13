---
title: linux下nginx部署配置多站点
date: 2018-05-13 11:25:50
tags:
  - linux
  - nginx
---

    摘要:
      有时候你想在一台服务器上为不同的域名运行不同的站点。
      比如www.siteA.com作为博客，www.siteB.com作为论坛。你可以把两个域名的IP都解析到你的服务器上，
      但是没法在Nginx的根目录里同时运行两个不同的网站.
      这时可以在nginx上面配置部署多个站点(使用nginx虚拟目录),为你节省服务器费用.
      假设你把博客放在”/home/user/www/blog”下，论坛放在”/home/user/www/forum”下。下面我们就开始进行配置:

1.在Nginx配置目录下，创建一个”vhost”目录。本例假设Nginx是默认安装，配置目录在”/etc/nginx”

`sudo mkdir /etc/nginx/vhost #创建保存站点配置文件的目录`

2.创建siteA的配置文件

'sudo vim /etc/nginx/vhost/siteA.conf #打开该文件(没有的话保存后会自动新建)'

在文件里面输入以下配置内容(具体的相关目录及location内容根据自己实际情况修改,下面只是nginx配置文件的基本结构,其实可以拷贝nginx自带的配置文件到vhost目录下,然后对文件内容进行修改):

    server {
      listen 80; # 监听端口
      server_name www.siteA.com siteA.com; # 站点域名
      root /home/user/www/blog; # 站点根目录
      index index.html index.htm index.php; # 默认导航页
      location / {
        # WordPress固定链接URL重写
        if (!-e $request_filename) {
          rewrite (.*) /index.php;
        }
      }

3.跟第二步一样,创建siteB的配置文件.("server_name”和"root”目录的内容和siteA不同)

'sudo vim /etc/nginx/vhost/siteB.conf #打开该文件(没有的话保存后会自动新建)'

在文件里面输入以下配置内容(具体的相关目录及location内容根据自己实际情况修改,下面只是nginx配置文件的基本结构,其实可以拷贝nginx自带的配置文件到vhost目录下,然后对文件内容进行修改):

    server {
      listen 80; # 监听端口
      server_name www.siteB.com siteB.com; # 站点域名
      root /home/user/www/blog; # 站点根目录
      index index.html index.htm index.php; # 默认导航页
      location / {
        # WordPress固定链接URL重写
        if (!-e $request_filename) {
          rewrite (.*) /index.php;
        }
      }

4.打开编辑nginx的配置文件

`sudo vim /etc/nginx/nginx.conf`

将我们第一步创建的虚拟目录的路径增加到nginx.conf文件中去,将下面的内容加入到”http {}”部分的末尾

    http {
      ...
      include /etc/nginx/vhost/*.conf;
    }

5.重启nginx服务(注意:所有的配置文件修改保存后,先不急重新加载配置,先使用`nginx -t`测试下文件内容是否有错在进行重新加载配置操作)

`sudo service nginx restart`

6.访问www.siteA.com和www.siteB.com，你将发现浏览器会打开不同的站点

nginx禁止ip访问的小技巧:

假如你的Nginx根目录设在”/home/user/www”，你想阻止别人通过”http://IP地址/blog”或”http://IP地址/forum”来访问你的站点，最简单的方法就是禁止IP地址访问。方法如下：

打开Nginx网站默认配置文件，记得先备份

`sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default_bak #备份原来的默认文件`

`sudo vim /etc/nginx/sites-available/default #编辑文件`

将其所有内容删除，只留以下配置

    server {
      listen 80 default_server;
      server_name _;
      return 404;
    }

然后重启nginx或者`nginx -s reload`使配置文件生效,别人将无法通过IP地址访问网站了

如果你不想禁止IP地址访问整个目录，只是要防止别人通过IP访问你的博客和论坛。那就需要禁止”/blog”和”/forum”的目录访问

打开Nginx网站默认配置文件，同上面一样，记得先备份一下

在"server { }"节点的部分加上以下配置,然后重启nginx或者reload nginx配置即可.

    location ^~ /blog/ {
      deny all;
    }
    location ^~ /forum/ {
      deny all;
    }
