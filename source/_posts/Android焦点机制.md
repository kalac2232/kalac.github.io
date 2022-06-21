---
title: Android焦点机制
tags: Android
categories: 
- Android
- View体系
cover: false
date: 2021-03-31 17:20:30
---

<article class="message message-immersive is-primary">
<div class="message-body">
<i class="fas fa-lightbulb mr-2"></i>
本篇所分析的源码为Android 28，可能与其他版本有所出入
</div>
</article>

## 焦点存储与设置
### 焦点的标志位
在`View`中，自身的许多状态都是使用`mPrivateFlags`来记录，其中`FOCUSABLE_MASK`位为记录`View`的焦点状态

``` java View.java
    /**
     * Mask for use with setFlags indicating bits used for focus.
     */
    private static final int FOCUSABLE_MASK = 0x00000011;
```
`View`的焦点焦点状态共有3种`NOT_FOCUSABLE`、`FOCUSABLE`、`FOCUSABLE_AUTO`。
```java View.java
    /**
     * This view does not want keystrokes.
     */
    public static final int NOT_FOCUSABLE = 0x00000000;

    /**
     * This view wants keystrokes.
     */
    public static final int FOCUSABLE = 0x00000001;

    /**
     * This view determines focusability automatically. This is the default.
     */
    public static final int FOCUSABLE_AUTO = 0x00000010;
```

其中`FOCUSABLE_AUTO`为默认状态，当未在xml未配置`android:focusable`属性时，系统会将焦点状态设置为`FOCUSABLE_AUTO`
<!-- more -->
```java View.java
    public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        // ...
        // 在构造方法中设置默认值为FOCUSABLE_AUTO
        viewFlagValues |= FOCUSABLE_AUTO;
        viewFlagMasks |= FOCUSABLE_AUTO;
        // ...
        // 获取xml中配置的focusable的值
        case com.android.internal.R.styleable.View_focusable:
            viewFlagValues = (viewFlagValues & ~FOCUSABLE_MASK) | getFocusableAttribute(a);
            if ((viewFlagValues & FOCUSABLE_AUTO) == 0) { // 如果新flag不是FOCUSABLE_AUTO，则更改viewFlagMasks中的标志位，准备在setFlags中更改状态
                viewFlagMasks |= FOCUSABLE_MASK;
            }
            break;
        // ...
        
        if (viewFlagMasks != 0) { // View状态发生变化进行更新flag
            setFlags(viewFlagValues, viewFlagMasks);
        }

    }

    private int getFocusableAttribute(TypedArray attributes) {
        TypedValue val = new TypedValue();
        if (attributes.getValue(com.android.internal.R.styleable.View_focusable, val)) {
            if (val.type == TypedValue.TYPE_INT_BOOLEAN) {
                return (val.data == 0 ? NOT_FOCUSABLE : FOCUSABLE);
            } else {
                return val.data;
            }
        } else {
            return FOCUSABLE_AUTO; // 如果没配置则为FOCUSABLE_AUTO
        }
    }
```
### 焦点状态的获取
`View`中获取焦点状态即是判断flag中的对应标志位
```JAVA View.java
    public boolean hasFocus() {
        return (mPrivateFlags & PFLAG_FOCUSED) != 0;
    }
```

而在`ViewGroup`中，存在一个`mFocused`属性，是当前获取了焦点的子View。但这个`mFocused`并不是直接就是视图树中当前有焦点的那个`View`，而是当前这个`ViewGroup`中的直接子`View`。当需要获取当前焦点`View`时，会递归的调用直到找到具体的`View`。在获取获取焦点状态时，不仅会判断自身是否有焦点，还有判断自己的子`View`是否也具有焦点。

```JAVA ViewGroup.java
    public boolean hasFocus() {
        return (mPrivateFlags & PFLAG_FOCUSED) != 0 || mFocused != null;
    }
```

### 焦点状态的设置

