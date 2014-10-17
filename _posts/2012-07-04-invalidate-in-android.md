---
layout: post
title: "Android中invalidate() 函数详解"
description: ""
category: 
tags: []
---
{% include JB/setup %}

```invalidate()```函数的主要作用是请求View树进行重绘，该函数可以由应用程序调用，或者由系统函数间接调用，例如```setEnable()```, ```setSelected()```, ```setVisiblity()```都会间接调用到```invalidate()```来请求View树重绘，更新View树的显示。

注：requestLayout()和requestFocus()函数也会引起视图重绘

下面我们通过源码来了解invalidate()函数的工作原理，首先我们来看View类中invalidate()的实现过程：

```
	/**
     * Invalidate the whole view. If the view is visible,
     * {@link #onDraw(android.graphics.Canvas)} will be called at some point in
     * the future. This must be called from a UI thread. To call from a non-UI thread,
     * call {@link #postInvalidate()}.
     */
    public void invalidate() {
        invalidate(true); 
    }
```

```invalidate()```函数会转而调用```invalidate(true)```，继续往下看：

```
	/**
     * This is where the invalidate() work actually happens. A full invalidate()
     * causes the drawing cache to be invalidated, but this function can be called with
     * invalidateCache set to false to skip that invalidation step for cases that do not
     * need it (for example, a component that remains at the same dimensions with the same
     * content).
     *
     * @param invalidateCache Whether the drawing cache for this view should be invalidated as
     * well. This is usually true for a full invalidate, but may be set to false if the
     * View's contents or dimensions have not changed.
     */
    void invalidate(boolean invalidateCache) {
        if (ViewDebug.TRACE_HIERARCHY) {
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.INVALIDATE);
        }

        if (skipInvalidate()) {
            return;
        }
        if ((mPrivateFlags & (DRAWN | HAS_BOUNDS)) == (DRAWN | HAS_BOUNDS) ||
                (invalidateCache && (mPrivateFlags & DRAWING_CACHE_VALID) == DRAWING_CACHE_VALID) ||
                (mPrivateFlags & INVALIDATED) != INVALIDATED || isOpaque() != mLastIsOpaque) {
            mLastIsOpaque = isOpaque();
            mPrivateFlags &= ~DRAWN;
            mPrivateFlags |= DIRTY;
            if (invalidateCache) {
                mPrivateFlags |= INVALIDATED;
                mPrivateFlags &= ~DRAWING_CACHE_VALID;
            }
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            //noinspection PointlessBooleanExpression,ConstantConditions
            if (!HardwareRenderer.RENDER_DIRTY_REGIONS) {
                if (p != null && ai != null && ai.mHardwareAccelerated) {
                    // fast-track for GL-enabled applications; just invalidate the whole hierarchy
                    // with a null dirty rect, which tells the ViewAncestor to redraw everything
                    p.invalidateChild(this, null);
                    return;
                }
            }

            if (p != null && ai != null) {
                final Rect r = ai.mTmpInvalRect;
                r.set(0, 0, mRight - mLeft, mBottom - mTop);
                // Don't call invalidate -- we don't want to internally scroll
                // our own bounds
                p.invalidateChild(this, r);
            }
        }
    }

```

下面我们来具体进行分析```invalidate(true)```函数的执行流程：

* 首先调用```skipInvalidate()```，该函数主要判断该View是否不需要重绘，如果不许要重绘则直接返回，不需要重绘的条件是该View不可见并且未进行动画
* 接下来的if语句是来进一步判断View是否需要绘制，其中表达式```(mPrivateFlags & (DRAWN | HAS_BOUNDS)) == (DRAWN | HAS_BOUNDS)```的意思指的是如果View需要重绘并且其大小不为0，其余几个本人也未完全理解，还望高手指点～～如果需要重绘，则处理相关标志位
* 对于开启硬件加速的应用程序，则调用父视图的```invalidateChild```函数绘制整个区域，否则只绘制dirty区域（r变量所指的区域），这是一个向上回溯的过程，每一层的父View都将自己的显示区域与传入的刷新Rect做交集。



