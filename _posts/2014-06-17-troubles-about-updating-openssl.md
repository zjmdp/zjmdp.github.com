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

很明显，系统的openssl版本过于古老， 而安装freeswitch的编译和运行需要`openssl >= 1.0.0e`，第一直觉就是先把openssl升级再说，无尽的折磨从此开始。

##坑一：openssl编译
一般c项目的configure过程中会通过配置`--prefix=xxx`来设定被编译lib的安装目录，我一般习惯放到`/usr/local/xxx`，所以openssl也不例外，顺利安装到`/usr/local/openssl-1.0.1h`目录下，但freeswitch的configure还是一直提示openssl版本太低:

 ```
 configure: error: OpenSSL >= 1.0.1e and associated developement headers required
 ```

不过很快意识到自己犯了个低级错误，编译默认搜索的header和lib还是系统自带的老版本openssl，于是在freeswitch的configure过程中显示设定openssl的相关路径：

```
./configure --prefix=/usr/local CFLAGS="-I/usr/local/openssl-1.0.1h/include" LDFLAGS="-L/usr/local/openssl-1.0.1h/lib"
```

一波未平一波又起，freeswitch的configure过程又抛出libcrypt.so需要`recompile with -fPIC`，看起来是openssl的config需要指定-fPIC来编译动态库，freeswitch通过动态链接的形式依赖openssl相关的库，于是加上`shared`和`-fPIC`后重新configure和make openssl：

```
./config shared --prefix=/usr/local/openssl-1.0.1h -fPIC
```

终于freeswitch的configure和make过了，长舒一口气，但一个更大的坑还在后面。

##坑二：openssl链接
原以为freeswitch可以愉快的运行了，事与愿违，server无情的抛出以下错误：

```
symbol lookup error: /usr/local/lib/libfreeswitch.so.1: undefined symbol: EVP_aes_128_ctr
```

简单查阅资料和分析后推断应该是freeswitch运行时链接的还是系统默认的openssl（虽然编译用的是自己编译的最新的openssl）。

水平有限，没找到太好的方法让freeswitch链接到自己编译的openssl的动态库，因此想了个办法就是覆盖系统默认的openssl，两种方法：

* 将/usr/lib64下和openssl相关的库软链接到自己编译的库
* 使用下面的configure重新make和install openssl：

```
 ./config shared --prefix=/usr --openssldir=/usr/local/openssl -fPIC





OpenSSL version mismatch. Built against 10000003, you have 1000108f


 ./configure --prefix=/usr/local/openssh --sysconfdir=/etc/ssh --with-pam --with-ssl-dir=/usr/local/openssl --with-md5-passwords --mandir=/usr/share/man --with-zlib=/usr/local/zlib


service sshd stop


ln -s /usr/local/openssh/sbin/sshd /usr/sbin/sshd


```