无论是调用了`View.setFocusable(Boolean)`还是新版本新增的api`View.setFocusable(int)`，最后都是要通过setFlags()方法进行标志位的更改
```java View.java
    void setFlags(int flags, int mask) {
    int old = mViewFlags;
        mViewFlags = (mViewFlags & ~mask) | (flags & mask);

        int changed = mViewFlags ^ old;
        if (changed == 0) {
            return;
        }
        int privateFlags = mPrivateFlags;
        boolean shouldNotifyFocusableAvailable = false;

        // If focusable is auto, update the FOCUSABLE bit.
        int focusableChangedByAuto = 0;
        if (((mViewFlags & FOCUSABLE_AUTO) != 0)
                && (changed & (FOCUSABLE_MASK | CLICKABLE)) != 0) {
            // Heuristic only takes into account whether view is clickable.
            final int newFocus;
            if ((mViewFlags & CLICKABLE) != 0) {
                newFocus = FOCUSABLE;
            } else {
                newFocus = NOT_FOCUSABLE;
            }
            mViewFlags = (mViewFlags & ~FOCUSABLE) | newFocus;
            focusableChangedByAuto = (old & FOCUSABLE) ^ (newFocus & FOCUSABLE);
            changed = (changed & ~FOCUSABLE) | focusableChangedByAuto;
        }

        /* Check if the FOCUSABLE bit has changed */
        if (((changed & FOCUSABLE) != 0) && ((privateFlags & PFLAG_HAS_BOUNDS) != 0)) {
            if (((old & FOCUSABLE) == FOCUSABLE)
                    && ((privateFlags & PFLAG_FOCUSED) != 0)) {
                /* Give up focus if we are no longer focusable */
                clearFocus();
                if (mParent instanceof ViewGroup) {
                    ((ViewGroup) mParent).clearFocusedInCluster();
                }
            } else if (((old & FOCUSABLE) == NOT_FOCUSABLE)
                    && ((privateFlags & PFLAG_FOCUSED) == 0)) {
                /*
                 * Tell the view system that we are now available to take focus
                 * if no one else already has it.
                 */
                if (mParent != null) {
                    ViewRootImpl viewRootImpl = getViewRootImpl();
                    if (!sAutoFocusableOffUIThreadWontNotifyParents
                            || focusableChangedByAuto == 0
                            || viewRootImpl == null
                            || viewRootImpl.mThread == Thread.currentThread()) {
                        shouldNotifyFocusableAvailable = canTakeFocus();
                    }
                }
            }
        }
    }
```

## 焦点事件的产生
### 第一次由系统产生焦点

当系统第一次调用到`ViewRootImpl.performTraversals()`方法时，会创建第一个焦点事件。
```JAVA ViewRootImpl.java
    if (mFirst) {
        // 当系统版本小于28时，sAlwaysAssignFocus为true
        if (sAlwaysAssignFocus || !isInTouchMode()) {
            // handle first focus request
            // mView 为 DecorView
            if (mView != null) {
                if (!mView.hasFocus()) {
                    // 将焦点产生在DecorView上
                    mView.restoreDefaultFocus();
                }
            }
        } else {
            // Some views (like ScrollView) won't hand focus to descendants that aren't within
            // their viewport. Before layout, there's a good change these views are size 0
            // which means no children can get focus. After layout, this view now has size, but
            // is not guaranteed to hand-off focus to a focusable child (specifically, the edge-
            // case where the child has a size prior to layout and thus won't trigger
            // focusableViewAvailable).
            // 确保ScrollView等滑动View的子View会正确的获取到焦点
            View focused = mView.findFocus();
            if (focused instanceof ViewGroup 
                    && ((ViewGroup) focused).getDescendantFocusability() 
                    == ViewGroup.FOCUS_AFTER_DESCENDANTS) {
                focused.restoreDefaultFocus();
            }
        }
    }
```


当系统第一次绘制时，会根据`sAlwaysAssignFocus`和当前是否为触摸模式进行判断是否产生焦点。
`sAlwaysAssignFocus`会在`ViewRootImpl`初始化时根据当前版本是否小于Android9被赋值。而其中的`mView`为添加到Window的`DecorView`，所以系统第一次产生的焦点默认会产生在根View上，由根View调用`restoreDefaultFocus`在自己的子view中寻找合适和View，一般为左上角或右下角，由系统页面方向决定。而`restoreDefaultFocus`本质上就是调用了`requestFocus`方法。

```java View.java
    public boolean restoreDefaultFocus() {
        return requestFocus(View.FOCUS_DOWN);
    }
```
> 在Android9前，打开一个包含EditView的Activity时EditView会自动获取焦点，导致软键盘弹出。可通过在父View上配置focusableInTouchMode = true来消除此现象。

> 应用一般默认处于触摸模式下，当接收到遥控器或其他设备产生的KeyEvent，会切换到非触摸模式下



