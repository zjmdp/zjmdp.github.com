---
layout: post
title: "Android中draw过程分析 (结合Android 4.0.4源码)"
description: ""
category: 
tags: []
---
{% include JB/setup %}

经过对View树的measure和layout过程后，接下来将结合前两步得到的结果对View树进行绘制，之前以为measure过程是measure、layout和draw三部曲中最复杂的一步，在仔细分析draw过程后才发现自己之前的论断有失准确性。不过从整体来看，draw过程的逻辑是比较清晰的，和measure和layout过程十分相似，而本文将从整体来介绍draw的整个流程，至于其中的细节可能会在另外一篇文章中介绍。

和measure和layout一样，draw过程也是在ViewRoot的`performTraversals()`的内部发起的，其调用顺序在`measure()`和`layout()`之后，同样的，`performTraversals()`发起的draw过程最终会调用到`mView`的`draw()`函数，这里的`mView`对于Actiity来说就是`PhoneWindow.DecorView`。

首先来看下与draw过程相关的函数，之所以先列出相关函数，目的是让大家对这些函数有个印象，以免在介绍具体细节的时候茫然：

* `ViewRootImpl.draw()`，仅在`ViewRootImpl.performTraversals()`的内部调用
* `DecorView.draw()`, 上一步中的`ViewRootImpl.draw()`会调用到该函数，`DecorView.draw()`继承自Framelayout，DecorView和FrameLayout，以及FrameLayout的父类ViewGroup都未重载`draw()`,而ViewGroup的父类是View类，因此`DecorView.draw()`调用的是`View.draw()`的默认实现
* `View.onDraw()`，绘制View本身，自定义View往往会重载该函数来绘制View本身的内容
* `View.dispatchDraw()`, View中的`dispatchDraw`默认是空实现，ViewGroup重载了该函数，内部会循环调用`View.drawChild()`来发起对子视图的绘制，应用程序不应该重载ViewGroup，因为该函数的默认实现代表了View的绘制流程
* `ViewGroup.drawChild()`，该函数只在ViewGroup中实现，原因就是只有ViewGroup才需要绘制child，`drawChild`内部又会调用`View.draw()`函数来完成子视图的绘制（有可能直接调用`dispatchDraw`）

下面笔者将从源码来展现draw的整体过程，当然源码中只展现关键部分，细节部分将在下一篇文章中具体介绍，首先来看`performTraversals()`，因为draw过程最先是从这里发起的：

{% highlight java %}
private void performTraversals() {
    final View host = mView;
    ...
    host.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    ...
    draw(fullRedrawNeeded);
}
{% endhighlight %}

注意到measure和layout过程直接调用的是`mView`的measure和layout函数，而draw调用的是`ViewRootImpl`的内部`draw(boolean fullRedrawNeeded)`函数，再由`draw(boolean fullRedrawNeeded)`函数来调用`mView.draw()`函数，`draw(boolean fullRedrawNeeded)`包含draw过程的一些前期处理，通过下面的代码可以看出调用关系：

{% highlight java %}
private void draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
        if (surface == null || !surface.isValid()) {
            return;
        }
		...
        try {
	     	canvas.translate(0, -yoff);
	     	if (mTranslator != null) {
	     		mTranslator.translateCanvas(canvas);
	     	}
	     	canvas.setScreenDensity(scalingRequired ? DisplayMetrics.DENSITY_DEVICE : 0);
	     	mAttachInfo.mSetIgnoreDirtyState = false;
	     	mView.draw(canvas);
       } finally {
	    	if (!mAttachInfo.mSetIgnoreDirtyState) {
	    	// Only clear the flag if it was not set during the mView.draw() call
	    	mAttachInfo.mIgnoreDirtyState = false;
         	}
       }
       ...
}
{% endhighlight %}

下面将转到`mView.draw()`,之前提到`mView.draw()`调用的就是View.java的默认实现，View类中的draw函数体现了View绘制的核心流程，因此我们下面重点来看下View.java中draw的调用流程：
{% highlight java %}
public void draw(Canvas canvas) {
	...
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
		...
        background.draw(canvas);
		...
        // skip step 2 & 5 if possible (common case)
		...
        // Step 2, save the canvas' layers
		...
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }
		...
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            canvas.drawRect(left, top, right, top + length, p);
        }
		...
        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);
    }
{% endhighlight %}

官方源码中对View的绘制流程给了非常详细的注释，笔者将整个绘制过程的关键步骤抽取出来方便了解整个过程，通过阅读上面的代码可以知道整个绘制过程包括View的背景绘制，View本身内容的绘制，子视图的绘制（如果包含子视图），渐变框的绘制以及滚动条的绘制。
我们重点要关注的是View本身内容的绘制和子视图的绘制，即`onDraw()`和`dispatchDraw()`函数。
对于View.java和ViewGroup.java，`onDraw()`默认都是空实现，因为具体View本身长什么样子是由View的设计者来决定的，默认不显示任何东西。
View.java中`dispatchDraw()`默认为空实现，因为其不包含子视图，而ViewGroup重载了`dispatchDraw()`来对其子视图进行绘制，通常应用程序不应该对`dispatchDraw()`进行重载，其默认实现体现了View系统绘制的流程。那么，接下来我们继续分析下ViewGroup中`dispatchDraw()`的具体流程：

{% highlight java %}
@Override
    protected void dispatchDraw(Canvas canvas) {
       ...

        if ((flags & FLAG_USE_CHILD_DRAWING_ORDER) == 0) {
            for (int i = 0; i < count; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                    more |= drawChild(canvas, child, drawingTime);
                }
            }
        } else {
            for (int i = 0; i < count; i++) {
                final View child = children[getChildDrawingOrder(count, i)];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                    more |= drawChild(canvas, child, drawingTime);
                }
            }
        }
      	...
    }

{% endhighlight %}

`dispatchDraw()`的核心代码就是通过for循环调用`drawChild()`对ViewGroup的每个子视图进行绘制，上述代码中如果`FLAG_USE_CHILD_DRAWING_ORDER`为`true`，则子视图的绘制顺序通过`getChildDrawingOrder`来决定，默认的绘制顺序即是子视图加入ViewGroup的顺序，而我们可以重载`getChildDrawingOrder`函数来更改默认的绘制顺序，这会影响到子视图之间的重叠关系。
`drawChild()`的核心过程就是为子视图分配合适的cavas剪切区，剪切区的大小正是由layout过程决定的，而剪切区的位置取决于滚动值以及子视图当前的动画。设置完剪切区后就会调用子视图的`draw()`函数进行具体的绘制，如果子视图的包含`SKIP_DRAW`标识，那么仅调用`dispatchDraw()`，即跳过子视图本身的绘制，但要绘制视图可能包含的字视图。
完成了`dispatchDraw()`过程后，View系统会调用`onDrawScrollBars()`来绘制滚动条，滚动条绘制的细节这里就不展开介绍了。

总结：以上文字着重于对draw整体流程的介绍，通过对关键函数以及它们之间的调用关系来展示draw的内部过程，从整体上把握整个过程将更有利于后面对细节的理解和分析。