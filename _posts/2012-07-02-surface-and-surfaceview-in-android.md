---
layout: post
title: "Android中Surface和SurfaceView的一些理解和总结"
description: ""
category: 
tags: []
---
{% include JB/setup %}

###什么是Surface？
简单地说Surface对应了一块屏幕缓冲区，每个window对应一个Surface，任何View都是画在Surface上的，传统的view共享一块屏幕缓冲区，所有的绘制必须在UI线程中进行
###什么是SurfaceView？
说SurfaceView是一个View也许不够严谨，然而从定义中```public class SurfaceView extends View {...}```显示SurfaceView确实是派生自View，但是SurfaceView却有着自己的Surface，继续看SurfaceView的源码：

```
if (mWindow == null) {
      mWindow = new MyWindow(this);
      mLayout.type = mWindowType;
      mLayout.gravity = Gravity.LEFT|Gravity.TOP;
      mSession.addWithoutInputChannel(mWindow, mWindow.mSeq, mLayout,
      mVisible ? VISIBLE : GONE, mContentInsets);
}
```

很明显，每个SurfaceView创建的时候都会创建一个```MyWindow```，```new MyWindow(this)```中的this正是SurfaceView自身，因此将SurfaceView和window绑定在一起，而前面提到过每个window对应一个Surface，所以SurfaceView也就内嵌了一个自己的Surface，可以认为SurfaceView是来控制Surface的位置和尺寸。

大家都知道，传统View及其派生类的更新只能在UI线程，然而UI线程还同时处理其他交互逻辑，这就无法保证view更新的速度和帧率了，而SurfaceView可以用独立的线程来进行绘制，因此可以提供更高的帧率，例如游戏，摄像头取景等场景就比较适合用SurfaceView来实现。

###什么是SurfaceHolder.Callback？
```SurfaceHolder.Callback```主要是当底层的Surface被创建、销毁或者改变时提供回调通知，由于绘制必须在surface被创建后才能进行，因此```SurfaceHolder.Callback```中的```surfaceCreated```和```surfaceDestroyed```就成了绘图处理代码的边界。SurfaceHolder，可以把它当成Surface的容器和控制器，用来操纵Surface。处理它的Canvas上画的效果和动画，控制表面，大小，像素等。

###为什么普通view只能在UI线程刷新？

UI线程是最重要的线程，它既不能被阻塞，也不是线程安全的，如果你了解多线程操作的话，对于线程安全、同步锁等等这样的词语应该不陌生，UI线程负责绘制界面和分发窗口事件，任务是非常之重，通常多线程处理时为了保证访问资源的正确性，通常对于某些操作都会加上同步锁，这样会显然会降低效率，而且还会涉及到线程的等待与线程上下文切换，为了提高效率，UI线程不在使用这些繁琐的多线程机制，为了保证对UI操作的正确性，只允许在UI线程中操作UI。在非UI线程中可通过```post```或者```runOnUiThread```来刷新view

###其它的一些总结：

SurfaceView是视图(View)的子类，这个视图里内嵌了一个专门用于绘制的Surface。你可以控制这个Surface的格式和尺寸。SurfaceView控制这个Surface的绘制位置。

Surface是纵深排序(Z-ordered)的，这表明它总在自己所在窗口的后面。Surfaceview提供了一个可见区域，只有在这个可见区域内的Surface部分内容才可见，可见区域外的部分不可见。Surface的排版显示受到视图层级关系的影响，它的兄弟视图结点会在顶端显示。这意味者 Surface的内容会被它的兄弟视图遮挡，这一特性可以用来放置遮盖物(overlays)(例如，文本和按钮等控件)。注意，如果Surface上面 有透明控件，那么它的每次变化都会引起框架重新计算它和顶层控件的透明效果，这会影响性能。

你可以通过SurfaceHolder接口访问这个Surface，```getHolder()```方法可以得到这个接口。

SurfaceView变得可见时，Surface被创建；SurfaceView隐藏前，Surface被销毁。这样能节省资源。如果你要查看Surface被创建和销毁的时机，可以重载```surfaceCreated(SurfaceHolder)```和```surfaceDestroyed(SurfaceHolder)```。

SurfaceView的核心在于提供了两个线程：UI线程和渲染线程。这里应注意：   

* 所有```SurfaceView```和```SurfaceHolder.Callback```的方法都应该在UI线程里调用，一般来说就是应用程序主线程。渲染线程所要访问的各种变量应该作同步处理
* 由于Surface可能被销毁，它只在```SurfaceHolder.Callback.surfaceCreated()```和```SurfaceHolder.Callback.surfaceDestroyed()```之间有效，所以要确保渲染线程访问的是合法有效的Surface。