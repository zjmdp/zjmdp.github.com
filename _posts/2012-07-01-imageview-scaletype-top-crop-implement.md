---
layout: post
title: "ImageView ScaleType 的扩展之Top Crop 的实现"
description: ""
category: 
tags: []
---
{% include JB/setup %}

`ImageView`中`ScaleType`属性可用来设置image的填充方式，主要通过以下两种途径：1、XML文件中设置`android:scaleType`属性。2、代码中使用函数`setScaleType(ScaleType scaleType)`来设定。目前内置的填充方式有如下8种：

* `CENTER`  按图片的原来size居中显示，当图片长/宽超过View的长/宽，则截取图片的居中部分显示
* `CENTER_CROP` 按比例扩大图片的size居中显示，使得图片长(宽)等于或大于View的长(宽)
* `CENTER_INSIDE` 将图片的内容完整居中显示，通过按比例缩小或原来的size使得图片长/宽等于或小于View的长/宽
* `FIT_CENTER` 把图片按比例扩大/缩小到View的宽度，居中显示
* `FIT_END` 把图片按比例扩大/缩小到View的宽度，显示在View的下部分位置
* `FIT_START` 图片按比例扩大/缩小到View的宽度，显示在View的上部分位置
* `FIT_XY` 把图片不按比例 扩大/缩小到View的大小显示
* `MATRIX`  用矩阵来绘制

首先要选择设定图片填充方式的时机，不管如何，我们在必须在draw之前设定填充方式，因此我们可以考虑在layout的时候，通过查看layout源码，注意到`setFrame`函数，`setFrame`主要是设定view的尺寸和位置，并返回view是否changed，因此可在setFrame中设定填充方式，layout代码片段如下：

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
}
{% endhighlight %}

然而对于人像为主的图片来说，默认的几种方式都无法很好的满足需求，因为人像的头部往往被切割了，我们可以忍受人像下半部分被切割，但是无法忍受头部被切割，因此我们考虑第9种填充方式：`TOP_CROP`，`TOP_CROP`按如下方式填充图片：按比例扩大图片的size，仅仅横向居中，使得图片长(宽)等于或大于View的长(宽)。对于`CENTER_CROP`，其不仅横向居中，而且垂直也居中。 

具体实现方式：重载`ImageView`，并重写`setFrame`方法来对`ImageView`中的`drawable`进行变换，代码如下：  
{% highlight java %}
public class TopCropImageView extends ImageView {

    public TopCropImageView(Context context, AttributeSet attrs) {
            super(context, attrs);
            setScaleType(ScaleType.MATRIX);
    }
    
    public TopCropImageView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        setScaleType(ScaleType.MATRIX);
    }

    public TopCropImageView(Context context) {
        super(context);
        setScaleType(ScaleType.MATRIX);
    }
   
    @Override
    protected boolean setFrame(int l, int t, int r, int b)
    {
        if (getDrawable() == null) {
            return super.setFrame(l, t, r, b);
        }
        Matrix matrix = getImageMatrix();
        float scaleWidth = getWidth()/(float)getDrawable().getIntrinsicWidth();
        float scaleHeight = getHeight()/(float)getDrawable().getIntrinsicHeight();
        float scaleFactor = (scaleWidth > scaleHeight) ? scaleWidth : scaleHeight;
        matrix.setScale(scaleFactor, scaleFactor, 0, 0);
        if (scaleFactor == scaleHeight) {
            float tanslateX = ((getDrawable().getIntrinsicWidth() * scaleFactor) - getWidth()) / 2;
            matrix.postTranslate(-tanslateX, 0);
        }
        setImageMatrix(matrix);
        return super.setFrame(l, t, r, b);
    }
}
{% endhighlight %}

代码中首先计算`ImageView`尺寸和`Drawable`尺寸的比值，用变量`scaleWidth`和`scaleHeight`保存，并用`scaleFactor`取两者较大值，因为我们要满足图片长(宽)等于或大于View的长(宽)，然后调用matrix.setScale进行图片的缩放操作，接下来的if (`scaleFactor == scaleHeight`) 的含义是`Drawable`放大后的`width`大于`ImageView`的`width`，这意味着我们需要进行水平居中平移，`if`分支中的代码即实现了平移操作，最后通过`setImageMatrix(matrix)`将`matrix`应用到image中来实现`TOP_CROP`的填充方式。下面两张图显示了`CENTER_CROP`和`TOP_CROP`的不同效果：

<img src="http://my.csdn.net/uploads/201206/18/1339997012_1879.png" alt="center_crop" width="300" />
<img src="http://my.csdn.net/uploads/201206/18/1339996905_7544.png" alt="top_crop" width="300" />