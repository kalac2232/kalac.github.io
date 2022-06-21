---
title: Android的TouchMode
date: 2021-03-30 14:10:56
tags: Android
categories: 
- Android
- View体系
---

## 什么是TouchMode

TouchMode就是”触摸模式“。在一般情况下，Android系统都是处于TouchMode的模式下也就是`View.isInTouchMode() == true`的状态下。大多Android开发都是开发的手机App应用，所以可能没有接触或使用过TouchMode，而在开发Android TV应用或其他没有触摸屏的应用时会接触到这个TouchMode，但一旦使用遥控器遥控或调用了`View.requestFocusFromTouch`等可以更改TouchMode的方法后，系统就会退出TouchMode，当用户点击屏幕后，就会进入触屏模式也就是TouchMode模式。

<!-- more -->

## TouchMode的影响

是否处于TouchMode会对控件的焦点（focus）产生影响。

当处于TouchMode时，直接请求`requestFocus()`是无效的，因为`requestFocus()`会最终调用到`View.requestFocusNoSearch()`

```JAVA
  private boolean requestFocusNoSearch(int direction, Rect previouslyFocusedRect) {
        
    ...
      
    // need to be focusable in touch mode if in touch mode
    if (isInTouchMode() &&
              (FOCUSABLE_IN_TOUCH_MODE != (mViewFlags & FOCUSABLE_IN_TOUCH_MODE))) {
        return false;
    }
        
    ...
            
    return true;
}
  
```

而其内部进行判断了`isInTouchMode()`,当处于触摸模式下，会直接返回false，导致请求焦点失败。

**在这个模式下，大多是View是无法获取到焦点的，如果你在开发TV应用时，发现每当开机后，焦点不能准确的选中设定的那一个，那么就需要检查下你的TouchMode与是否使用了`requestFocus()`进行请求焦点。**



像一些EditView等一些特殊的View，是可以下TouchMode下获取到焦点的，因为这些View的isFocusableInTouchMode()为true，其他的View被点击时不会获得焦点，只是会触发`onClick()`



## 如何更改TouchMode

- 退出TouchMode

直接调用`View.requestFocusFromTouch()`
```Java
		// View.requestFocusFromTouch
    public final boolean requestFocusFromTouch() {
        // Leave touch mode if we need to
        if (isInTouchMode()) {
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null) {
            		// 此处就是将TouchMode设置为false
                viewRoot.ensureTouchMode(false);
            }
        }
        return requestFocus(View.FOCUS_DOWN);
    }
    
    // ViewRootImpl.ensureTouchMode
    boolean ensureTouchMode(boolean inTouchMode) {
        if (DBG) Log.d("touchmode", "ensureTouchMode(" + inTouchMode + "), current "
                + "touch mode is " + mAttachInfo.mInTouchMode);
        if (mAttachInfo.mInTouchMode == inTouchMode) return false;

        // tell the window manager
        try {
        		// 此处真正设置TouchMode，而mWindowSession是一个全局的单例对象，
        		// 故设置后TouchMode是可以整个应用内生效的
            mWindowSession.setInTouchMode(inTouchMode);
        } catch (RemoteException e) {
            throw new RuntimeException(e);
        }

        // handle the change
        return ensureTouchModeLocally(inTouchMode);
    }
```



- 进入TouchMode

随意点击或滑动即可进入TouchMode模式

```JAVA
    // Enter touch mode on down or scroll, if it is coming from a touch screen device,
    // exit otherwise.
    final int action = event.getAction();
    if (action == MotionEvent.ACTION_DOWN || action == MotionEvent.ACTION_SCROLL) {
        ensureTouchMode(event.isFromSource(InputDevice.SOURCE_TOUCHSCREEN));
    }
```

