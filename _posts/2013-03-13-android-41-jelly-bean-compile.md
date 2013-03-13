---
layout: post
title: "Android 4.1 (Jelly Bean) 源码编译过程总结"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Google官方发布了Android 4.1 Jelly Bean的源码，本人第一时间将源码托了下来(花了一个通宵)，又花了一早上时间编译ROM并刷到自己的Galaxy Nexus中，作为Android开发者和爱好者，体验源码下载、编译、刷机的整个过程还是很有意义的，然而在编译和刷机过程中也遇到了一些问题，晚上抽了点时间将整个过程总结一下，同时也希望能帮助到想通过自己编译刷机的朋友，废话不多说了～切入正题。

1. 编译环境搭建，本人使用的是ubuntu 12.04，下面所有过程都基于该平台。具体可参考：<http://source.android.com/source/initializing.html>
2. 下载Android 4.1源码，参考：<http://source.android.com/source/downloading.html>，源码下载过程中经常会遇到下载失败的情况，非常浪费时间，所以编写一个失败重传的脚本可以减少很多不必要的麻烦。将下面的shell脚本保存为download.sh文件放到源代码目录中，执行`./download.sh`开始下载（替代文档中最后一步repo sync，其余步骤必须按照文档中介绍的一步一步来）{% highlight bash %}
#!/bin/bash
echo "======start repo sync======"
repo sync
while [ $? == 1 ]; do
echo "======sync failed, re-sync again======"
sleep 3
repo sync
done{% endhighlight %}
3. 接下来准备开始编译rom，可参考<http://source.android.com/source/building.html>，首先输入`source ./build/envsetup.sh`，然后输入`lunch full_maguro-user`（不同平台参数不一样，具体参考文档说明），接下来执行`make -j4`编译rom(根据机器CPU的核心数量来设定参数)，下来就是漫长的等待过程了（笔者机器性能不给力，整个编译过程大概花费了3个半小时）。如果`make -j4`的执行过程中一开始提示jdk版本不对，那么按照下面步骤来解决：
	* 从Oracle官方下载jdk1.6.0_33.bin <http://www.oracle.com/technetwork/java/javase/downloads/jdk6-downloads-1637591.html>
	* 执行`sudo chmod a+x /home/jamie/jdk/jdk1.6.0_33.bin`（注意这里的目录和你机器上的目录是不一样的）
	* `sudo ./home/jamie/jdk/jdk1.6.0_33.bin`
	* `sudo gedit /etc/profile`，尾部添加 
	{% highlight bash %}
#set java environment
	export JAVA_HOME=/home/jamie/jdk1.6.0_33
	export JRE_HOME=/home/jamie/jdk1.6.0_31/jre
	export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
	export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PA {% endhighlight %}
 	* 执行`source /etc/profile`
 	* `sudo update-alternatives --install /usr/bin/java java /home/jamie/jdk1.6.0_33/bin/java 300`
 	* `sudo update-alternatives --install /usr/bin/javac javac /home/jamie/jdk1.6.0_33/bin/javac 300`
 	* `sudo update-alternatives --config java`选择所需要的jdk版本
4. 编译完成后会在android_souce/out/target/product/maguro/中生成一大堆img文件，这就是编译生成的刷机rom了
5. 按照google官方文档的步骤接下来就是刷机了，然而结果很悲剧，由于官方文档针对的是模拟器，而笔者编译的是Galaxy Nexus，出现刷完直接黑屏无法启动的情况，突然想到昨天网上看到google发布了Android 4.1的二进制文件，而这些文件正好是蓝牙、wifi、显卡等驱动，google将这些驱动单独提供下载，并未集成到源码中，所以我们需要将其编译进rom中，步骤如下：
	* 打开<https://developers.google.com/android/nexus/drivers>，下载Galaxy Nexus对应的4个文件并解压到源代码的目录，解压出来的4个文件是4个shell脚本，分别为extract-broadcom-maguro.sh，extract-imgtec-maguro.sh，extract-invensense-maguro.sh，extract-samsung-maguro.sh 
	* 分别执行这4个脚本，执行期间会要求输入“I ACCEPT”  
	* 重新执行make -j4进行编译，这次编译过程时间很短
6. 完成上面的驱动集成后，输入 `cd /home/jamie/android_source/out/host/linux-x86/bin`（该目录下有fastboot文件）, 然后输入`sudo ./fastboot flashall` -w开始刷机，一般情况下这一步会出现以下错误提示：`neither -p product specified nor ANDROID_PRODUCT_OUT set`，原因是没有设置rom的位置，解决方法要么使用`-p`参数，后面跟着rom的位置，或者配置环境变量`ANDROID_PRODUCT_OUT`，过程如下：
	* 输入`sudo gedit /etc/profile`，在尾部添加`export ANDROID_PRODUCT_OUT=/home/jamie/android_source/out/target/product/maguro`，保存并退出
	* 然后输入`source /etc/profile`使刚设置的环境变量立即生效，最后等待刷机完成，完成后手机会自动重启，大功告成！


