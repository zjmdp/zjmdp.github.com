---
layout: post
title: "Android中按键消息的派发过程及源码分析"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Android中消息的整体派发过程：接收消息——消息处理前端——窗口管理系统派发消息——窗口进行消息处理

以上过程中前三步都在WmS中完成，按键消息直接发送给当前窗口，而触摸消息则根据触摸坐标位置来匹配所有窗口，并判断坐标落到哪个窗口区域中，然后把消息发送给相应的窗口。对于按键消息还会涉及到“生理长按”的检测，比如一直按住某个键，那么会产生一些列的按键消息，然而第1个和第2个消息之间往往会间隔较长的时间，这种设计是人类本身的生理特点决定的，因为从按下到弹起的过程中，如果CPU处理太快，会导致产生多次该消息，这往往不是用户所期望的，因此Android把这种消息处理延迟加入到了消息处理前端中，应用程序不需要关心第一次的延迟，只需按普通的DOWN消息处理。

下面具体分析Android中按键消息的派发流程：

每个窗口定义了一个```ViewRoot```（4.0中是```ViewRootImpl```）对象，而```ViewRoot```对象中定义了一个```inputHandler```，窗口管理系统（WmS）派发消息的过程中会调用```inputHandler的handlekey()```，该函数再调用```ViewRoot```中的```dispatchKey()```函数


{% highlight java %}
private final InputHandler mInputHandler = new InputHandler() {
    public void handleKey(KeyEvent event, InputQueue.FinishedCallback finishedCallback) {
        startInputEvent(finishedCallback);
        dispatchKey(event, true);
    }

    public void handleMotion(MotionEvent event, InputQueue.FinishedCallback finishedCallback) {
        startInputEvent(finishedCallback);
        dispatchMotion(event, true);
    }
};
{% endhighlight %}

```dispatchKey()```函数内部发送一个```DISPATCH_KEY```消息，消息的处理函数为```deliverKeyEvent()```:


{% highlight java %}
private void dispatchKey(KeyEvent event, boolean sendDone) {
    //noinspection ConstantConditions
    if (false && event.getAction() == KeyEvent.ACTION_DOWN) {
        if (event.getKeyCode() == KeyEvent.KEYCODE_CAMERA) {
            if (DBG) Log.d("keydisp", "===================================================");
            if (DBG) Log.d("keydisp", "Focused view Hierarchy is:");

            debug();

            if (DBG) Log.d("keydisp", "===================================================");
        }
    }

    Message msg = obtainMessage(DISPATCH_KEY);
    msg.obj = event;
    msg.arg1 = sendDone ? 1 : 0;

    if (LOCAL_LOGV) Log.v(
        TAG, "sending key " + event + " to " + mView);

    enqueueInputEvent(msg, event.getEventTime());
}
@Override
public void handleMessage(Message msg) {
    switch (msg.what) {
    ...
    case FINISHED_EVENT:
        handleFinishedEvent(msg.arg1, msg.arg2 != 0);
        break;
    case DISPATCH_KEY:
        deliverKeyEvent((KeyEvent)msg.obj, msg.arg1 != 0);
        break;
    case DISPATCH_POINTER:
        deliverPointerEvent((MotionEvent) msg.obj, msg.arg1 != 0);
        break;

{% endhighlight %}

```deliverKeyEvent()```函数的执行流程如下：

* 调用```mView.dispatchKeyEventPreIme()```，如果有输入法存在，那么按键消息首先会被派发到输入法窗口，如果想在输入法截获消息之前处理该消息，那么可以重载该函数。

* ```imm.dispatchKeyEvent()```将消息派发到输入法窗口

* 调用```deliverKeyEventPostIme()```继而调用到```mView.dispatchKeyEvent()```


{% highlight java %}
private void deliverKeyEvent(KeyEvent event, boolean sendDone) {
    if (ViewDebug.DEBUG_LATENCY) {
        mInputEventDeliverTimeNanos = System.nanoTime();
    }

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onKeyEvent(event, 0);
    }

    // If there is no view, then the event will not be handled.
    if (mView == null || !mAdded) {
        finishKeyEvent(event, sendDone, false);
        return;
    }

    if (LOCAL_LOGV) Log.v(TAG, "Dispatching key " + event + " to " + mView);

    // Perform predispatching before the IME.
    if (mView.dispatchKeyEventPreIme(event)) {
        finishKeyEvent(event, sendDone, true);
        return;
    }

    // Dispatch to the IME before propagating down the view hierarchy.
    // The IME will eventually call back into handleFinishedEvent.
    if (mLastWasImTarget) {
        InputMethodManager imm = InputMethodManager.peekInstance();
        if (imm != null) {
            int seq = enqueuePendingEvent(event, sendDone);
            if (DEBUG_IMF) Log.v(TAG, "Sending key event to IME: seq="
                    + seq + " event=" + event);
            imm.dispatchKeyEvent(mView.getContext(), seq, event, mInputMethodCallback);
            return;
        }
    }

    // Not dispatching to IME, continue with post IME actions.
    deliverKeyEventPostIme(event, sendDone);
}

private void deliverKeyEventPostIme(KeyEvent event, boolean sendDone) {
    ...

    // Deliver the key to the view hierarchy.
    if (mView.dispatchKeyEvent(event)) {
        finishKeyEvent(event, sendDone, true);
        return;
    }
    ...
}

{% endhighlight %}

```mView```对于应用窗口而言就是```PhoneWindow.DecorView```，否则就是普通的```ViewGroup```，我们只讨论```DecorView```中```dispatchKeyEvent```的实现：

* 处理系统快捷键

* 调用```View```中```Callback```对象的```dispatchKeyEvent()```，即调用```Activity```的```dispatchKeyEvent()```

* 如果```Activity```没有消耗该消息，则调用```PhoneWindow的OnKeyEvent()```对消息做最后的处理


{% highlight java %}
@Override
public boolean dispatchKeyEvent(KeyEvent event) {
    final int keyCode = event.getKeyCode();
    final int action = event.getAction();
    final boolean isDown = action == KeyEvent.ACTION_DOWN;

    if (isDown && (event.getRepeatCount() == 0)) {
        // First handle chording of panel key: if a panel key is held
        // but not released, try to execute a shortcut in it.
        if ((mPanelChordingKey > 0) && (mPanelChordingKey != keyCode)) {
            boolean handled = dispatchKeyShortcutEvent(event);
            if (handled) {
                return true;
            }
        }

        // If a panel is open, perform a shortcut on it without the
        // chorded panel key
        if ((mPreparedPanel != null) && mPreparedPanel.isOpen) {
            if (performPanelShortcut(mPreparedPanel, keyCode, event, 0)) {
                return true;
            }
        }
    }

    if (!isDestroyed()) {
        final Callback cb = getCallback();
        final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)
                : super.dispatchKeyEvent(event);
        if (handled) {
            return true;
        }
    }

    return isDown ? PhoneWindow.this.onKeyDown(mFeatureId, event.getKeyCode(), event)
            : PhoneWindow.this.onKeyUp(mFeatureId, event.getKeyCode(), event);
}
{% endhighlight %}