接下来看```invalidateChild()```的 实现过程：

```
    public final void invalidateChild(View child, final Rect dirty) {
        if (ViewDebug.TRACE_HIERARCHY) {
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.INVALIDATE_CHILD);
        }

        ViewParent parent = this;

        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            // If the child is drawing an animation, we want to copy this flag onto
            // ourselves and the parent to make sure the invalidate request goes
            // through
            final boolean drawAnimation = (child.mPrivateFlags & DRAW_ANIMATION) == DRAW_ANIMATION;

            if (dirty == null) {
                if (child.mLayerType != LAYER_TYPE_NONE) {
                    mPrivateFlags |= INVALIDATED;
                    mPrivateFlags &= ~DRAWING_CACHE_VALID;
                    child.mLocalDirtyRect.setEmpty();
                }
                do {
                    View view = null;
                    if (parent instanceof View) {
                        view = (View) parent;
                        if (view.mLayerType != LAYER_TYPE_NONE) {
                            view.mLocalDirtyRect.setEmpty();
                            if (view.getParent() instanceof View) {
                                final View grandParent = (View) view.getParent();
                                grandParent.mPrivateFlags |= INVALIDATED;
                                grandParent.mPrivateFlags &= ~DRAWING_CACHE_VALID;
                            }
                        }
                        if ((view.mPrivateFlags & DIRTY_MASK) != 0) {
                            // already marked dirty - we're done
                            break;
                        }
                    }

                    if (drawAnimation) {
                        if (view != null) {
                            view.mPrivateFlags |= DRAW_ANIMATION;
                        } else if (parent instanceof ViewRootImpl) {
                            ((ViewRootImpl) parent).mIsAnimating = true;
                        }
                    }

                    if (parent instanceof ViewRootImpl) {
                        ((ViewRootImpl) parent).invalidate();
                        parent = null;
                    } else if (view != null) {
                        if ((view.mPrivateFlags & DRAWN) == DRAWN ||
                                (view.mPrivateFlags & DRAWING_CACHE_VALID) == DRAWING_CACHE_VALID) {
                            view.mPrivateFlags &= ~DRAWING_CACHE_VALID;
                            view.mPrivateFlags |= DIRTY;
                            parent = view.mParent;
                        } else {
                            parent = null;
                        }
                    }
                } while (parent != null);
            } else {
                // Check whether the child that requests the invalidate is fully opaque
                final boolean isOpaque = child.isOpaque() && !drawAnimation &&
                        child.getAnimation() == null;
                // Mark the child as dirty, using the appropriate flag
                // Make sure we do not set both flags at the same time
                int opaqueFlag = isOpaque ? DIRTY_OPAQUE : DIRTY;

                if (child.mLayerType != LAYER_TYPE_NONE) {
                    mPrivateFlags |= INVALIDATED;
                    mPrivateFlags &= ~DRAWING_CACHE_VALID;
                    child.mLocalDirtyRect.union(dirty);
                }

                final int[] location = attachInfo.mInvalidateChildLocation;
                location[CHILD_LEFT_INDEX] = child.mLeft;
                location[CHILD_TOP_INDEX] = child.mTop;
                Matrix childMatrix = child.getMatrix();
                if (!childMatrix.isIdentity()) {
                    RectF boundingRect = attachInfo.mTmpTransformRect;
                    boundingRect.set(dirty);
                    //boundingRect.inset(-0.5f, -0.5f);
                    childMatrix.mapRect(boundingRect);
                    dirty.set((int) (boundingRect.left - 0.5f),
                            (int) (boundingRect.top - 0.5f),
                            (int) (boundingRect.right + 0.5f),
                            (int) (boundingRect.bottom + 0.5f));
                }

                do {
                    View view = null;
                    if (parent instanceof View) {
                        view = (View) parent;
                        if (view.mLayerType != LAYER_TYPE_NONE &&
                                view.getParent() instanceof View) {
                            final View grandParent = (View) view.getParent();
                            grandParent.mPrivateFlags |= INVALIDATED;
                            grandParent.mPrivateFlags &= ~DRAWING_CACHE_VALID;
                        }
                    }

                    if (drawAnimation) {
                        if (view != null) {
                            view.mPrivateFlags |= DRAW_ANIMATION;
                        } else if (parent instanceof ViewRootImpl) {
                            ((ViewRootImpl) parent).mIsAnimating = true;
                        }
                    }

                    // If the parent is dirty opaque or not dirty, mark it dirty with the opaque
                    // flag coming from the child that initiated the invalidate
                    if (view != null) {
                        if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                                view.getSolidColor() == 0) {
                            opaqueFlag = DIRTY;
                        }
                        if ((view.mPrivateFlags & DIRTY_MASK) != DIRTY) {
                            view.mPrivateFlags = (view.mPrivateFlags & ~DIRTY_MASK) | opaqueFlag;
                        }
                    }

                    parent = parent.invalidateChildInParent(location, dirty);
                    if (view != null) {
                        // Account for transform on current parent
                        Matrix m = view.getMatrix();
                        if (!m.isIdentity()) {
                            RectF boundingRect = attachInfo.mTmpTransformRect;
                            boundingRect.set(dirty);
                            m.mapRect(boundingRect);
                            dirty.set((int) boundingRect.left, (int) boundingRect.top,
                                    (int) (boundingRect.right + 0.5f),
                                    (int) (boundingRect.bottom + 0.5f));
                        }
                    }
                } while (parent != null);
            }
        }
    }
```


