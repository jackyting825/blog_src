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

      #rewrite ^(.*) https://$host$1 permanent; #重定向到https

      location / {
        # WordPress固定链接URL重写
        if (!-e $request_filename) {
          rewrite (.*) /index.php;
        }
      }
      location / {
        # WordPress固定链接URL重写
        if (!-e $request_filename) {
          rewrite (.*) /index.php;
        }
      }
      location ^~ /device/ {
    	  proxy_pass http://127.0.0.1:8080;
      }
      location ^~ /upload/ {  
        root  /aaa/bbb;
        expires   7d;
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

      #rewrite ^(.*) https://$host$1 permanent; #重定向到https

      location / {
        # WordPress固定链接URL重写
        if (!-e $request_filename) {
          rewrite (.*) /index.php;
        }
      }
      location ^~ /device/ {
    	  proxy_pass http://127.0.0.1:8080;
      }
      location ^~ /upload/ {  
        root  /aaa/bbb;
        expires   7d;
      }
      # websocket地址
      location ^~ /ws {
		    proxy_pass http://127.0.0.1:10002;
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

7.nginx常用的配置选项模板
    /etc/nginx/nginx.conf

    underscores_in_headers on; #自定义Header中含有下划线的情况 必须定义
    gzip  on;
    gzip_min_length 1k;
    gzip_buffers 16 64k;
    gzip_http_version 1.1;
    gzip_comp_level 6;
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary on;
    gzip_disable "MSIE [1-6]\.";

    #proxy_connect_timeout 600;  #nginx跟后端服务器连接超时时间(代理连接超时)
    
    proxy_buffer_size     32k;  #设置代理服务器（nginx）保存用户头信息的缓冲区大小
    proxy_buffers         4 32k;#proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
    proxy_busy_buffers_size  64k;           #高负荷下缓冲大小（proxy_buffers*2）
    proxy_temp_file_write_size  1024m;       #设定缓存文件夹大小，大于这个值，将从upstream服务器传
    client_max_body_size 100M;

    # 给后端服务器暴露获取客户端真实IP地址的头
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # websocket 支持
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout    600;  #连接成功后，后端服务器响应时间(代理接收超时)
    proxy_send_timeout    600;  #后端服务器数据回传时间(代理发送超时)