### 按键产生焦点

当第一次系统处于触摸模式或其他原因导致未产生第一个焦点，那么在按下任意一个按键将产生第一个焦点。

按下键后会触发`ViewRootImpl.processKeyEvent`。该方法负责了按键事件的分发。

```JAVA ViewRootImpl.java
        private int processKeyEvent(QueuedInputEvent q) {
            final KeyEvent event = (KeyEvent)q.mEvent;

            if (mUnhandledKeyManager.preViewDispatch(event)) {
                return FINISH_HANDLED;
            }

            // Deliver the key to the view hierarchy.
            // 在视图树上依次分发，判断焦点View是否自行处理KeyEvent，
            // 如果自定义key事件并返回true，后续不再执行
            if (mView.dispatchKeyEvent(event)) {
                return FINISH_HANDLED;
            }

            //一些保护措施
            //在View层次结构不消费事件，判断窗口是否有输入事件或者已经停止和销毁
            if (shouldDropInputEvent(q)) {
                return FINISH_NOT_HANDLED;
            }

            // This dispatch is for windows that don't have a Window.Callback. Otherwise,
            // the Window.Callback usually will have already called this (see
            // DecorView.superDispatchKeyEvent) leaving this call a no-op.
            if (mUnhandledKeyManager.dispatch(mView, event)) {
                return FINISH_HANDLED;
            }

            int groupNavigationDirection = 0;
            // 判断是否是前进后退组合键，决定焦点的移动方向
            if (event.getAction() == KeyEvent.ACTION_DOWN
                    && event.getKeyCode() == KeyEvent.KEYCODE_TAB) {
                if (KeyEvent.metaStateHasModifiers(event.getMetaState(), KeyEvent.META_META_ON)) {
                    groupNavigationDirection = View.FOCUS_FORWARD;
                } else if (KeyEvent.metaStateHasModifiers(event.getMetaState(),
                        KeyEvent.META_META_ON | KeyEvent.META_SHIFT_ON)) {
                    groupNavigationDirection = View.FOCUS_BACKWARD;
                }
            }

            // If a modifier is held, try to interpret the key as a shortcut.
            if (event.getAction() == KeyEvent.ACTION_DOWN
                    && !KeyEvent.metaStateHasNoModifiers(event.getMetaState())
                    && event.getRepeatCount() == 0
                    && !KeyEvent.isModifierKey(event.getKeyCode())
                    && groupNavigationDirection == 0) {
                // 对快捷键进行处理，目前子View只有TextView进行了实现。
                if (mView.dispatchKeyShortcutEvent(event)) {
                    return FINISH_HANDLED;
                }
                if (shouldDropInputEvent(q)) {
                    return FINISH_NOT_HANDLED;
                }
            }

            // Apply the fallback event policy.
            if (mFallbackEventHandler.dispatchKeyEvent(event)) {
                return FINISH_HANDLED;
            }
            if (shouldDropInputEvent(q)) {
                return FINISH_NOT_HANDLED;
            }

            // Handle automatic focus changes.
            if (event.getAction() == KeyEvent.ACTION_DOWN) {
                if (groupNavigationDirection != 0) {
                    // 执行组合键策略
                    if (performKeyboardGroupNavigation(groupNavigationDirection)) {
                        return FINISH_HANDLED;
                    }
                } else {
                    // 执行上下左右键策略
                    if (performFocusNavigation(event)) {
                        return FINISH_HANDLED;
                    }
                }
            }
            return FORWARD;
        }
```

如果用户没有自定义KeyEvent处理事件，则会进行系统的焦点处理机制，在之前没有焦点时，会产生第一个焦点，而如果已经有了焦点，则会根据按键的类型进行焦点的移动。

会根据是按了前进后退键还是上下左右键调用`performKeyboardGroupNavigation(direction)`或`performFocusNavigation(keyEvent)`,而其内部通过调用`mView.restoreDefaultFocus()`进行产生焦点。

```JAVA ViewRootImpl.java
    // 按下前进后退键
    private boolean performKeyboardGroupNavigation(int direction) {
        final View focused = mView.findFocus();
        if (focused == null && mView.restoreDefaultFocus()) {
            return true;
        }
      
      // ..省略移动焦点代码
    }
```

