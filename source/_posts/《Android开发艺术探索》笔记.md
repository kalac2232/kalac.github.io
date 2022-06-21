---
title: 《Android开发艺术探索》笔记
date: 2021-03-26 16:03:08
tags: Android
categories: Android

---

# 第一章 Activty的生命周期和启动模式
## Activity的生命周期全面分析
### 典型情况下的生命周期分析

在正常的情况下，Activity会经历如下生命周期。
1. `onCreate`: **表示Activity正在被创建**,是生命周期的第一个方法
2. `onRestart`: **表示Activity正在被重新启动**。一般由用户从ActivityA启动ActivityB后，重新返回ActivityA触发。
3. `onStart`: **表示Activity已经可见，但无法与用户交互**。*（没有获取到焦点）*
4. `onResume`: **表示activity获得了焦点，用户可以进行操作**。
5. `onPause`: **Activity正在停止，失去了焦点，不能进行操作**。此时可以做一些存储数据、停止动画等工作，但**不能太耗时，会影响新Activity的显示** *（新Activity的onResume会在老Activity的`onPause`方法后执行）*
6. `onStop`: **表示Activity即将停止，对用户来说已经不可见。** 可以在此时做稍重的回收工作，也不能太耗时。**一般的操作尽量在onStop中执行,不要放到onPause中**
7. `onDestory` : **表示activity即将被销毁**,是最后一个回调。可以做回收工作和最终资源的释放。

<!-- more -->

