---
title: Android屏幕适配知识总结
date: 2022-06-15 18:04:56
tags: Android
categories: 
- Android
---

## 1. 概念解释

### 屏幕尺寸

手机对角线的物理尺寸，单位：英寸（inch），1英寸=2.54cm

### 屏幕分辨率

手机在横向、纵向上的像素点数的总和，由横向像素数 * 纵向像素数表示（1280*720），每个像素都是1px

#### 屏幕PPI（屏幕像素密度）

表示每英寸像素数，则在相同尺寸下，PPI越高，则越清晰。在PPI相同的情况下，屏幕尺寸越大，越模糊。

![PPI计算规则](/images/PPI计算规则.webp)

eg：小米8的像素密度为402PPI

#### DPI

是印刷业使用的单位，其表示的是打印纸上每一英寸包含的墨点数量，*网络上的观点是Google误用了该单位*。在android中可以将PPI与DPI等价看待。

物理上DPI值是一个固定值，无法修改。但Android系统中的`densityDpi`参数可以通过root或adb命令修改。修改后会有一个`Physical density` 与`Override density`。

### Density 

- `DisplayMetrics.density`
	该density是为了方便px与dp的转换，额外提供了此参数，density = densityDpi / 160。
- `adb shell wm density`
	这个density其实就是dpi的概念。

*很多文章中的density也就是指的是dpi*。

### DP

密度无关像素(density-independent pixel)，与终端上的实际物理像素点无关。

在dpi = 160的设备上，1dp=1px。当dpi提升到240时，1dp所能表示的像素点提高，扩大到(240/160)=1.5倍，此时1dp=1.5dp。

dp与px转换可以使用 `px = dp * (densityDpi / 160)`或者`px = dp * density`。

### SP

可缩放像素，与dp基本相同，用于字体大小设置。唯一不同就是sp会随着系统设置的字体缩放进行缩放。

<!-- more -->

## 2.适配方案

##### 为什么不使用Android的dp适配方案

> 屏幕分辨率为：1920*1080，屏幕尺寸为5吋的话，那么dpi为440。假设我们UI设计图是按屏幕宽度为360dp来设计的，那这样会存在什么问题呢？
在上述设备上，屏幕宽度其实为1080/(440/160)=392.7dp，也就是屏幕是比设计图要宽的。这种情况下， 即使使用dp即使写了最大的dp值，也无法填充满屏幕。 同时还存在部分设备屏幕宽度不足360dp，这时就会导致按360dp宽度来开发实际显示不全的情况。

**会导致按设计图换算过来的最大尺寸小于设备实际最大尺寸**



### 使用RelativeLayout、ConstraintLayout进行百分比布局

如果只考虑控件大小与位置情况，使用此方案可以解决大部分问题，但对于涉及到TextView等文字大小显示问题，依旧存在上述dp方案所存在的缺陷问题。（相同sp实际在不同手机中显示的大小不一致）

### 尺寸（size）限定符适配

将设计图的尺寸作为基准，比如设计图为360x640，手机实际的分辨率根据这个基准宽高生成对应的尺寸文件。

例如一款720x1280的手机：

宽为720，将宽度分为360份，取值为x1到x360，则取值如下

```
<dimen name="x1">2.0px</dimen>
<dimen name="x2">4.0px</dimen>
...
```

高也同理，生成相应的y1,y2...y640的数值。

将生成的dimens文件放到values-720x1280中，开发中使用R.dimens.x120，R.dimens.y60进行布局开发。

**如果手机实际尺寸没有找到对应的尺寸限定符，就会去默认的dimens中取尺寸**

**缺点：**

1. 需要生成大量的dimes文件。该方案为了适配及其小众的尺寸手机，会占用大量存储空间，导致增加安装包的大小。即使生成了大量的文件，也不可能覆盖不到全部的机型。

2. 手机实际尺寸比例可能与设计稿不同，导致UI拉伸。设计图可能是16:9，而手机可能是16:10等其他尺寸。



### 最小宽度（Smallest-width）限定符

同尺寸限定符适配同理，通过生成大量的dimen文件期望覆盖大量机型尺寸。与尺寸限定符不同的是，尺寸限定符在没有匹配到相应的限定符时，会直接去默认的dimen中寻找尺寸，而sw限定符如果没有找到与自己相同的文件夹，会找小一点的最近dimens，这样，只要dimens文件夹生成的间隔控制的得当，UI也不会有很大的偏差，也不需要占用大量的安装包空间。

系统会将屏幕较小的那一边的尺寸计算为dp，计算后的值就去匹配对应限定符。

> 比如分辨率为1280x720，densityDpi为240dpi的设备。短边为720px，此设备的denstiy=240/160=1.5，那么最小宽度即为720/1.5=480dp (px = dp * density)，这样这个设备会从value-sw480dp的文件夹下取资源。

而每个限定符文件夹中的dimens的生成规则和尺寸限定符相同，按照设计图的尺寸与最小宽度dp进行比例计算