```JAVA ViewRootImpl.java
// 按下上下左右键
private boolean performFocusNavigation(KeyEvent event) {
    // .. 省略判断方向代码
    if (direction != 0) {
        View focused = mView.findFocus();
        if (focused != null) {
            // .. 省略移动代码
        } else {
            if (mView.restoreDefaultFocus()) {
                return true;
            }
        }
    }
    return false;
}
```
> `performKeyboardGroupNavigation`与`performFocusNavigation`完整代码分析见焦点的移动

### requestFocus() 请求焦点

在View中，`requestFocus`最终是调用了`requestFocusNoSearch`
```java View.java
    private boolean requestFocusNoSearch(int direction, Rect previouslyFocusedRect) {
        // need to be focusable
        // 检测view可见性、layout空间大小等
        if (!canTakeFocus()) {
            return false;
        }

        // need to be focusable in touch mode if in touch mode
        // 判断当前是否为触摸模式与是否配置了focusableInTouchMode = true
        if (isInTouchMode() &&
            (FOCUSABLE_IN_TOUCH_MODE != (mViewFlags & FOCUSABLE_IN_TOUCH_MODE))) {
               return false;
        }

        // need to not have any parents blocking us
        // 判断是否有父View拦截焦点事件
        if (hasAncestorThatBlocksDescendantFocus()) {
            return false;
        }

        if (!isLayoutValid()) {
            mPrivateFlags |= PFLAG_WANTS_FOCUS;
        } else {
            clearParentsWantFocus();
        }
        // 处理焦点事件
        handleFocusGainInternal(direction, previouslyFocusedRect);
        return true;
    }
```
`requestFocusNoSearch`先通过一系列方法检测该View是否可以获得焦点，最后使用`handleFocusGainInternal`处理焦点事件。
> 这里判断了focusableInTouchMode的值，所以当为触摸模式的时候，通过focusableInTouchMode=true也可以使得View获取到焦点，所以大多数文章中使用通过在RecycleView或EditView等自动获取焦点的View的父View上配置这个属性来禁止RecycleView、EditView等自动获取焦点，因为这样可以使父View消耗掉这个焦点事件。

在ViewGroup中，requestFocus有所不同。
```java ViewGroup.java
    public boolean requestFocus(int direction, Rect previouslyFocusedRect) {
        // 获取配置的descendantFocusability
        int descendantFocusability = getDescendantFocusability();

        boolean result;
        switch (descendantFocusability) {
            
            case FOCUS_BLOCK_DESCENDANTS:
                // 直接请求自己的requestFocus
                result = super.requestFocus(direction, previouslyFocusedRect);
                break;
            case FOCUS_BEFORE_DESCENDANTS: {
                // 1. 先自行进行请求焦点
                final boolean took = super.requestFocus(direction, previouslyFocusedRect);
                // 2. 如果自己不获取焦点，那么就通过onRequestFocusInDescendants方法分发到子View上
                result = took ? took : onRequestFocusInDescendants(direction,
                        previouslyFocusedRect);
                break;
            }
            case FOCUS_AFTER_DESCENDANTS: {
                // 1. 先通过onRequestFocusInDescendants尝试分发给子View
                final boolean took = onRequestFocusInDescendants(direction, previouslyFocusedRect);
                // 2. 如果子view不获取焦点，那么自己请求焦点
                result = took ? took : super.requestFocus(direction, previouslyFocusedRect);
                break;
            }
            default:
                throw new IllegalStateException("descendant focusability must be "
                        + "one of FOCUS_BEFORE_DESCENDANTS, FOCUS_AFTER_DESCENDANTS, FOCUS_BLOCK_DESCENDANTS "
                        + "but is " + descendantFocusability);
        }
        if (result && !isLayoutValid() && ((mPrivateFlags & PFLAG_WANTS_FOCUS) == 0)) {
            mPrivateFlags |= PFLAG_WANTS_FOCUS;
        }
        return result;
    }
```
在ViewGroup中，判断了自身配置的`descendantFocusability`的值，其各自所代表的含义为：

- **FOCUS_BLOCK_DESCENDANTS** ：
仅尝试将焦点分发给当前 ViewGroup

- **FOCUS_BEFORE_DESCENDANTS** ：
先尝试将焦点分发给当前 ViewGroup，然后才尝试将焦点分发给ChildView。

- **FOCUS_AFTER_DESCENDANTS** ：
先尝试将焦点分发给ChildView，然后才尝试将焦点分发给当前 ViewGroup。


