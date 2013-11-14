---
layout: post
title: "minSdkVersion和targetSdkVersion的意义"
description: ""
category: 
tags: []
---
{% include JB/setup %}

做了几个项目，一直没有在意`minSdkVersion`字段和`targetSdkVersion`字段的意义，今天简单查阅了一下Android官方文档的介绍，总算是搞明白，也简单做个总结。

1. 说白了`minSdkVersion`就是为了指定你这个app最低可以跑在哪个Api Level的机器上，比如设定`android:minSdkVersion="9"`（Android 2.3），而你的机器系统又是Android 2.2，那么对不起，这个app你装不了。那么设定minSdkVersion的意义何在呢？一般情况下是因为你的代码中用了`>=minSdkVersion`的接口，低版本的系统不认识，当然app也跑不了了。
2. `targeSdkVersion`的意思是说我可以确保在这个Api Level的机器上app可以正常运行，请不要以兼容模式运行，而大于等于`targeSdkVersion`的机器为了保持向前兼容性（比`targeSdkVersion`指定的Api Level更新的操作系统），则允许以兼容模式运行，也就是说我们要保证`minSdkVersion`和`targeSdkVersion`之间的版本都经过严格测试。最典型的一个例子就是holo主题，如果你指定的 `targeSdkVersion<11`，那么holo就不会作为默认的主题，如果`targeSdkVersion>=11`，那么holo主题就会默认应用到app中，当然前提是你机器os的Api Level >= 11。
3. 总的来说，`minSdkVersion`保证了向后兼容（老os的机器），`targeSdkVersion`保证了向前兼容（新os的机器）。