![image](https://tuchuang-beijing.oss-cn-beijing.aliyuncs.com/2509931-1409c13efedd9b37.PNG)


### 异常情况下的生命周期分析
1. 资源相关的系统配置发生改变导致Activity被杀死并重新创建

    在不对Activity进行配置时，旋转手机等操作会使系统配置发生改变，会引发Activity销毁并重新创建。
    
    - 销毁
    
        当Activity开始销毁时，其onPause、onStop、onDestory均会被调用，并在**onStop之前**调用`onSaveInstanceState`来保存当前Activity的状态 *（与onPause没有明确的调用前后顺序）*。
        
    - 保存
    
        将需要保存的数据保存到`onSaveInstanceState`传入的`bundle`中。**注意**：`onSaveInstanceState`有俩个同名函数，通常只需重写`onSaveInstanceState(Bundle outState)`的即可，`onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState) {`只有在activity中配置`persistableMode`标签才会调用，此标签用于数据的持久化。
    - 恢复
      
        当Activity重建后，会将保存的bundle数据在`onCreate(Bundle b)`的参数中传入,并调用`onRestoreInstanceState`。如果要在onCreate中进行状态恢复需要进行对bundle进行判空，**推荐在`onRestoreInstanceState`中进行状态的恢复**,因为该方法一旦调用，其参数一定不为空。
        
        `onRestoreInstanceState`的调用时机在`onStart`之后，与`onResume`的调用时机不能保证。

    - 调用输出顺序
    
```
        //Activity正常创建
        I/Main2Activity: onCreate: 
        I/Main2Activity: onStart: 
        I/Main2Activity: onResume: 
        //屏幕旋转导致Activity开始销毁
        I/Main2Activity: onPause: 
        //保存数据
        I/Main2Activity: onSaveInstanceState:
        I/Main2Activity: onStop: 
        I/Main2Activity: onDestroy: 
        //开始重建
        I/Main2Activity: onCreate: 
        I/Main2Activity: onStart: 
        //恢复数据
        I/Main2Activity: onRestoreInstanceState:
        I/Main2Activity: onResume: 
```

	系统只有在Activity即将被销毁并有机会重新显示的情况下才会调用`onSaveInstanceState`,正常退出的Activity是不会触发调用

2. 资源内存不足导致低优先级的Activity被杀死

    Activity优先级排序
    
    1. 前台Activity -- 正在和用户交互的Activity,优先级最高
    2. 可见但非前台Activity -- 比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法与用户直接交互。
    3. 后台Activity -- 已经被暂停的Activity，比如执行了onStop,优先级最低。
    
    当系统系统内存不足时，系统会按照上述优先级去杀死Activity所在的进程，并在后续通过`onSaveInstanceState`和`onRestoreInstanceState`来存储和恢复数据。

##  Activity的启动模式

###  Activity的LaunchMode

1. standard:

    标准模式，也是默认模式。**每次启动一个Activity都会重新创建一个新的实例，不管这个实例是否已经存在。**，一个任务栈中可以有多个实例，每个实例也可以属于不同的任务栈。在这种模式下，谁启动这个Activity,那么这个Activity就运行在启动它的那个Activity所在的栈中。

    *当用ApplicationContext去启动standard模式的Activity会报错，因为非Activity类型的Context没有任务栈，此时，需要为Activity指定`FLAG_ACTIVITY_NEW_TASK`标记位新建一个任务栈来存放这个Activity。此时，该Activity实际上使用singTask模式启动的*
2. singleTop:

    栈顶复用模式。**如果新Acitivy已经位于任务栈的栈顶，那么此Activiy不会被重新创建，但是会被调用该Activity的`onNewIntent`方法**。
3. singleTask:

    栈内复用模式，在这种模式下，入栈时，在目标栈中如果存在此Activity实例，那么不会重新创建实例。**当已经存在的实例不在栈顶时，会先将栈中实例之上的Activity全部出栈，移出任务栈，使singleTask模式的Activity位于栈顶**。当栈中存在的实例在栈顶时，直接入栈，并调用`onNewIntent`

4. 单实例模式。该模式具有singleTask模式的所有特性外，还有这个模式的Acitivity**只能单独的位于一个任务栈中**。启动时，会新建一个任务栈，该任务栈只会有该Acitivity一个实例。

### 任务栈回退

*在存在多个任务栈的情况下，由前台任务栈（第一个有焦点的Activity所在的任务栈）启动后台任务栈中的Activity的时候，整个后台任务栈会被切换为前台任务栈，整个栈中的Activity都会前移，之前的前台任务栈集体后移。*

**当启动ActivityD时的任务栈**
    
![任务栈回退1](https://tuchuang-beijing.oss-cn-beijing.aliyuncs.com/%E4%BB%BB%E5%8A%A1%E6%A0%88%E5%9B%9E%E9%80%801.png)
**当启动ActivityC时的任务栈**
    
![任务栈回退2](https://tuchuang-beijing.oss-cn-beijing.aliyuncs.com/%E4%BB%BB%E5%8A%A1%E6%A0%88%E5%9B%9E%E9%80%802.png)

ActivityD被出栈的原因：++当启动ActivityC时，C处于后台任务栈中，首先将整个任务栈提到前台任务栈，变为 D -> C -> B -> A,又因为C的启动模式为SingleTask，当不为栈顶时会将之上的Activity全部出栈。++


### onNewIntent的调用时机


### Activity 的 Flags


# 第三章 View的事件体系
## View基础知识
### View的位置参数
View的位置信息由4个顶点来决定，分别为top、left、bottom、right，这都是**相对于View的父控件**来说的。所以
```
width = right - left
height = bottom - top
```

从Android3.0开始增加了x,y,translationX,translationY。其中**x，y是View左上角的坐标**，translationX和translationY**是View左上角相对于父容器的偏移量**。换算关系如下
```
x = left + translationX
y = top + translationY
```

View在平移变换中，top，left表示的是原始左上角的位置信息 ，**不会改变**，而**x，y，translationX，translationY会发生改变**

### MotionEvent 和 TouchSlop
1. MotionEvent
   
    通过MotionEvent可以获得点击事件的x，y坐标。getX/getY获取的是相对于**当前View**左上角的x和y，而getRawX/getRawY获取的是相对于**手机屏幕**左上角的坐标
2. TouchSlop
   
    TouchSlop为滑动的最小距离，当小于这个距离则不认为是滑动，**TouchSlop是一个常量**，和设备有关，在不同的设备上有可能不同，可以通过`ViewConfiguration,get(getContext()).getScaledTouchSlop()`。当处理滑动时，可以获取这个值进行判断来认为是不是滑动。

### VelocityTracker GestureDetector 和 Scroller
1. VelocityTracker

    VelocityTracker用于计算滑动速度。在View的onTouchEvent中进行创建和添加监听
    ```JAVA
    VelocityTracker velocityTracker = VelocityTracker.obtain();
    velocityTracker.addMovement(event);
    ```
    初始化完成就可以获取水平方向和垂直方向的滑动速度。**在获取速度之前，必须要先调用computeCurrentVelocity进行计算速度。**
    ```
    //必须先调用这个
    velocityTracker.computeCurrentVelocity(1000); //设置时间间隔为1000，计算在这个时间间隔中运动了多少个像素  
    Log.i("test","velocityTraker"+velocityTracker.getXVelocity());  
    Log.i("test","velocityTraker"+velocityTracker.getYVelocity());
    ```
    使用完成后，需要进行释放
    ```JAVA
    velocityTracker.clear();
    velocityTracker.recycle();
    velocityTracker = null;
    ```
2. GestureDetector

    手势检测工具，用于辅助检测用户的**单击、滑动、长按、双击**
    
    **回调接口：**
    - OnGestureListener，这个Listener监听一些手势，如单击、滑动、长按等操作；
    
        |<div style="width:150px">方法名</div>       | 描述       |
        |:------|:---:|
        |onDown | 手指轻轻触摸屏幕的一瞬间，由1个Action_down触发
        onShowPress | 手指轻轻触摸屏幕，尚未松开或拖动，由1个ACTION_DOWN触发
        onLongPress | 用户长按屏幕
        onSingleTapUp | 用户手指松开（UP事件）的时候如果没有执行onScroll()和onLongPress()这两个回调的话，就会回调这个，说明这是一个点击抬起事件，但是不能区分是否双击事件的抬起。
        onScroll | 手指按下并拖动，由1个ACTION_DOWN+多个ACTION_MOVE触发
        onFling | 用户执行快速滑动之后的回调，MOVE事件之后手松开（UP事件）那一瞬间的x或者y方向速度，如果达到一定数值（源码默认是每秒50px），就是快速滑动操作（也就是快速滑动的时候松手会有这个回调，因此基本上有onFling必然有onScroll）。
    - OnDoubleTapListener，这个Listener监听双击和单击事件。
        <div style="width:150px">方法名</div>   | 描述
        ---|---
        onSingleTapConfirmed | 可以确认（通过单击DOWN后300ms没有下一个DOWN事件确认）这不是一个双击事件，而是一个单击事件的时候会回调。
        onDoubleTap | 可以确认这是一个双击事件的时候回调。
        onDoubleTapEvent | onDoubleTap()回调之后的输入事件（DOWN、MOVE、UP）都会回调这个方法（这个方法可以实现一些双击后的控制，如让View双击后变得可拖动等）。
    - OnContextClickListener 平板接入外接键盘后，鼠标/触摸板，右键点击时候的回调。
    - **SimpleOnGestureListener，实现了上面三个接口的类，拥有上面三个的所有回调方法。一般会使用该接口**
    
    使用方式：
    ```JAVA
    //创建GestureDetector对象
    private void init(Context context) {
        mGestureDetector = new GestureDetector(context, new GestureDetector.SimpleOnGestureListener(){
            @Override //按需重写相关回调
            public boolean onDown(MotionEvent e) {
                return super.onDown(e);
            }
        });
    }
    
    //接管View的onTouchEvent
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return mGestureDetector.onTouchEvent(event);
    }
    ```
3.Scroller
    

使用Scroller对象完成弹性滑动
```JAVA
    Scroller mScroller = new Scroller(mContext);
    //缓慢移动到指定位置
    private void smoothScrollTo(int destX,int destY){
        int scrollX = getScrollX();
        int delta = destX - scrollX;
        //1000ms内滑向destX,效果就是慢慢滑动
        mScroller.startScroll(scrollX,0,delta,0,1000);
        invalidate();
    } 
    @Override
    public void computeScroll(){
        if(mScroller.computeScrollOffset()){
            scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
            postInvalidate();
        }
    }
```

## View的滑动
### 使用scrollTo/scrollBy完成View的滑动

Android源码
```JAVA
    /**
     * Set the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the x position to scroll to
     * @param y the y position to scroll to
     */
    public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    /**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {
        scrollTo(mScrollX + x, mScrollY + y);
    }
```