```java View.java
    void handleFocusGainInternal(@FocusRealDirection int direction, Rect previouslyFocusedRect) {
        
        if ((mPrivateFlags & PFLAG_FOCUSED) == 0) {
            // 更改焦点标志位
            mPrivateFlags |= PFLAG_FOCUSED;
            // 获取当前已经聚焦的View，用于在监听回调处理相应的逻辑
            View oldFocus = (mAttachInfo != null) ? getRootView().findFocus() : null;

            if (mParent != null) {
                // 通知 mParent 焦点变化事件
                mParent.requestChildFocus(this, this);
                updateFocusedInCluster(oldFocus, direction);
            }

            if (mAttachInfo != null) {
                // 通知 ViewTreeObserver 焦点变化事件
                mAttachInfo.mTreeObserver.dispatchOnGlobalFocusChange(oldFocus, this);
            }
            // 通知当前View焦点变化事件，回调焦点变化监听
            onFocusChanged(true, direction, previouslyFocusedRect);
            // 刷新当前View的状态
            refreshDrawableState();
        }
    }
```
在`handleFocusGainInternal`方法中，将本View标志位已改为了聚焦的状态，，并通知了一系列监听的变化及状态的改变，但在整个视图树中ViewGroup中存储的`mFocues`对象的值还并没有修改，该值将会在`requestChildFocus`方法中进行修改。

```java View.java
// ViewGroup.java
    public void requestChildFocus(View child, View focused) {
        // 拦截了ChildView的获焦事件，此时焦点不需要继续向上一层级透传
        if (getDescendantFocusability() == FOCUS_BLOCK_DESCENDANTS) {
            return;
        }

        // Unfocus us, if necessary
        // 主要作用为如果当前ViewGroup为焦点View，那么清除当前 ViewGroup 的焦点
        super.unFocus(focused);

        // We had a previous notion of who had focus. Clear it.
        if (mFocused != child) {
            if (mFocused != null) {
                // 清除当前存储的View的焦点状态
                mFocused.unFocus(focused);
            }
            // 更改存储的聚焦的view
            mFocused = child;
        }
        if (mParent != null) {
            // 这里一层一层向上通知焦点View的改变
            mParent.requestChildFocus(this, focused);
        }
    }
```

unFocus
```java ViewGroup.java
    void unFocus(View focused) {
        // 如果自身是聚焦View，那么清除自身
        if (mFocused == null) {
            super.unFocus(focused);
        } else {
            mFocused.unFocus(focused);
            mFocused = null;
        }
    }
```

``` java View.java
    void unFocus(View focused) {
        clearFocusInternal(focused, false, false);
    }
    
    void clearFocusInternal(View focused, boolean propagate, boolean refocus) {
        if ((mPrivateFlags & PFLAG_FOCUSED) != 0) {
            // 清除标志位
            mPrivateFlags &= ~PFLAG_FOCUSED;
            clearParentsWantFocus();
            
            // 如果propagate为true，则会向上层层调用清除mFocus的值
            // 调用unFocus时为false，调用clearFocus时为true
            if (propagate && mParent != null) {
                mParent.clearChildFocus(this);
            }
            // 通知监听
            onFocusChanged(false, 0, null);
            refreshDrawableState();

            if (propagate && (!refocus || !rootViewRequestFocus())) {
                notifyGlobalFocusCleared(this);
            }
        }
    }
```
其实此处传入的focused并没有实际作用。
## 焦点的移动

焦点的移动总的来说可以分为这几步。
1. 确定移动方向
2. 寻找到当前具有焦点View
3. 寻找到需要接受焦点的View
4. 切换View的状态

###  组合键焦点移动

