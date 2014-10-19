---
layout: post
title: "openssl升级的一些坑"
description: ""
category: 
tags: []
---
{% include JB/setup %}

最近由于工作需求要搭建freeswitch，整个过程比较纠结，遂总结于此留作备忘。

##起源

由于我的需求是将freeswitch跑起来，所以先按照freeswitch wiki上的介绍编译freeswitch，但configure过程便遇到问题：

```
configure: error: OpenSSL >= 1.0.1e and associated developement headers required
```

很明显，系统的openssl版本过于古老， 而安装freeswitch的编译和运行需要```openssl >= 1.0.0e```，第一直觉就是先把openssl升级再说，无尽的折磨从此开始。


##坑一：openssl编译

一般c项目的configure过程中会通过配置```--prefix=xxx```来设定被编译lib的安装目录，我一般习惯放到```/usr/local/xxx```，所以openssl也不例外，顺利安装到```/usr/local/openssl-1.0.1h```目录下，但freeswitch的configure还是一直提示openssl版本太低:

```
configure: error: OpenSSL >= 1.0.1e and associated developement headers required
```

不过很快意识到自己犯了个低级错误，编译默认搜索的header和lib还是系统自带的老版本openssl，于是在freeswitch的configure过程中显示设定openssl的相关路径：

```
./configure --prefix=/usr/local CFLAGS="-I/usr/local/openssl-1.0.1h/include" LDFLAGS="-L/usr/local/openssl-1.0.1h/lib"
```

一波未平一波又起，freeswitch的configure过程又抛出libcrypt.so需要```recompile with -fPIC```，看起来是openssl的config需要指定-fPIC来编译动态库，freeswitch通过动态链接的形式依赖openssl相关的库，于是加上```shared```和```-fPIC```后重新configure和make openssl：

```
./config shared --prefix=/usr/local/openssl-1.0.1h -fPIC
```

终于freeswitch的configure和make过了，长舒一口气，但一个更大的坑还在后面。

##坑二：openssl链接
原以为freeswitch可以愉快的运行了，事与愿违，server无情的抛出以下错误：

```
symbol lookup error: /usr/local/lib/libfreeswitch.so.1: undefined symbol: EVP_aes_128_ctr
```

看起来是因为默认的sshd是用老版本openssl编译的，于是想到升级openssh，指定依赖最新版本的openssl，于是一阵download，configure，make，make install，这里需要注意的是openssh的configure参数配置需要手动指定openssl库的地址：

```
./configure --prefix=/usr/local/openssh --sysconfdir=/etc/ssh --with-pam --with-ssl-dir=/usr/local/openssl --with-md5-passwords --mandir=/usr/share/man --with-zlib=/usr/local/zlib
```

因为考虑到openssh的重要性，覆盖默认的ssh风险比较高，因此选择先安装到```/usr/local/openssh```，然后将```/usr/sbin/sshd```符号链接到```/usr/local/openssh/sbin/sshd```中:

* 首先停止正在运行的sshd服务：```service sshd stop```
* 备份老版本sshd文件，然后删除```/usr/sbin/ssh```
* 创建符号链接
* 启动sshd服务```service sshd start```

立马通过跳板机ssh登录来测试新版本openssh的效果，一切看起来完美！


#总结
纠结了这么久，最后的效果就两点：

* 升级openssl，覆盖默认的老版本openssl，这里选择的是覆盖，即安装到```/usr```目录下，因为我没找到可以让freeswitch运行时手动指定openssl相关so路径的办法。
* 升级openssh，考虑到风险，先安装到```/usr/local/openssh```，然后通过符号链接将```/usr/sbin/sshd```链接到```/usr/local/openssh/sbin/sshd```