大概流程如下，我们主要关注dirty区域不是null（非硬件加速）的情况：

* 判断子视图是否是不透明的（不透明的条件是```isOpaque()```返回```true```，视图未进行动画以及```child.getAnimation() == null）```，并将判断结果保存到变量```isOpaque```中，如果不透明则将变量```opaqueFlag```设置为```DIRTY_OPAQUE```，否则设置为```DIRTY```。
* 定义```location```保存子视图的左上角坐标
* 如果子视图正在动画，那么父视图也要添加动画标志，如果父视图是```ViewGroup```，那么给```mPrivateFlags```添加```DRAW_ANIMATION```标识，如果父视图是```ViewRoot```，则给其内部变量```mIsAnimating```赋值为```true```
* 设置```dirty```标识，如果子视图是不透明的，则父视图设置为```DIRTY_OPAQUE```，否则设置为```DIRTY```
* 调用```parent.invalidateChildInparent()```，这里的```parent```有可能是```ViewGroup```，也有可能是```ViewRoot```（最后一次```while```循环），首先来看```ViewGroup```, ```ViewGroup```中该函数的主要作用是对dirty区域进行计算

以上过程的主体是一个```do{}while{}```循环，不断的将子视图的dirty区域与父视图做运算来确定最终要重绘的dirty区域，最终循环到```ViewRoot```（```ViewRoot```的```parent```为```null```）为止，并将dirty区域保存到```ViewRoot```的```mDirty```变量中