在前进后退组合键触发的焦点移动中，移动方向已经在`processKeyEvent`中确定。
```JAVA ViewRootImpl.java
    private boolean performKeyboardGroupNavigation(int direction) {
        // 1. 通过findFocus方法找到当前具有焦点的View，如果当前没有View获取焦点则返回null
        //    产生一个默认的焦点
        final View focused = mView.findFocus();
        if (focused == null && mView.restoreDefaultFocus()) {
            return true;
        }
        // 2. 寻找需要接受焦点的View
        View cluster = focused == null ? keyboardNavigationClusterSearch(null, direction)
                    : focused.keyboardNavigationClusterSearch(null, direction);

        // Since requestFocus only takes "real" focus directions (and therefore also
        // restoreFocusInCluster), convert forward/backward focus into FOCUS_DOWN.
        int realDirection = direction;
        if (direction == View.FOCUS_FORWARD || direction == View.FOCUS_BACKWARD) {
            realDirection = View.FOCUS_DOWN;
        }
        // 3. 通过调用restoreFocusNotInCluster restoreFocusInCluster切换焦点
        if (cluster != null && cluster.isRootNamespace()) {
            // the default cluster. Try to find a non-clustered view to focus.
            if (cluster.restoreFocusNotInCluster()) {
                playSoundEffect(SoundEffectConstants.getContantForFocusDirection(direction));
                return true;
            }
            // otherwise skip to next actual cluster
            cluster = keyboardNavigationClusterSearch(null, direction);
        }

        if (cluster != null && cluster.restoreFocusInCluster(realDirection)) {
            // 播发按键音
            playSoundEffect(SoundEffectConstants.getContantForFocusDirection(direction));
            return true;
        }

        return false;
    }
```
首先调用了`View.findFocus`方法开始查找焦点View。

```JAVA View.java
    public View findFocus() {
        return (mPrivateFlags & PFLAG_FOCUSED) != 0 ? this : null;
    }
```
``` java ViewGroup.java
    public View findFocus() {
        if (isFocused()) {
            return this;
        }
        // mFocused是具有焦点的View
        if (mFocused != null) {
            return mFocused.findFocus();
        }
        return null;
    }
```
在View中，和`hasFocus`方法一样，是通过判断标识位来判断是否具有焦点。而在ViewGroup，如果自身具有焦点直接返回自己，如果是子View中具有焦点则通过递归调用`mFocused.findFocus`直到找到焦点View。

当查询到焦点View后通过`View.keyboardNavigationClusterSearch`或`ViewRootImpl.keyboardNavigationClusterSearch`进行查找目标焦点View。而View与ViewRootImpl都是会将搜寻工作代理给`FocusFinder`类，但在View中，会进行判断是否为根View，也就是是否为DecorView。

```java View.java  
    public View keyboardNavigationClusterSearch(View currentCluster,
            @FocusDirection int direction) {
        if (isKeyboardNavigationCluster()) {
            currentCluster = this;
        }
        if (isRootNamespace()) {
            // Root namespace means we should consider ourselves the top of the
            // tree for group searching; otherwise we could be group searching
            // into other tabs.  see LocalActivityManager and TabHost for more info.
            // 如果是根View，则代理给FocusFinder负责进行搜寻工作
            return FocusFinder.getInstance().findNextKeyboardNavigationCluster(
                    this, currentCluster, direction);
        } else if (mParent != null) {
            // 一层一层向上找根View
            return mParent.keyboardNavigationClusterSearch(currentCluster, direction);
        }
        return null;
    }

//ViewRootImpl
    public View keyboardNavigationClusterSearch(View currentCluster,
            @FocusDirection int direction) {
        checkThread();
        // 代理到FocusFinder
        return FocusFinder.getInstance().findNextKeyboardNavigationCluster(
                mView, currentCluster, direction);
    }
```



在确定目标View后，通过调用`View.restoreFocusNotInCluster`与` View.restoreFocusInCluster`切换焦点,内部均是通过调用了`requestFocus()`实现了焦点的切换

```java View.java
    public boolean restoreFocusInCluster(@FocusRealDirection int direction) {
        // Prioritize focusableByDefault over algorithmic focus selection.
        if (restoreDefaultFocus()) {
            return true;
        }
        return requestFocus(direction);
    }
    public boolean restoreDefaultFocus() {
        return requestFocus(View.FOCUS_DOWN);
    }

    
    public boolean restoreFocusNotInCluster() {
        return requestFocus(View.FOCUS_DOWN);
    }
```



###  通过方向键焦点移动

