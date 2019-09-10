---
title: nginx常用的location匹配规则
date: 2019-09-10 09:32:36
tags:
  - nginx
---

>记录和整理一下nginx用到的匹配规则,部分内容搜集于网络上

### Nginx的location语法

location指令的作用是根据用户请求的URI来执行不同的应用，也就是根据用户请求的网站URL进行匹配，匹配成功即进行相关的操作

#### nginx常见正则匹配符号表示

    ^ ： 匹配字符串的开始位置
    $ ：匹配字符串的结束位置
    .* : .匹配任意字符，*匹配数量0到正无穷
    \. : 斜杠用来转义,\.匹配.,特殊用法
    （值1|值2|值3|值4）：或匹配模式，例：（jpg|gif|png|bmp）匹配jpg或gif或png或bmp
    i : 不区分大小写

#### 普通匹配

  >location = URI { configuration } # 精确匹配,表示完全匹配规则才执行操作
    
    例:
      # http://{domain-name}/index
      location = /index {
        [configuration A]
      }
  
  >location ^~ URI { configuration } # 非正则匹配，表示URI以某个常规字符串开头

    例:
      # URL为http://{domain_name}/images/xxx.jpg
      location ^~ /images/ {
        [ cofigurations D ]
      }

  >location [space] URI { configuration} # 前缀匹配, 匹配后，继续更长前缀匹配和正则匹配

    例:
      # URL为http://{domain_name}/images/xxx.jpg
      location /images/ {
        # 字符匹配到 /images/，继续往下，会发现 ^~ 存在
        [ cofigurations D ]
      }

#### 正则表达式匹配(~表示执行正则匹配)

  >location ~ URI { configuration } # 执行正则匹配，但区分大小写

    例:
      # http://{domain-name}/page/1匹配结尾数字为1~99时，配置生效
      location ~ /page/\d{1,2} {
          [ configuration B ]
      }

  >location ~* URI { configuration } # 执行正则匹配，但不区分大小写

    例:
      # 匹配所有URL以.jpg、.jpeg、.gif结尾时,配置生效
      location ~* /\.(jpg|jpeg|gif)$ {
          [ configuration C ]
      }

  >@符号,定义一个location，用于处理内部重定向

    例:
      location @error {
          proxy_pass http://error;
      }

      error_page 404 @error;

location的各个匹配之间的优先级顺序:

    (location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)

#### 文件及目录匹配

    *-f和!-f用来判断是否存在文件
    *-d和!-d用来判断是否存在目录
    *-e和!-e用来判断是否存在文件或目录
    *-x和!-x用来判断文件是否可执行

#### rewrite指令的最后一项参数为flag标记，flag标记有以下

    last      相当于apache里面的[L]标记，表示rewrite
    break     本条规则匹配完成后，终止匹配，不再匹配后面的规则
    redirect  返回302临时重定向，浏览器地址会显示跳转后的URL地址
    permanent 返回301永久重定向，浏览器地址会显示跳转后的URL地址

### location指令配置示例

``` json
location  = / {
  # 精确匹配 / ，主机名后面不能带任何字符串
  [ configuration A ]
}

location  / {
  # 因为所有的地址都以 / 开头，所以这条规则将匹配到所有请求
  # 但是正则和最长字符串会优先匹配
  [ configuration B ]
}

location /documents/ {
  # 匹配任何以 /documents/ 开头的地址，匹配符合以后，还要继续往下搜索
  # 只有后面的正则表达式没有匹配到时，这一条才会采用这一条
  [ configuration C ]
}

location ~ /documents/Abc {
  # 匹配任何以 /documents/ 开头的地址，匹配符合以后，还要继续往下搜索
  # 只有后面的正则表达式没有匹配到时，这一条才会采用这一条
  [ configuration CC ]
}

location ^~ /images/ {
  # 匹配任何以 /images/ 开头的地址，匹配符合以后，停止往下搜索正则，采用这一条。
  [ configuration D ]
}

location ~* \.(gif|jpg|jpeg)$ {
  # 匹配所有以 gif,jpg或jpeg 结尾的请求
  # 然而，所有请求 /images/ 下的图片会被 config D 处理，因为 ^~ 到达不了这一条正则
  [ configuration E ]
}

location /images/ {
  # 字符匹配到 /images/，继续往下，会发现 ^~ 存在
  [ configuration F ]
}

location /images/abc {
  # 最长字符匹配到 /images/abc，继续往下，会发现 ^~ 存在
  # F与G的放置顺序是没有关系的
  [ configuration G ]
}

location ~ /images/abc/ {
  # 只有去掉 config D 才有效：先最长匹配 config G 开头的地址，继续往下搜索，匹配到这一条正则，采用
    [ configuration H ]
}
```