```
/**
     * Don't call or override this method. It is used for the implementation of
     * the view hierarchy.
     *
     * This implementation returns null if this ViewGroup does not have a parent,
     * if this ViewGroup is already fully invalidated or if the dirty rectangle
     * does not intersect with this ViewGroup's bounds.
     */
    public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
        if (ViewDebug.TRACE_HIERARCHY) {
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.INVALIDATE_CHILD_IN_PARENT);
        }

        if ((mPrivateFlags & DRAWN) == DRAWN ||
                (mPrivateFlags & DRAWING_CACHE_VALID) == DRAWING_CACHE_VALID) {
            if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) !=
                        FLAG_OPTIMIZE_INVALIDATE) {
                dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                        location[CHILD_TOP_INDEX] - mScrollY);

                final int left = mLeft;
                final int top = mTop;

                if ((mGroupFlags & FLAG_CLIP_CHILDREN) != FLAG_CLIP_CHILDREN ||
                        dirty.intersect(0, 0, mRight - left, mBottom - top) ||
                        (mPrivateFlags & DRAW_ANIMATION) == DRAW_ANIMATION) {
                    mPrivateFlags &= ~DRAWING_CACHE_VALID;

                    location[CHILD_LEFT_INDEX] = left;
                    location[CHILD_TOP_INDEX] = top;

                    if (mLayerType != LAYER_TYPE_NONE) {
                        mLocalDirtyRect.union(dirty);
                    }

                    return mParent;
                }
            } else {
                mPrivateFlags &= ~DRAWN & ~DRAWING_CACHE_VALID;

                location[CHILD_LEFT_INDEX] = mLeft;
                location[CHILD_TOP_INDEX] = mTop;
                if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                    dirty.set(0, 0, mRight - mLeft, mBottom - mTop);
                } else {
                    // in case the dirty rect extends outside the bounds of this container
                    dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
                }

                if (mLayerType != LAYER_TYPE_NONE) {
                    mLocalDirtyRect.union(dirty);
                }

                return mParent;
            }
        }

        return null;
    }
```

该函数首先调用```offset```将子视图的坐标位置转换为在父视图（当前视图）的显示位置，这里主要考虑scroll后导致子视图在父视图中的显示区域会发生变化，接着调用```union```函数求得当前视图与子视图的交集,求得的交集必定是小于```dirty```的范围，因为子视图的```dirty```区域有可能超出其父视图（当前视图）的范围，最后返回当前视图的父视图。

再来看```ViewRoot```中```invalidateChildInparent```的执行过程：

```
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
        invalidateChild(null, dirty);
        return null;
    }
```

该函数仅仅调用了```ViewRoot```的```invalidateChild```，下面继续看```invalidateChild```的源码：

```
public void invalidateChild(View child, Rect dirty) {
        checkThread();
        if (DEBUG_DRAW) Log.v(TAG, "Invalidate child: " + dirty);
        if (dirty == null) {
            // Fast invalidation for GL-enabled applications; GL must redraw everything
            invalidate();
            return;
        }
        if (mCurScrollY != 0 || mTranslator != null) {
            mTempRect.set(dirty);
            dirty = mTempRect;
            if (mCurScrollY != 0) {
               dirty.offset(0, -mCurScrollY);
            }
            if (mTranslator != null) {
                mTranslator.translateRectInAppWindowToScreen(dirty);
            }
            if (mAttachInfo.mScalingRequired) {
                dirty.inset(-1, -1);
            }
        }
        if (!mDirty.isEmpty() && !mDirty.contains(dirty)) {
            mAttachInfo.mSetIgnoreDirtyState = true;
            mAttachInfo.mIgnoreDirtyState = true;
        }
        mDirty.union(dirty);
        if (!mWillDrawSoon) {
            scheduleTraversals();
        }
    }
```

具体分析如下：

* 判断此次调用是否在UI线程中进行
* 将dirty的坐标位置转换为```ViewRoot```的屏幕显示区域
* 更新```mDirty```变量，并调用```scheduleTraversals```发起重绘请求

至此一次```invalidate()```就结束了。

###总结
```invalidate```主要给需要重绘的视图添加```DIRTY```标记，并通过和父视图的矩形运算求得真正需要绘制的区域，并保存在```ViewRoot```中的```mDirty```变量中，最后调用```scheduleTraversals```发起重绘请求，```scheduleTraversals```会发送一个异步消息，最终调用```performTraversals()```执行重绘，```performTraversals()```的具体过程以后再分析。


*以上所有代码基于Android 4.0.4，并结合《Android内核剖析》分析总结而成，源码中涉及到的部分细节本人也未完全理解，还望高手指点~~*