下面来具体看下```Activity```中```dispatchKeyEvent```的执行过程，首先来看源码：


{% highlight java %}
public boolean dispatchKeyEvent(KeyEvent event) {
    onUserInteraction();
    Window win = getWindow();
    if (win.superDispatchKeyEvent(event)) {
        return true;
    }
    View decor = mDecor;
    if (decor == null) decor = win.getDecorView();
    return event.dispatch(this, decor != null
            ? decor.getKeyDispatcherState() : null, this);
}

{% endhighlight %}

主要过程如下：

* 调用```onUserInteraction()```，可重载该函数在消息派发前做一些处理

* 回调```Activity```包含的```Window```对象的```superDispatchKeyEvent```，该函数继而调用```mDecor.superDispatchKveyEent```，该函数继而又调用```super.dispatchKeyEvent```，```DecorView```的父类是```FrameLayout```，而```FrameLayout```未重载```dispatchKeyEvent```，因此最终调用```ViewGroup```的```dispatchKeyEvent```

* 如果```DecorView```未消耗消息，则调用```event```的```dispatch()```函数，这里的第一个参数```receiver```是```Activity```对象


{% highlight java %}
@Override
public boolean superDispatchKeyEvent(KeyEvent event) {
    return mDecor.superDispatchKeyEvent(event);
}

public boolean superDispatchKeyEvent(KeyEvent event) {
    if (super.dispatchKeyEvent(event)) {
        return true;
    }

    // Not handled by the view hierarchy, does the action bar want it
    // to cancel out of something special?
    if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
        final int action = event.getAction();
        // Back cancels action modes first.
        if (mActionMode != null) {
            if (action == KeyEvent.ACTION_UP) {
                mActionMode.finish();
            }
            return true;
        }

        // Next collapse any expanded action views.
        if (mActionBar != null && mActionBar.hasExpandedActionView()) {
            if (action == KeyEvent.ACTION_UP) {
                mActionBar.collapseActionView();
            }
            return true;
        }
    }

    return false;
}
{% endhighlight %}

下面分析```ViewGroup```中```dispatchKeyEvent```的执行流程：如果```ViewGroup```本身拥有焦点，则调用```super.dispatchKeyEvent```把该消息派发到```ViewGroup```自身，如果其子视图拥有焦点，则调用```mFocused.dispatchKeyEvent```将消息派发给子视图，假如子视图也是```ViewGroup```，并且焦点是其子视图，则继续递归调用```ViewGroup```的```dispatchKeyEvent```