```java ViewRootImpl.java
    private boolean performFocusNavigation(KeyEvent event) {
        int direction = 0;
        // 1. 根据按键确定方向
        switch (event.getKeyCode()) {
            case KeyEvent.KEYCODE_DPAD_LEFT:
                if (event.hasNoModifiers()) {
                    direction = View.FOCUS_LEFT;
                }
                break;
            case KeyEvent.KEYCODE_DPAD_RIGHT:
                if (event.hasNoModifiers()) {
                    direction = View.FOCUS_RIGHT;
                }
                break;
            case KeyEvent.KEYCODE_DPAD_UP:
                if (event.hasNoModifiers()) {
                    direction = View.FOCUS_UP;
                }
                break;
            case KeyEvent.KEYCODE_DPAD_DOWN:
                if (event.hasNoModifiers()) {
                    direction = View.FOCUS_DOWN;
                }
                break;
            case KeyEvent.KEYCODE_TAB:
                if (event.hasNoModifiers()) {
                    direction = View.FOCUS_FORWARD;
                } else if (event.hasModifiers(KeyEvent.META_SHIFT_ON)) {
                    direction = View.FOCUS_BACKWARD;
                }
                break;
        }
        if (direction != 0) {
            // 2. 找到当前聚焦的View
            View focused = mView.findFocus();
            if (focused != null) {
                // 3. 寻找需要获取焦点的View
                View v = focused.focusSearch(direction);
                if (v != null && v != focused) {
                    // do the math the get the interesting rect
                    // of previous focused into the coord system of
                    // newly focused view
                    focused.getFocusedRect(mTempRect);
                    if (mView instanceof ViewGroup) {
                        ((ViewGroup) mView).offsetDescendantRectToMyCoords(
                                    focused, mTempRect);
                        ((ViewGroup) mView).offsetRectIntoDescendantCoords(
                                    v, mTempRect);
                    }
                    // 4. 进行焦点状态的切换
                    if (v.requestFocus(direction, mTempRect)) {
                        playSoundEffect(SoundEffectConstants
                                    .getContantForFocusDirection(direction));
                        return true;
                    }
                }

                // Give the focused view a last chance to handle the dpad key.
                if (mView.dispatchUnhandledMove(focused, direction)) {
                    return true;
                }
            } else {
                // 当没有焦点时产生默认的焦点
                if (mView.restoreDefaultFocus()) {
                    return true;
                }
            }
        }
        return false;
    }
```

首先会通过用户实际按下的按键判断当前需要向哪个方向进行搜寻，然后也是通过调用`mView.findFocus()`找到当前聚焦的View，与组合键操作不同的是，方向键查找下一个是调用的是`View.focusSearch(direction)`，不过，这个方法内部也是将具体的搜寻工作代理到了`FocusFinder`这个类上。

```java View.java
    public View focusSearch(@FocusRealDirection int direction) {
        if (mParent != null) {
            return mParent.focusSearch(this, direction);
        } else {
            return null;
        }
    }
```
``` java ViewGroup.java
    public View focusSearch(View focused, int direction) {
        if (isRootNamespace()) {
            // root namespace means we should consider ourselves the top of the
            // tree for focus searching; otherwise we could be focus searching
            // into other tabs.  see LocalActivityManager and TabHost for more info.
            return FocusFinder.getInstance().findNextFocus(this, focused, direction);
        } else if (mParent != null) {
            return mParent.focusSearch(focused, direction);
        }
        return null;
    }
```

#### FocusFinder查找的具体实现

但现在先暂时跳过探究`FocusFinder`如何找到目标View，先来关注下`requestFocus`这个方法中是如何进行焦点切换工作的。**1问题：为什么当前ViewGourp内找不到适合的焦点View话会找到更高级的ViewGroup里的View（可能在添加备选列表时将所有父View符合的View添加到了列表中）** 



## 清除焦点
### 清除焦点的方法
当手动调用clearFocus方法或使获焦的 View 被移除(隐藏不可见)的时候，会进行焦点的清除工作。
``` java View.java 
    void setFlags(int flags, int mask) {
    // ...
    /* Check if the GONE bit has changed */
        if ((changed & GONE) != 0) {

            if (((mViewFlags & VISIBILITY_MASK) == GONE)) {
                if (hasFocus()) {
                  // 进行清除焦点
                    clearFocus();
                    if (mParent instanceof ViewGroup) {
                        ((ViewGroup) mParent).clearFocusedInCluster();
                    }
                }
                clearAccessibilityFocus();
                // ...
            }
        }
    }
```
``` java ViewGourp.java
protected void removeDetachedView(View child, boolean animate) {
    // ...

        if (child == mFocused) {
            child.clearFocus();
        }
    // ...
    }
```

```java View.java 
    public void clearFocus() {

        clearFocusInternal(null, true, true);
    }

```
### 焦点的重分配（矫正焦点）

