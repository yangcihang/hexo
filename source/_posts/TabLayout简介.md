---
title: TabLayout简介
date: 2017-10-21 23:12:48
tags: Android
categories: Android
---

在app的主页等地方加入tabs是很常见的，它有许多实现方式，比如自定义控件等。在Google的Design库中，推出了TabLayout这么一个布局来方便开发者们实现tab的效果。TabLayout提供了一个水平的布局用来展示Tabs。[参考链接](http://www.jianshu.com/p/2b2bb6be83a8 "参考链接")<!--more-->

## TabLayout的相关属性
1. 改变选中字体的颜色
	```app:tabSelectedTextColor="@android:color/	holo_orange_light"```
2. 改变未选中字体的颜色
	```app:tabTextColor="@color/colorPrimary"```
3. 改变指示器下标的颜色
	```app:tabIndicatorColor="@android:color/	holo_orange_light"```
4. 改变整个TabLayout的颜色
	```app:tabBackground="color"```
5. 改变指示器下表高度
	```app:tablndicatorHeight="1dp"```
6. （代码中）添加tab
	```tabLayout.addTab(tabLayout.newTab().setText("123"))```
7. （布局文件中）添加tab
	在TabLaytou标签下面，添加TabItem标签即可。
8. 添加图标
	```tabLayout.addTab().setIcon(R.mipmap.ic_launcher)```
9. 设置模式
	有固定和滚动模式可选择
	```app:tabMode = "scrollable"```
	```app:tabMode = "fixed"```
10. Tab宽度限制
	```app:tabMaxWidth="xxxdp"```
	```app:tabMinWidt="xxxdp"```
11. 最重要的，和ViewPager联动
	和ViewPager联动，需要重写adapter 的getPagerTitle方法,否则无法显示TabLayout上的标签
	```tabLayout.setupWithViewPager(vp)```