> 假如设计图的分辨率为1920x1080，设备计算出的最小宽度为sw440dp，那么设计图中的1px = 440dp/1080 = 0.407dp，依据此规则依次生成其他尺寸的px所对应的dp值即可。

选用最小宽度进行适配还有一个好处是布局过程中UI的宽高都使用一个边生成的尺寸进行开发，不像尺寸适配符宽取x的值，高取y的值，使用两个边的尺寸开发，当出现屏幕尺寸比例与设计图比例不相同的时候，不会被拉伸。



更多的限定符见[限定符表](https://developer.android.com/guide/topics/resources/providing-resources?hl=zh-cn)

### 头条适配方案

Android原生dp方案最大的缺点是每个View的dp是写死不变的，但是手机最大的dp值随着尺寸的变化而变化。而头条方案保证了设计图中标注的最大dp值转换出的px值为都为手机的最大尺寸。

头条方案通过计算屏幕尺寸与设计图的比例，计算出`targetDensity = screenWidth / uiWidth;`

动态修改`DisplayMetrics#density `,`DisplayMetrics#densityDpi`,参数，使得系统在将xml中的dp值转换为px时，`px = dp * targetDensity`，保证了在各个屏幕尺寸的一致性。

在转换sp时，`DisplayMetrics#scaledDensity(字体的缩放因子)`需要对系统字体缩放进行进一步处理

```
targetScaledDensity = targetDensity  * (scaledDensity / density)
```

并需要在`Application#registerComponentCallbacks` 注册下 `onConfigurationChanged`监听字体缩放变化，实时更改最新的`targetScaledDensity`。



[一种极低成本的Android屏幕适配方式](https://mp.weixin.qq.com/s/d9QCoBP6kV9VSWvVldVVwA)

[骚年你的屏幕适配方式该升级了!-今日头条适配方案](https://juejin.cn/post/6844903661819133960)

[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize)

## 其他尺寸问题

### getDimension、getDimensionPixelOffset 和 getDimensionPixelSize区别

- 相同点：返回获取某个dimen的值，如果dimen单位是dp或sp，则需要将其乘以density（屏幕密度）；如果单位是px，则不用。

- 不同点：
  `getDimension`：返回类型为float，
  `getDimensionPixelSize`：返回类型为int，由浮点型转成整型时，采用四舍五入原则。 
  `getDimensionPixelOffset`：返回类型为int，由浮点型转成整型时，原则是忽略小数点部分。

  

### TextView中的setTextSize参数问题

```java TextView.java
public void setTextSize(int unit, float size) 
```

其实很好理解，参数unit就是代表传入的size的类型，如果unit为`TypedValue.COMPLEX_UNIT_SP`,那么就会将传入的size按sp的方式计算（乘density），最终转化为px值。如果直接传入`TypedValue.COMPLEX_UNIT_PX`，就不需要转换为px值。

**一个小例子：**

```java example.java
textView.setTextSize(TypedValue.COMPLEX_UNIT_SP,getResources().getDimension(R.dimen.fontsize))
```

```xml app/src/main/res/values/dimens.xml
<dimen name="font1">18sp</dimen>
```

这种情况时，`getResources().getDimension()`在获取值的时候已经将18\*density转换为px值后，然后再TextView中设置时又再次\*density，相当于TextView最终设置到的字体大小为18\*density\*density。

## 附录：常见密度限定符对应表

| 密度限定符 | 说明                                                         |
| :--------- | :----------------------------------------------------------- |
| `ldpi`     | 适用于低密度 (ldpi) 屏幕 (~ 120dpi) 的资源。                 |
| `mdpi`     | 适用于中密度 (mdpi) 屏幕 (~ 160dpi) 的资源（这是基准密度）。 |
| `hdpi`     | 适用于高密度 (hdpi) 屏幕 (~ 240dpi) 的资源。                 |
| `xhdpi`    | 适用于加高 (xhdpi) 密度屏幕 (~ 320dpi) 的资源。              |
| `xxhdpi`   | 适用于超超高密度 (xxhdpi) 屏幕 (~ 480dpi) 的资源。           |
| `xxxhdpi`  | 适用于超超超高密度 (xxxhdpi) 屏幕 (~ 640dpi) 的资源。        |
| `nodpi` | 适用于所有密度的资源。这些是与密度无关的资源。无论当前屏幕的密度是多少，系统都不会缩放以此限定符标记的资源。 |
| `tvdpi` | 适用于密度介于 mdpi 和 hdpi 之间的屏幕（约 213dpi）的资源。这不属于“主要”密度组。它主要用于电视，而大多数应用都不需要它。对于大多数应用而言，提供 mdpi 和 hdpi 资源便已足够，系统将视情况对其进行缩放。如果您发现有必要提供 tvdpi 资源，应按一个系数来确定其大小，即 1.33*mdpi。例如，如果某张图片在 mdpi 屏幕上的大小为 100px x 100px，那么它在 tvdpi 屏幕上的大小应该为 133px x 133px。 |