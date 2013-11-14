---
layout: post
title: "Android中layout过程详解 (结合Android 4.0.4源码)"
description: ""
category: 
tags: []
---
{% include JB/setup %}

上一篇文章Android中mesure过程详解 (结合Android 4.0.4 最新源码)介绍了View树的measure过程，相对与measure过程，本文介绍的layout过程要简单多了，正如layout的中文意思“布局”中表达的一样，layout的过程就是确定View在屏幕上显示的具体位置，在代码中就是设置其成员变量`mLeft`，`mTop`，`mRight`，`mBottom`的值，这几个值构成的矩形区域就是该View显示的位置，不过这里的具体位置都是相对与父视图的位置。

与`onMeasure`过程类似，ViewGroup在`onLayout`函数中通过调用其children的`layout`函数来设置子视图相对与父视图中的位置，具体位置由函数`layout`的参数决定，当我们继承ViewGroup时必须重载`onLayout`函数（ViewGroup中o`nLayout`是`abstract`修饰），然而`onMeasure`并不要求必须重载，因为相对与layout来说，measure过程并不是必须的，具体后面会提到。首先我们来看下View.java中函数`layout`和`onLayout`的源码：

{% highlight java %}
public void layout(int l, int t, int r, int b) {
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) {
        if (ViewDebug.TRACE_HIERARCHY) {
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.ON_LAYOUT);
        }

        onLayout(changed, l, t, r, b);
        mPrivateFlags &= ~LAYOUT_REQUIRED;

        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }
    mPrivateFlags &= ~FORCE_LAYOUT;
}

{% endhighlight %}

函数`layout`的主体过程还是很容易理解的，首先通过调用`setFrame`函数来对4个成员变量（`mLeft`，`mTop`，`mRight`，`mBottom`）赋值，然后回调`onLayout`函数，最后回调所有注册过的listener的`onLayoutChange`函数。

对于View来说，`onLayout`只是一个空实现，一般情况下我们也不需要重载该函数：

{% highlight java %}
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
{% endhighlight %}


接着我们来看下ViewGroup.java中layout的源码：

{% highlight java %}
public final void layout(int l, int t, int r, int b) {
    if (mTransition == null || !mTransition.isChangingLayout()) {
        super.layout(l, t, r, b);
    } else {
        // record the fact that we noop'd it; request layout when transition finishes
        mLayoutSuppressed = true;
    }
}
{% endhighlight %} 
    
`super.layout(l, t, r, b)`调用的即是View.java中的`layout`函数，相比之下ViewGroup增加了`LayoutTransition`的处理，`LayoutTransition`是用于处理ViewGroup增加和删除子视图的动画效果，也就是说如果当前ViewGroup未添加`LayoutTransition`动画，或者`LayoutTransition`动画此刻并未运行，那么调用`super.layout(l, t, r, b)`，继而调用到ViewGroup中的`onLayout`，否则将`mLayoutSuppressed`设置为`true`，等待动画完成时再调用`requestLayout()`。

上面`super.layout(l, t, r, b)`会调用到ViewGroup.java中`onLayout`，其源码实现如下：

{% highlight java %}
@Override
protected abstract void onLayout(boolean changed,int l, int t, int r, int b);
{% endhighlight %}  
          
和前面View.java中的`onLayout`实现相比，唯一的差别就是ViewGroup中多了关键字`abstract`的修饰，也就是说ViewGroup类只能用来被继承，无法实例化，并且其子类必须重载`onLayout`函数，而重载`onLayout`的目的就是安排其children在父视图的具体位置。重载`onLayout`通常做法就是起一个`for`循环调用每一个子视图的`layout(l, t, r, b)`函数，传入不同的参数`l`, `t`, `r`, `b`来确定每个子视图在父视图中的显示位置。

那`layout(l, t, r, b)`中的4个参数`l`, `t`, `r`, `b`如何来确定呢？联想到之前的measure过程，measure过程的最终结果就是确定了每个视图的`mMeasuredWidth`和`mMeasuredHeight`，这两个参数可以简单理解为视图期望在屏幕上显示的宽和高，而这两个参数为layout过程提供了一个很重要的依据（但不是必须的），为了说明这个过程，我们来看下LinearLayout的layout过程：

{% highlight java %}
void layoutVertical() {
    ……
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
            ……
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}
private void setChildFrame(View child, int left, int top, int width, int height) {
        child.layout(left, top, left + width, top + height);
}

{% endhighlight %} 
   
从`setChildFrame`可以看到LinearLayout中的子视图的右边界等于`left + width`，下边界等于`top + height`，也就是说在LinearLayout中其子视图显示的宽和高由measure过程来决定的，因此measure过程的意义就是为layout过程提供视图显示范围的参考值。
    
layout过程必须要依靠measure计算出来的`mMeasuredWidth`和`mMeasuredHeight`来决定视图的显示大小吗？事实并非如此，layout过程中的4个参数`l`, `t`, `r`, `b`完全可以由视图设计者任意指定，而最终视图的布局位置和大小完全由这4个参数决定，measure过程得到的`mMeasuredWidth`和`mMeasuredHeight`提供了视图大小的值，但我们完全可以不使用这两个值，可见measure过程并不是必须的。
      
说到这里就不得不提`getWidth()`、`getHeight()`和`getMeasuredWidth()`、`getMeasuredHeight()`这两对函数之间的区别，`getMeasuredWidth()`、`getMeasuredHeight()`返回的是measure过程得到的`mMeasuredWidth`和`mMeasuredHeight`的值，而`getWidth()`和`getHeight()`返回的是`mRight - mLeft`和`mBottom - mTop`的值，看View.java中的源码便一清二楚了：

{% highlight java %}
public final int getMeasuredWidth() {
    return mMeasuredWidth & MEASURED_SIZE_MASK;
}
public final int getWidth() {
	return mRight - mLeft;
}
{% endhighlight %}
    
这也解释了为什么有些情况下`getWidth()`和`getMeasuredWidth()`以及`getHeight()`和`getMeasuredHeight()`会得到不同的值。

总结：整个layout过程比较容易理解，一般情况下layout过程会参考measure过程中计算得到的`mMeasuredWidth`和`mMeasuredHeight`来安排子视图在父视图中显示的位置，但这不是必须的，measure过程得到的结果可能完全没有实际用处，特别是对于一些自定义的ViewGroup，其子视图的个数、位置和大小都是固定的，这时候我们可以忽略整个measure过程，只在layout函数中传入的4个参数来安排每个子视图的具体位置。