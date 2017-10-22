---
title: Android优化--布局优化
date: 2017-10-21 23:21:01
tags: Android
categories: Android
---

系统在渲染UI界面时将要消耗大量的资源。熟悉View绘制流程的小伙伴们应该知道，View绘制是一层层对子View进行遍历的。因此，布局的好坏影响着用户的体验。

在Android中，系统通过VSYNC信号触发对UI的渲染、重绘。其间隔时间是16ms、如果系统每次渲染的时间都在16ms内，那么我们看到的界面应该是相当流畅的。如果发生帧率和刷新频率不一致的情况时，就会出现卡顿。<!--more-->

## 避免Overdraw
什么是overdraw呢？描述的是屏幕上的某个像素在同一帧的时间内被绘制了多次。在多层次的UI结构里面，如果不可见的UI也在做绘制的操作，就会导致某些像素区域被绘制了多次，浪费大量的CPU以及GPU资源。（可以通过开发者选项，打开Show GPU Overdraw的选项，观察UI上的Overdraw情况，红色区域就是过绘制）。例如默认系统会绘制Activity背景，而如果再给布局绘制了重叠的背景，那么默认Activity的背景就属于过度绘制。

## 优化布局层级
一开始已经提到了，View的绘制都是通过对View树的遍历来进行的。如果一个View树太高，就会严重影响其测量、布局、绘制的速度。Google官方建议View树最高不过10层。所以我们为了提高效率，尽量在一些复杂的场合下使用RelativeLayout来降低View树的高度。

## 善于使用抽象标签
使用抽象标签在某些场合下能减少代码量，还能起到优化布局的效果。
### include标签
include标签可以将布局中的公共部分复用。比如一些自定义的toolbar或者底部的导航栏之类能复用的东西。使用起来也是很方便的，比如

```
 < include layout="@layout/item_test_linear_layout"  />
```
这样就把一个自定义的组件加载到了布局中。

### merge标签
merge标签是作为include标签的一种辅助扩展来使用，它的主要作用是为了防止在引用布局文件时产生多余的布局嵌套。Android渲染需要消耗时间，布局越复杂，性能就越差。如上述include标签引入了之前的LinearLayout之后导致了界面多了一个层级。在布局中用merge就可以减少绘制一层viewGroup

## viewStub标签
viewstub是view的子类。他是一个轻量级View， 隐藏的，没有尺寸的View。他可以用来在程序运行时简单的填充布局文件。

```
< ViewStub
        android:id="@+id/vs_test"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout="@layout/item_test_linear_layout"
        android:layout_marginTop="10dp"  />
```
运行后，我们会发现这个使用viewstub标签的布局果然没有显示出来。如何重新加载显示的布局呢？
首先，我们通过普通的findViewById来找到这个组件，然后，有两种方法可以展示这个组件的内容：

* 我们可以setVisibility来将他视为可见。但是这样使用的话没法返回显示的布局
* 使用inflate方法来展示。inflate()方法会返回一个viewStub布局，然后我们就可以使用这个返回的布局通过findViewById的方法来获取里面响应的控件了。

不管哪种方法使用后，viewStub就不存在了，取而代之的就是其引用的layout。因此调用两次inflate方法会报错的。