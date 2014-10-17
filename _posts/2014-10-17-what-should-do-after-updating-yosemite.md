---
layout: post
title: "升级Yosemite后需要做的事"
description: ""
category: 
tags: []
---
{% include JB/setup %}

#升级Yosemite后需要做的事

##Homebrew

###问题
当你运行brew命令会出现以下错误：

```
/usr/local/bin/brew: /usr/local/Library/brew.rb: /System/Library/Frameworks/Ruby.framework/Versions/1.8/usr/bin/ruby: bad interpreter: No such file or directory
/usr/local/bin/brew: line 21: /usr/local/Library/brew.rb: Undefined error: 0
```

从错误信息看是ruby版本出现了问题，原来的brew使用的是ruby 1.8，但更新Yosemite后ruby也更新到了2.0，因此原来的ruby路径已经无效了。

###解决方法
首先备份```/usr/local/Cellar```下安装的程序，brew安装的程序都在```/usr/local/Cellar```目录下，执行以下命令备份：

```
tar -cvf Cellar.tar.gz /usr/local/Cellar
```

删除老版本信息：

```
rm -rf /usr/local/Cellar /usr/local/.git
```

重新安装：

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

恢复Cellar:

```
tar -zxvf Cellar.tar.gz && mv usr/local/Cellar /usr/local/Cellar
```


##JDK缺失
所有依赖Java Runtime的程序都无法运行，包括Eclipse，Intellij，Webstom等。

###解决方法：
在终端执行 ```java```，即会跳出以下对话框：

![Java](http://www.tu265.com/di-4c980881e2e8cf2f5d2088633dc75c44.png)


点击“更多信息”即可跳转apple官网下载jdk，或直接点[这里](http://support.apple.com/kb/DL1572)。

##Proxifier无法initialize kext

更新Yosemite后运行Proxifier会出现以下错误提示：

![java](http://www.tu265.com/di-210ee0fe76ed24664f3a138d10f3a16b.png)

###解决方法
执行以下命令并重启系统即可解决：

```
sudo nvram boot-args="debug=0x146 kext-dev-mode=1"
```