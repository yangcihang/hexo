---
title: 自定义Notification
date: 2017-10-21 23:12:03
tags: Android
categories: Android
---
## Notification简介

Notification是安卓开发中常用的组件之一，不过Notification在每个版本里面都有差别（安卓8.0在前台服务设置Notification的话貌似有点问题，不知道为啥在服务中start不上去...）常见的创建有两种，一种是直接构造，另一种是使用builder模式来创建，最后都用manager来notify出来（不是本文重点）。不过原生的Notification样式肯定不在各个场合都适用，因此我们有时候需要自定义Notification样式来满足不同的需求。在此之前，先简单介绍Notification中需要的一些组件。<!--more-->

## RemoteViews简介

 - RemoteViews不是当前进程的View,是属于SystemServer进程.因此可以在一些前台操作中来使用，避免一直占用UI线程的资源。应用程序和RemoteViews之间可以通过Binder来实现进程的通信。
 - 应用程序与RemoteViews之间依赖Binder实现了进程间通信.RemoteViews使用最多的场合是通知栏和桌面小插件.
 - RemoteViews并不能直接获得控件实例,然后对控件进行操作.它提供了 
setTextViewText(viewId, text)、setImageViewResource(viewId, srcId)等方法进行操作,传入控件id和相应的修改内容. 常用的属性有：
 	- setTextViewText(viewId, text)设置文本
 	- setTextColor(viewId, color)设置文本颜色
 	- setTextViewTextSize(viewId, units, size)设置文本大小 
 	- setImageViewBitmap(viewId, bitmap)设置图片
 	- setImageViewResource(viewId, srcId)根据图片资源设置图片
 	- setViewPadding(viewId, left, top, right, bottom)设置Padding间距
	- setOnClickPendingIntent(viewId, pendingIntent)设置点击事件 
 - RemoteViews的实现底层原理  最好百度看看吧...挺有意思的，这儿不是本文的重点
  
## PendingIntent简介

 - 如果你写过Notification的话，PendingIntent应该有一定的了解。这儿再介绍一下：PendingIntent是”延迟意图”的意思,就是当满足某一条件时出触发这个Intent，通过PendingIntent的getActivity、getBroadcast、getService等分别构建一个打开对应组件的延迟Intent. 
 - 传入四个参数，context、intent、requestCode(自定义)、flag.
比如这样初始化：

```
Intent intent=new Intent(MainActivity.this,MainActivity.class);
PendingIntent mPendingIntent=PendingIntent.getActivity(MainActivity.this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
```
 - PendingIntent有4种flag

 	- FLAG_ONE_SHOT                只执行一次，然后就被cancel了
 	- FLAG_NO_CREATE               若描述的Intent不存在则返回NULL值（不懂这个具体用来是干啥的）
 	- FLAG_CANCEL_CURRENT          如果描述的PendingIntent已经存在，则在产生新的Intent之前会先取消掉当前的PendingIntent。对于通知栏的消息来说，以前被取消的PendingIntent就无法启用。这种情况最常见的就是IM的后台消息提示了。
 	- FLAG_UPDATE_CURRENT          总是执行，表示描述的PendingIntent如果存在，那么它们都会更新。也就是说里面的intent的Extras会被更替为最新。
 - PendingIntent的匹配规则：requestCode表示PendingIentent发送方的请求码，多情况下设为0即可。requestCode会影响到flags效果。 
如果两个PendingIntent它们内部的Intent相同并且requestCode也相同，那么这两个PendingIntent就是相同的。intent的匹配规则是：如果两个Intent的ComponentName和intent-filter都相同，那么这两个Intent就是相同的。注意Extras不参与Intent的匹配过程。这点很重要的，因为后面的RemoteViews的监听事件需要PendingIntent来具体实现。
 - 和intent的区别
	1. Intent是立即使用的，而PendingIntent可以等到事件发生后触发，PendingIntent可以cancel
	2. Intent在程序结束后即终止，而PendingIntent在程序结束后依然有效
	3. PendingIntent自带Context，而Intent需要在某个Context内运行
	4. Intent在原task中运行，PendingIntent在新的task中运行

## 自定义步骤

1. 自定义通知的布局。 这个没啥好说的。。。不过这个只支持部分组件
 	Layout ：FrameLayout,LinearLayout,RelativeLayout，GridLayout 
 	View ：AnalogClock，Button，Chronometer，ImageButton，ImageView，ProgressBar，TextView，ViewFlipper，ListView，GridView，StackView，AdapterViewFilter，ViewStub。

2. 使用RemoteView来加载这个布局。RemoteViews没有提供findViewById方法，因此无法直接访问里面的View元素。必须通过RemoteViews所提供的一些列set方法来完成。举个例子：
  
	```  
	remoteView = new RemoteViews(this.getPackageName(), R.layout.view_music_player_notification);
	Intent intent = new Intent(BROADCAST_NAME);
	intent.putExtra(KEY_FLAG, FLAG_NEXT);
	PendingIntent nextPendingIntent = PendingIntent.getBroadcast(this, FLAG_NEXT, intent, FLAG_UPDATE_CURRENT);
	remoteView.setOnClickPendingIntent(R.id.img_next, nextPendingIntent);
	intent.putExtra(KEY_FLAG, FLAG_PRE);
	PendingIntent prePendingIntent = PendingIntent.getBroadcast(this, FLAG_PRE, intent, FLAG_UPDATE_CURRENT);
	remoteView.setOnClickPendingIntent(R.id.img_previous, prePendingIntent);
	intent.putExtra(KEY_FLAG, FLAG_PAUSE);
	PendingIntent pausePendingIntent = PendingIntent.getBroadcast(this, FLAG_PAUSE, intent, FLAG_UPDATE_CURRENT);
	```
这是一个用广播来实现音乐播放器的例子。这个播放器就是通知栏展示的那种。

3. 使用RemoteView实例创建Notification。这个简单，如果是buider模式的话，调用setContent方法，此时的参数要求就是RemoteViews。如果是直接构造的话，将RemoteViews直接赋值给notification的ContentView。

4. 处理事件。事件是通过intent的action来标识的，而intent是“嵌套”在PendingIntent中的，因此点击后，先根据PendingIntent的类型来决定的。这个时候可以用intent进行参数传递，再在相应的代码块中实现相应的方法。比如上面的PendingIntent是构建了一个打开广播的intent，因此我们可以在相应的BroadcastReceiver中进行相关方法的实现。
