---
layout: post
title: "Android中mesure过程详解 (结合Android 4.0.4源码)"
description: ""
category: 
tags: []
---
{% include JB/setup %}

如何遍历并绘制View树？之前的文章Android中`invalidate()`函数详解(结合Android 4.0.4源码)中提到`invalidate()`最后会发起一个View树遍历的请求，并通过执行`performTraersal()`来响应该请求，`performTraersal()`正是对View树进行遍历和绘制的核心函数，内部的主体逻辑是判断是否需要重新测量视图大小（measure），是否需要重新布局（layout），是否重新需要绘制（draw）。measure过程是遍历的前提，只有measure后才能进行布局（layout）和绘制（draw），因为在layout的过程中需要用到measure过程中计算得到的每个View的测量大小，而draw过程需要layout确定每个view的位置才能进行绘制。下面我们主要来探讨一下measure的主要过程，相对与layout和draw，measure过程理解起来比较困难。

我们在编写layout的xml文件时会碰到`layout_width`和`layout_height`两个属性，对于这两个属性我们有三种选择：赋值成具体的数值，`match_parent`或者`wrap_content`，而measure过程就是用来处理`match_parent`或者`wrap_content`，假如layout中规定所有View的`layout_width`和`layout_height`必须赋值成具体的数值，那么measure其实是没有必要的，但是google在设计Android的时候考虑加入`match_parent`或者`wrap_content`肯定是有原因的，它们会使得布局更加灵活。

首先我们来看几个关键的函数和参数：

1. `public final void measue(int widthMeasureSpec, int heightMeasureSpec)`

2. `protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec)`

3. `protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec)`

4. `protected void measureChild(View child, int parentWidthMeasureSpec, int parentHeightMeasureSpec)`

5. `protected void measureChildWithMargins(View child,int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed)`

接着我们来看View类中`measure`和`onMeasure`函数的源码

{% highlight java %}
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    if ((mPrivateFlags & FORCE_LAYOUT) == FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {

        // first clears the measured dimension flag
        mPrivateFlags &= ~MEASURED_DIMENSION_SET;

        if (ViewDebug.TRACE_HIERARCHY) {
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.ON_MEASURE);
        }

        // measure ourselves, this should set the measured dimension flag back
        onMeasure(widthMeasureSpec, heightMeasureSpec);

        // flag not set, setMeasuredDimension() was not invoked, we raise
        // an exception to warn the developer
        if ((mPrivateFlags & MEASURED_DIMENSION_SET) != MEASURED_DIMENSION_SET) {
            throw new IllegalStateException("onMeasure() did not set the"
                    + " measured dimension by calling"
                    + " setMeasuredDimension()");
        }

        mPrivateFlags |= LAYOUT_REQUIRED;
    }

    mOldWidthMeasureSpec = widthMeasureSpec;
    mOldHeightMeasureSpec = heightMeasureSpec;
}     
{% endhighlight %}

由于函数原型中有`final`字段，那么`measure`根本没打算被子类继承，也就是说`measure`的过程是固定的，而`measure`中调用了`onMeasure`函数，因此真正有变数的是`onMeasure`函数，`onMeasure`的默认实现很简单，源码如下：

{% highlight java %}
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
							getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
{% endhighlight %}

`onMeasure`默认的实现仅仅调用了`setMeasuredDimension`，`setMeasuredDimension`函数是一个很关键的函数，它对View的成员变量`mMeasuredWidth`和`mMeasuredHeight`变量赋值，而measure的主要目的就是对View树中的每个View的`mMeasuredWidth`和`mMeasuredHeight`进行赋值，一旦这两个变量被赋值，则意味着该View的测量工作结束。

{% highlight java %}
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
	mMeasuredWidth = measuredWidth;
	mMeasuredHeight = measuredHeight;

	mPrivateFlags |= MEASURED_DIMENSION_SET;
}
{% endhighlight %}

对于非ViewGroup的View而言，通过调用上面默认的`measure`——>`onMeasure`，即可完成View的测量，当然你也可以重载`onMeasure`，并调用`setMeasuredDimension`来设置任意大小的布局，但一般不这么做，因为这种做法太“专政”，至于为何“专政”，读完本文就会明白。

对于ViewGroup的子类而言，往往会重载`onMeasure`函数负责其children的measure工作，重载时不要忘记调用`setMeasuredDimension`来设置自身的`mMeasuredWidth`和`mMeasuredHeight`。如果我们在layout的时候不需要依赖子视图的大小，那么不重载`onMeasure`也可以，但是必须重载`onLayout`来安排子视图的位置，这在下一篇博客中会介绍。  

再来看下`measue(int widthMeasureSpec, int heightMeasureSpec)`中的两个参数， 这两个参数分别是父视图提供的测量规格，当父视图调用子视图的`measure`函数对子视图进行测量时，会传入这两个参数，通过这两个参数以及子视图本身的`LayoutParams`来共同决定子视图的测量规格，在ViewGroup的`measureChildWithMargins`函数中体现了这个过程，稍后会介绍。

`MeasureSpec`参数的值为`int`型，分为高32位和低16为，高32位保存的是`specMode`，低16位表示`specSize`，`specMode`分三种：

1. `MeasureSpec.UNSPECIFIED`,父视图不对子视图施加任何限制，子视图可以得到任意想要的大小；

2. `MeasureSpec.EXACTLY`，父视图希望子视图的大小是`specSize`中指定的大小；

3. `MeasureSpec.AT_MOST`，子视图的大小最多是`specSize`中的大小。

以上施加的限制只是父视图“希望”子视图的大小按`MeasureSpec`中描述的那样，但是子视图的具体大小取决于多方面的。

ViewGroup中定义了`measureChildren`, `measureChild`,  `measureChildWithMargins`来对子视图进行测量，`measureChildren`内部只是循环调用`measureChild`，`measureChild`和`measureChildWithMargins`的区别就是是否把margin和padding也作为子视图的大小，我们主要分析`measureChildWithMargins`的执行过程：

{% highlight java %}
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
{% endhighlight %}

总的来看该函数就是对父视图提供的`measureSpec`参数进行了调整（结合自身的`LayoutParams`参数），然后再来调用`child.measure()`函数，具体通过函数`getChildMeasureSpec`来进行参数调整，过程如下：

{% highlight java %}
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = 0;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = 0;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
{% endhighlight %}

`getChildMeasureSpec`的总体思路就是通过其父视图提供的`MeasureSpec`参数得到`specMode和specSize`，并根据计算出来的`specMode`以及子视图的`childDimension`（`layout_width`和`layout_height`中定义的）来计算自身的`measureSpec`，如果其本身包含子视图，则计算出来的`measureSpec`将作为调用其子视图measure函数的参数，同时也作为自身调用`setMeasuredDimension`的参数，如果其不包含子视图则默认情况下最终会调用`onMeasure`的默认实现，并最终调用到`setMeasuredDimension`，而该函数的参数正是这里计算出来的。

总结：从上面的描述看出，决定权最大的就是View的设计者，因为设计者可以通过调用`setMeasuredDimension`决定视图的最终大小，例如调用`setMeasuredDimension(100， 100)`将视图的`mMeasuredWidth`和`mMeasuredHeight`设置为`100`，`100`，那么父视图提供的大小以及程序员在xml中设置的`layout_width`和`layout_height`将完全不起作用，当然良好的设计一般会根据子视图的`measureSpec`来设置`mMeasuredWidth`和`mMeasuredHeight`的大小，已尊重程序员的意图。

