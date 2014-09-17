---
layout: post
title: "如何在mac下工作"
description: ""
category: 
tags: []
---
{% include JB/setup %}

#拥抱Mac之码农篇
![](http://img4.picbed.org/uploads/2014/09/a71ea8d3fd1f4134a83e4315271f95cad1c85e4b.jpg)

使用Mac大概两年时间，之前用着公司配的一台27寸的iMac，无奈机械硬盘严重拖慢速度，影响工作心情，于是入手Macbook Retina 13，这两年的开发工作全部在Mac上完成，也积累了一点心得，遂总结此文，文章主要介绍一些我认为可以提高程序员工作效率的工具软件，希望对使用Mac的码农有点帮助。


##包管理
Mac系统上主要的包管理有[Macport](https://www.macports.org/index.php)和[Homebrew](http://brew.sh)，类似于Debian系列的apt-get，Redhat的yum，主要用来安装一些开源软件，这些工具的存在大大简化了开源软件的安装过程，要不然安装一个软件可能需要提前安装一大堆依赖的软件。

网上貌似普遍推荐Homebrew，所以当时也直接选择了Homebrew，两者的优缺点大家可以Google一下，按网上的说法主要是两者对依赖包处理方式不一样，引用[知乎](http://www.zhihu.com/question/19862108)上的一个回答：

>Flink是直接编译好的二进制包，MacPorts是下载所有依赖库的源代码，本地编译安装所有依赖，Homebrew是尽量查找本地依赖库，然后下载包源代码编译按照。 
Flink容易出现依赖库问题，MacPorts相当于自己独立构建一套，下载和编译的东西太多太麻烦，Homebrew的方式最合理。


Homebrew通过以下命令安装即可：

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Homebrew上所有的软件以ruby代码封装成formula的形式提供，通过命令```brew install xxx```下载formula，得到formula中定义的软件的地址，通过wget获取软件安装包，然后本地解压编译。

###常用命令介绍

####搜索软件包
在安装之前可以先搜索想要的安装包，这里以搜索macvim为例，输入以下命令：

`brew search macvim`

####安装软件包

`brew install macvim`

####列出已安装的软件包

`brew list`

####删除软件包

`brew uninstall macvim`

####升级过时的软件包


* `brew update`

* `brew upgrade`



##配置快捷键
作为一个程序员，我相信Control键是最常使用的键位之一，而键盘默认Control位置按起来十分变扭，容易按错，我的做法是将Caps Lock映射成Control键，个人感觉体验很赞。



##终端
终端是码农的利器，一个好的终端会带来效率的提升，这里推荐[iTerm2](http://iterm2.com/index.html)，很方便的快捷键呼出和隐藏，强大的分屏支持，方便的历史命令自动完成，丰富的UI定制等等，具体可参考[iTerm 2 Features](http://iterm2.com/features.html)。

这里简单介绍一下我常用的iTerm2快捷键：

* cmd + t： 新建标签页， cmd + 数字：
* ctrl + h：清空当前行
* cmd + d： 垂直分屏， cmd + shift + d：水平分屏， cmd + [ 和 cmd + ]：分屏切换
* cmd + ; : 历史命令自动完成
* cmd + shift + h：剪贴板历史

强烈建议大家稍微花点时间学习并打造一个适合自己的iTerm2。

##Shell
我之所以喜欢用Mac工作，很大一部分原因就是喜欢Mac上的Shell环境，熟悉Shell的朋友都知道命令行界面在大部分情况下是可以高效替代图形界面，但因为命令行界面使用门槛高，而且图形界面长得更讨人喜欢，所以命令行界面才沦为少数人的工具，可以说是码农专用工具。

这里毫无疑问推荐[oh-my-zsh](http://github.com/robbyrussell/oh-my-zsh)，github上高达18000的stars可见其受欢迎程度。通过以下命令安装：

```
wget --no-check-certificate http://install.ohmyz.sh -O - | sh
```

不过默认的oh-my-zsh长得略丑，并且也不太好用，通过配置```~/.zshrc```可以让oh-my-zsh脱胎换骨。

首先选择一个漂亮的theme，终端是使用频率最高的软件之一，UI不美观会影响工作心情，我这里比较喜欢```robbyrussell```主题，不同主题的样式可参考[这里](https://github.com/robbyrussell/oh-my-zsh/wiki/themes)。

配置你需要的插件，在```~/.zshrc```中编辑```plugins=(git ruby osx brew sublime)```将你需要的插件选上，所有支持的插件可参考[这里](https://github.com/robbyrussell/oh-my-zsh/wiki/Plugins)

[这里](http://macshuo.com/?p=676)对oh-my-zsh有一个不错的介绍。

##编辑器
这里推荐MacVim和[Sublime](http://www.sublimetext.com)。
###MacVim
首先说说MacVim，MacVim为vim提供了Mac上的原生GUI，如果你纠结为啥不用CUI的vim，那么可以看下[这里](http://www.zhihu.com/question/20020306)对MacVim的一个讨论。

安装MacVim最快的方式是执行```brew install macvim```

vim的精髓就是强大的插件支持，当然最繁琐的也是配置的插件，幸运的是github上已经有了很完善的解决方案 [janus](https://github.com/carlhuda/janus)，按照说明完成janus的安装后，你基本上拥有了一套还不错的vim配置了。

我基于janus以及自己的习惯做了一些改进，[点我](https://github.com/zjmdp/Vim-Config):

*	删除了一些不太用得到的插件，比如一堆用不到的主题（插件多了会影响vim的性能）
*	替换了一些插件（比如将自动补全插件supertab替换成[YouCompleteMe](https://github.com/Valloric/YouCompleteMe)，这货真心很强大，vim上最智能的C风格语言自动补全插件，基于clang实现）
*	增加了一些插件，例如[surround](https://github.com/tpope/vim-surround), [auto-pairs](https://github.com/jiangmiao/auto-pairs)等
*	增加一些快捷键，参考[这里](https://github.com/zjmdp/Vim-Config/blob/master/janus/vim/core/before/plugin/mappings.vim)的最后

我也不打算讨论vim能如何如何提高编辑速度，这里推荐一篇stackoverflow的文章[What is your most productive shortcut with Vim?](http://stackoverflow.com/questions/1218390/what-is-your-most-productive-shortcut-with-vim)


###Sublime
首先我个人非常推荐使用vim的编辑方式，虽然学习曲线有点不那么友好，但是一旦熟悉后，你就会明白这样的付出是值得的，何况现在的IDE基本都提供了vim的插件，包括下面要介绍的sublime。

如果你对vim不熟，并且根本不打算学习vim，那么sublime是一个不错的选择，基于我是个vim控，对sublime的使用频率也笔记比较低，因此努力找了一篇还算不错的介绍sublime的[文章](http://www.iplaysoft.com/sublimetext.html)，当然比较推荐下载sublime的vim插件。


##IDE
这里要重点推荐一下Jetbrains系列的IDE工具。

首先我是个Android开发，最早的时候在Windows下使用eclipse开发，后来用Mac上的eclipse，周围的同事基本也都是eclipse开发，因为eclipse是google官方推荐的IDE，用着也没觉得哪里不好，但也没少吐槽，最大的问题还是eclipse真的很慢，而且bug不少（比如全局搜索经常会弹出一个错误对话框）。一次偶然的机会读了一篇关于赞美Intellij IDEA的文章，google后发现大家一致推荐Intellij，罗列了各种Intellij比eclipse好的地方，于是业余也开始尝试一下Intellij，后来Google官方推出Android Studio使得Intellij IDEA成为其御用IDE，于是铁了心将项目都迁移到Intellij上，Intellij也提供了eclipse模式的快捷键配置，整个迁移过程基本没什么学习成本。在我的推荐下，周围不少同事也开始使用Intellij，特别是刚接触Android开发的同事我都会极力推荐他们直接使用Intellij。

如果你非得让我说出有哪个功能Intellij有而eclipse没有，这的确会难倒我，因为eclipse通过插件的方式也基本都能支持到intellij也有的功能，但是你会发现Intellij很多小点做得就是比eclipse体验更好，比如Intellij通过cmd + shift + g可以实现对xml文件中关键字的引用查找，也可以实现对文件的引用查找，比如查找某个layout在哪里被引用，这几乎是我平时开发中用得最多的功能之一，而据我所知eclipse是没有支持的，当然这种点还有不少，还有就是再也不会像eclipse那样无缘无故的卡半天。

当然我也没有深入去比较两者的不同，这类文章网上到处都是，对我而言最直观的感受eclipse做到的是能用，而Intellij做到的是能用并且体验好，如果读者有兴趣，我在这里也推荐一篇[文章](http://www.oschina.net/news/26929/why-intellij-is-better-than-eclipse)。

基于Intellij优秀的体验后，我开始关注Jetbrains旗下的其他IDE，比如RubyMine，WebStom，PyCharm，AppCode，所有IDE的使用体验基本和Intellij保持一个水准，在使用上也保持一致，同时Intellij通过插件也可以实现Ruby或者Web开发。当然这些IDE都是收费的，并且也不便宜，Intellij有社区免费版。大家在开发相关语言的时候可以考虑使用Jetbrains的IDE，我相信不会令你失望的。

这里罗列一些我经常使用的快捷键（eclipse风格）：

* cmd + shift + r:快速定位文件
* cmd + o: 方法索引
* cmd + shift + g: 快速查找引用，支持各种元素的引用查找
* ctrl + h: 全局搜索
* ctrl + f: 当前文件搜索
* cmd + 1的应用，比如定义一个未声明的方法或者变量，cmd + 1会帮助你创建，比如使用了未import的class，cmd + 1会帮你自动import
* cmd + n: 自动生成代码，比如自动生成override方法，自动生成构造函数

##效率工具Alfred
作为检索工具，Mac自带的Spotlight功能已经十分强大了，但Alfred提供了除了检索以外更多的功能，官方是这么描述Alfred:
>Alfred saves you time when you search for files online or on your Mac. Be more productive with hotkeys, keywords and file actions at your fingertips.

我已经将Alfred作为我一切操作的入口，快捷键呼出->输入命令->打开，整个操作一气呵成，完全不需要借助touch pad或者鼠标，大大提高了工作的效率。

我一般使用Alfred主要基于以下场景：

* 简单查找文件， cmd + num快速定位结果集中的文件，回车打开，cmd + 回车打开文件所在的文件夹。
* 复杂操作文件：find定位文件，open定位并打开文件，in在文件中进行全文检索，in命令真心好用。
* 快速启动App
* 快速执行系统功能，比如清理废纸篓（Empty Trash），强行关闭未响应的app（fq xxx）等
* workflow的使用，Alfred的精髓主要在这里，你可以自定义自己的workflow，比如我想快速翻译单词hello，我只要安装yd翻译的workflow后，在Alfred中输入yd hello就可以得到结果。推荐知乎上的一篇文章[借助 Alfred 2 的 Workflows 功能可以做哪些好玩的事情？](http://www.zhihu.com/question/20656680)

最好的熟悉Alfred方式就是打开它的设置项，在设置里基本能看到Alfred所有的功能，我就是通过这种方式熟悉的。


##设计工具
平时工作经常会遇到需要画个架构图或者流程图来表达你的设计，Mac上比较推荐[Omni Graffale](http://www.omnigroup.com/omnigraffle/)，类似于Microsoft Visio，这款软件功能十分强大，摘了一段百度百科上的介绍：
>OmniGraffle可以用来绘制图表，流程图，组织结构图以及插图，也可以用来组织头脑中思考的信息，组织头脑风暴的结果，绘制心智图，作为样式管理器，或设计网页或PDF文档的原型

OmniGraffle通过插件形式支持Stencils的扩展，Stencils是一组用于拖放的形状，目前也有大量的第三方Stencils来满足你的各种创意和设计。

Omni Group是一家只在Apple平台开发软件的公司，旗下其它几款App的体验都做得非常出色，例如：

* [OmniOutliner](http://www.omnigroup.com/omnioutliner/)是一款简单易用的用于搜集并组织信息的软件，一般我会用来做会议纪要，或者写大纲。
* [OmniPlan](http://www.omnigroup.com/omniplan/)使得项目管理变得更加容易，用来写项目周报是个不错的选择。
* [OmniFocus](http://www.omnigroup.com/omnifocus/)是一款强大的GTD软件，用来管理你的任务，这东西入门不容易，我前也花了不少时间在这上面，好不容易入门了，坚持用了几个月，最后也没坚持下来。

不过以上几款App的价格有点吓人，大家量力而行。


##文档撰写Markdown
写文档应该是大部分码农比较痛苦的事情，特别是纠结于排版的时候，因此基于纯文本书写的Markdown在程序员之间开始流行开来，很多程序员使用Markdown来书写博客，著名的博客平台WordPress和jekyll都能很好的支持Markdown，包括github的Readme也是兼容Markdown语法。Markdown使用易读易写的纯文本格式编写文档，然后转换成有效的HTML。

Markdown的宗旨是易读易写，使用Markdown书写的文档具有很高的可读性，不会看起来像是由许多Tag或者命令组成的，其设计理念来自于纯文本电子邮件格式。

Markdown精选了一些符号作为语法，你花半小时基本就能学会。本文使用Markdown完成，[这里](http://wowubuntu.com/markdown/)有Markdown详细的语法介绍，这篇[文章](http://www.yangzhiping.com/tech/r-markdown-knitr.html)写得也不错。

在Mac下比较推荐[Mou](http://25.io/mou/)，很小巧的一个免费软件，但基本具备一个Markdown编辑器该有的功能，左边书写，右边就可以看到结果，同时你也可以在Mou中配置CSS来更改最后生成的HTML的效果，比如有Github风格，Solarized风格，也可以导出PDF或者HTML。[这里](https://github.com/daktales/Mou-Themes-Collection)有大量Mou的CSS主题。

这里推荐一个比较流行的博客写作平台Git+Github+Markdown+Jekyll，有兴趣的同学可以搜索相应关键字，整个搭建过程并不复杂，这里就不赘述了。

##代码管理
如果你平时使用git，那我比较推荐[Sourcetree](http://www.sourcetreeapp.com)，它是一款功能很强大的git客户端，比如具备git项目的管理，可以同步github和bitbucket上托管的代码，可以图形化执行各种git命令，有着简单友好的diff功能（用来做简单的code review是个不错的选择），提供git-flow的支持，它几乎提供了git所有功能的图形化操作，但出于效率考虑，有些简单的git操作直接在终端里完成即可。

使用git避免不了要选择一款优秀的diff和merge工具，在以前我会推荐直接使用vim作为diff工具(配置```~/.gitconfig```即可)，不过现在Windows上很流行的[BeyondCompare](http://www.scootersoftware.com/download.php)发布Mac版本了，可惜是个收费软件，有30天的免费试用期，目前也没破解，建议入手正版，貌似也不贵。

##写在最后
虽然啰嗦得讲了一大堆，但每一个涉及到的点都没有讲细，这篇文章的主要目的是对那些刚接触Mac的同学能有一个比较好的引导，使他们能快速熟悉Mac有哪些工具可以提高开发效率，当然这里也只是一个建议，每一个工具必然都有其替代品，我推荐的只是我个人的喜好和品味，大家可根据自身情况选择。


**如果你目前还没有Mac，又误打误撞读了此文，并且你是个程序员，那么请拥抱Mac吧。**