{% highlight java %}
@Override
public boolean dispatchKeyEvent(KeyEvent event) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onKeyEvent(event, 1);
    }

    if ((mPrivateFlags & (FOCUSED | HAS_BOUNDS)) == (FOCUSED | HAS_BOUNDS)) {
        if (super.dispatchKeyEvent(event)) {
            return true;
        }
    } else if (mFocused != null && (mFocused.mPrivateFlags & HAS_BOUNDS) == HAS_BOUNDS) {
        if (mFocused.dispatchKeyEvent(event)) {
            return true;
        }
    }

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 1);
    }
    return false;
}

{% endhighlight %}

在View类的```dispatchKeyEvent```中首先回调```onKey()```函数，应用程序可重载该函数以实现自定义消息处理，如果```onKey```函数未消耗该消息，则调用```event```的```dispatch```函数，在调用该函数是，第一个参数```receiver```是View对象本身


{% highlight java %}
public boolean dispatchKeyEvent(KeyEvent event) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onKeyEvent(event, 0);
    }

    // Give any attached key listener a first crack at the event.
    //noinspection SimplifiableIfStatement
    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnKeyListener.onKey(this, event.getKeyCode(), event)) {
        return true;
    }

    if (event.dispatch(this, mAttachInfo != null
            ? mAttachInfo.mKeyDispatchState : null, this)) {
        return true;
    }

    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }
    return false;
}
{% endhighlight %}

如果拥有焦点的View没有处理该按键消息，则继续调用```event.dispatch()```函数：


{% highlight java %}
/**
 * Deliver this key event to a {@link Callback} interface.  If this is
 * an ACTION_MULTIPLE event and it is not handled, then an attempt will
 * be made to deliver a single normal event.
 * 
 * @param receiver The Callback that will be given the event.
 * @param state State information retained across events.
 * @param target The target of the dispatch, for use in tracking.
 * 
 * @return The return value from the Callback method that was called.
 */
public final boolean dispatch(Callback receiver, DispatcherState state,
        Object target) {
    switch (mAction) {
        case ACTION_DOWN: {
            mFlags &= ~FLAG_START_TRACKING;
            if (DEBUG) Log.v(TAG, "Key down to " + target + " in " + state
                    + ": " + this);
            boolean res = receiver.onKeyDown(mKeyCode, this);
            if (state != null) {
                if (res && mRepeatCount == 0 && (mFlags&FLAG_START_TRACKING) != 0) {
                    if (DEBUG) Log.v(TAG, "  Start tracking!");
                    state.startTracking(this, target);
                } else if (isLongPress() && state.isTracking(this)) {
                    try {
                        if (receiver.onKeyLongPress(mKeyCode, this)) {
                            if (DEBUG) Log.v(TAG, "  Clear from long press!");
                            state.performedLongPress(this);
                            res = true;
                        }
                    } catch (AbstractMethodError e) {
                    }
                }
            }
            return res;
        }
        case ACTION_UP:
            if (DEBUG) Log.v(TAG, "Key up to " + target + " in " + state
                    + ": " + this);
            if (state != null) {
                state.handleUpEvent(this);
            }
            return receiver.onKeyUp(mKeyCode, this);
        case ACTION_MULTIPLE:
            final int count = mRepeatCount;
            final int code = mKeyCode;
            if (receiver.onKeyMultiple(code, count, this)) {
                return true;
            }
            if (code != KeyEvent.KEYCODE_UNKNOWN) {
                mAction = ACTION_DOWN;
                mRepeatCount = 0;
                boolean handled = receiver.onKeyDown(code, this);
                if (handled) {
                    mAction = ACTION_UP;
                    receiver.onKeyUp(code, this);
                }
                mAction = ACTION_MULTIPLE;
                mRepeatCount = count;
                return handled;
            }
            return false;
    }
    return false;
}
{% endhighlight %}

该函数中主要根据相应的逻辑回调了```receiver```中的```onKeyDown```, ```onKeyUp```, ```OnKeyLongPress```, ```OnKeyMultiple```函数。View中```onKeyDown```和```onKeyUp```有自己默认的处理，主要处理presse状态，长按检测，```onCick```回调。而```OnKeyLongPress```和```OnKeyMultiple```为空实现。对于Activity的```OnKeyDown```和```onKeyUp```函数主要实现按数字启动打电话程序（```onKeyDown```）以及back键的```onBackPressed```回调(```onKeyUp```)

如果按键消息在View树内部和Activity中没有被处理，就会调用到PhoneWindow的```OnKeyDown```和```OnKeyUp```函数，这是按键消息的最后处理机会，在PhoneWindow的```OnKeyDown```和```OnKeyUp```函数中主要处理了一些系统按键，例如音量键、音乐播放控制按键、照相机键、菜单键、拨号键、Search键等，具体代码就不再贴了。