当某个View从Gone转变为Visible，或在一个聚焦的父View中的子View可获焦了，焦点需要重新分配到正确的位置上。

```java View.java
    void setFlags(int flags, int mask) {
        final int newVisibility = flags & VISIBILITY_MASK;
        if (newVisibility == VISIBLE) {
            if ((changed & VISIBILITY_MASK) != 0) {
                /*
                 * If this view is becoming visible, invalidate it in case it changed while
                 * it was not visible. Marking it drawn ensures that the invalidation will
                 * go through.
                 */
                mPrivateFlags |= PFLAG_DRAWN;
                invalidate(true);

                needGlobalAttributesUpdate(true);

                // a view becoming visible is worth notifying the parent about in case nothing has
                // focus. Even if this specific view isn't focusable, it may contain something that
                // is, so let the root view try to give this focus if nothing else does.
                // 判断是否可需要通知矫正焦点1
                shouldNotifyFocusableAvailable = hasSize();
            }
        }

        if ((changed & ENABLED_MASK) != 0) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED) {
                // a view becoming enabled should notify the parent as long as the view is also
                // visible and the parent wasn't already notified by becoming visible during this
                // setFlags invocation.
                // 判断是否可需要通知矫正焦点2
                shouldNotifyFocusableAvailable = canTakeFocus();
            } else {
                if (isFocused()) clearFocus();
            }
        }

        if (shouldNotifyFocusableAvailable && mParent != null) {
            // 通知父View需要进行焦点重分配，最终会被ViewRootImpl接收
            mParent.focusableViewAvailable(this);
        }
    }
```

```java ViewGroup.java
    public void focusableViewAvailable(View v) {
        if (mParent != null
                // shortcut: don't report a new focusable view if we block our descendants from
                // getting focus or if we're not visible
                && (getDescendantFocusability() != FOCUS_BLOCK_DESCENDANTS)
                && ((mViewFlags & VISIBILITY_MASK) == VISIBLE)
                && (isFocusableInTouchMode() || !shouldBlockFocusForTouchscreen())
                // shortcut: don't report a new focusable view if we already are focused
                // (and we don't prefer our descendants)
                //
                // note: knowing that mFocused is non-null is not a good enough reason
                // to break the traversal since in that case we'd actually have to find
                // the focused view and make sure it wasn't FOCUS_AFTER_DESCENDANTS and
                // an ancestor of v; this will get checked for at ViewAncestor
                && !(isFocused() && getDescendantFocusability() != FOCUS_AFTER_DESCENDANTS)) {
            // 一层一层向上传递
            mParent.focusableViewAvailable(v);
        }
    }

```
在一层一层向上传递后，最终会传递到`ViewRootImpl`中进行具体焦点的分配工作。
```java ViewRootImpl.java
    public void focusableViewAvailable(View v) {
        checkThread();
        if (mView != null) {
            if (!mView.hasFocus()) {
                // 如果视图树中没有一个View有焦点，则直接在传递来的View上获取焦点
                if (sAlwaysAssignFocus || !mAttachInfo.mInTouchMode) {
                    v.requestFocus();
                }
            } else {
                // the one case where will transfer focus away from the current one
                // is if the current view is a view group that prefers to give focus
                // to its children first AND the view is a descendant of it.
                // 如果当前有焦点View
                View focused = mView.findFocus();
                // 检查是否需要将焦点转移到这个View上
                if (focused instanceof ViewGroup) {
                    ViewGroup group = (ViewGroup) focused;
                    // 如果当前焦点View是传递来的View的直接父View，
                    // 并且焦点优先子获取View，则将焦点传递给v
                    if (group.getDescendantFocusability() == ViewGroup.FOCUS_AFTER_DESCENDANTS
                            && isViewDescendantOf(v, focused)) {
                        v.requestFocus();
                    }
                }
            }
        }
    }
```

所以转移规则为：

- 如果当前视图树中不存在焦点，则直接尝试将焦点分发给这个可获焦的 View

- 如果存在焦点，则检查焦点View为直系父View，并配置了`FOCUS_AFTER_DESCENDANTS`,是否需要将焦点转移到这个可获焦的 View

## 参考文章

- [1] [Android焦点分发过程解析 - 薛定谔的程序猫](https://juejin.cn/post/6871541574019317768)

- [2] [Android焦点搜索过程解析 - 薛定谔的程序猫](https://juejin.cn/post/6871542178506604551)

