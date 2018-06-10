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

#### 3.Shell编程之Bash变量

  Shell中,所有变量默认都是字符串类型

  1. 用户自定义变量  变量名=变量值(等号2边不能有空格)

    用户自己定义的变量,变量名和值可随意更改

    name="zhangsan"

    echo $name #显示name的值

    1.1 变量叠加

      name="$name"isaname

    1.2  删除变量(释放变量的内存地址) unset 变量名

  2. 环境变量

    环境变量是全局变量,用户可更改值,不能更改名称

  3. 位置参数变量

    $n :n为数字,$0代表命令本身.$1-$9代表第1-第9个参数,10以上的参数需要用大括号包含,如${10}

    例:vim sum.sh

      `#!/bin/bash

        num1=$1

        num2=$2

        sum=$(($num1+$num2))

        #变量sum的和是num1+num2

        echo $sum`

        执行;./sum.sh 10 20  #./sum.sh是$0,10是$1,20是$2

        结果:30

    $* : 这个变量代表命令行中中所有的参数,$* 把所有的参数看成一个整体

    $@ : 这个变量也代表命令行中所有的参数,不过$@ 是把每个参数区分对待

    @# : 这个变量代表命令行中所有参数的个数

    例:vim /demo.sh

    `#!/bin/bash

    echo "参数是: $* "

    echo "参数也是: $@ "

    echo "参数个数是: $#"`

    执行;./demo.sh 11 22 33

    结果:参数是: 11 22 33 参数也是: 11 22 33 参数个数是: 3

    $* 和 $@ 区别

    vim ./demo.sh

    #!/bin/bash

    `for i in "$*"

    #$* 把所有的参数看成一个整体,所以执行循环1次

      do

        echo "参数是: $i"

      done

    for y in "$@"

    #$@ 是把每个参数区分对待,所有有几个参数就循环几次

      do

        echo "参数是: $y"

      done
    `

  4. 预定义变量

    $? : 最后依次执行的命令的返回结果,如果返回是0,代表上一个命令执行成功,如果返回是非0,代表上一个命令执行失败

    $$ : 返回当前进程的PID号

    $! : 后台运行的最后一个进程的进程号(PID)

  5. 接收键盘输入:read 命令

    read [选项] [变量名]

    选项

      -p "提示信息":在等待read输入时,输出提示信息

      -t 秒数: read命令会一直等待用户输入,输入次选项可以指定用户等待时间

      -n 字符数: read命令只接受指定的字符数,就会执行

      -s : 隐藏输入的数据,适用于输入密码等情况

#### 4.Shell编程之运算符

    1. declare命令

      declare声明变量类型

      declare [+/-] [选项] 变量名

        选项:用-给变量设定类型属性,用+取消变量的类型属性

      常见选项类型

        -a 将变量声明为数组类型

        -i 将变量声明为整形

        -x 将变量声明为环境变量

        -r 将变量声明为只读变量(设置为只读属性后,不能对变量进行删除,修改,取消属性的操作)

        -p 查看显示指定变量的被声明的类型

    2. 数值运算的方法

      方法1:

      [root@localhost~]# aa=11

      [root@localhost~]# bb=22

      [root@localhost~]# declare -i cc=$aa+$bb

      方法2:

        expr或者let数值运算工具

        [root@localhost~]# aa=11

        [root@localhost~]# bb=22

        [root@localhost~]# dd=$(expr $aa + $bb)

        #dd的值是aa和bb的和,注意:"+"号两侧必须有空格

      方法3:

        "$(())"或"$[运算式]"

        [root@localhost~]# aa=11

        [root@localhost~]# bb=22

        [root@localhost~]# cc=$(($aa + $bb))

        [root@localhost~]# gg=$[$aa + $bb]

    3. 变量测试(只是针对Shell,其他常用不适用.一般不常用,对脚本进行优化的时候才使用)

#### 5.Shell编程之环境变量配置文件

    `/etc/profile

    /etc/profile.d/*.sh

    /etc/bashrc

    ~/.bashrc

    ~/.bash_profile`

    /etc目录下的是系统环境变量文件,~目录下的是当前用户的环境变量配置文件
