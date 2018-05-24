---
title: Shell基础
date: 2018-05-20 18:52:36
tags:
  - linux
  - shell
---

    摘要:
      Shell是一个命令行解释器,它为用户提供了一个向Linux内核
      发送请求以便运行程序的界面系统级程序.用户可以用Shell来启动,挂起,
      停止甚至是编写一些程序.
      Shell还是一个功能相当强大的编程语言.易编写,意调试,灵活性较强.
      Shell是解释执行的语言,在Shell中可以直接调用Linux系统命令.

#### 1.脚本的执行方式

    1. echo 输出命令
        echo [选项] [输出内容]
        选项:
          -e : 支持反斜杠控制的字符转换

    2. 编写第一个脚本
      vim hello.sh
      #!/bin/bash
      # this is hello program!
      echo "hello"

    3. 脚本执行
      1.赋予执行权限  
        chmod 755 ./hello.sh
        ./hello.sh
      2.通过bash调用执行脚本
        bash ./hello.sh
      3.使用sh命令执行
        sh ./hello.sh

#### 2.Bash的基本功能

    1. 命令的别名,很多泛指为Linux下的命令,其实本质是属于Bash
      `alias` 查看系统中所有的命令的别名
    2. 设置命令别名
      alias 别名= '原命令'
      alias ll='ls -l'  #给ls -l 设置别名ll

      以上设置别名的方式只是当次有效,系统重启后无效.要设置别名永久有效,
      可以写入环境变量中
      vim ~/.bashrc
      alias ll='ls -l'
      保存,执行source ~/.bashrc即可

    3. 删除别名
      unalias 别名
      unalias ll
      unalias是删除临时别名的,永久生效的别名需要删除环境变量中的配置

    4. 命令的生效顺序
      第一顺位执行用绝对路径或者相对路径的命令
      第二顺位执行别名
      第三顺位执行Bash的内部命令
      第四顺位执行按照$PATH环境变量定义的目录查找顺序找到的第一个命令

      注:因为别名的执行顺序是高于$PATH下的命令的,
      所以一般情况下请勿将别名设为与其他原始命令相同的命令.
