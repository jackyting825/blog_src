---
title: 使用AES加密密钥大于128抛出Illegal key size or default parameters
date: 2019-12-26 09:46:48
tags: java
---

    摘要:使用AES加密时，当密钥大于128时，代码会抛出java.security.InvalidKeyException: Illegal key size or default parameters.

刚入职公司后,第一次check运行项目的时候,连接配置中心的时候,密码使用了加密方式.抛出异常"java.lang.IllegalStateException: Cannot decrypt: key=spring.euraka.config.password....Caused by: java.security.InvalidKeyException: Illegal key size..."查阅相关资料后,得知Illegal key size or default parameters是指密钥长度是受限制的，java运行时环境读到的是受限的policy文件。文件位于${java_home}/jre/lib/security/.这种限制是因为美国对软件出口的控制。
解决办法：去掉这种限制需要下载Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files.

文件的下载链接可使用搜索引擎搜索JCE Unlimited download即可找到多个版本的链接地址,然后选择对应版本的即可.我当前机器的JDK版本是8,下载链接如下 https://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html

下载完成后,解压.阅读里面的readme.txt文档.大致就是替换掉${java_home}/jre/lib/security/目录下的2个jar文件
