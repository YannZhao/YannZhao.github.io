---
layout: post
title:  "自定义自动适配各手机系统的Notification"
desc: "拥有三个page页的自定义View"
keywords: "notification,system,android"
date: 2016-2-15
categories: [Android]
tags: [android, Notification, System]
---
# 起因
很久之前接到一个需求，需要在通知栏常驻一个notification，title和subtitle都比较短，在末尾加一个类似数据状态的提示。常驻notification很简单，通过service发送一个前台服务就可以了，自定义RemoteViews也很简单，写好对应的layout就好了，大体样式可以做的跟系统的notification差不多。

但是发现做完之后，在各种手机上的performance相差很多，特别是小米的，简直奇丑无比，除非把整个色块全都上色，类似360那种。出于强迫症，十分想把这个notification做的跟系统一个样式，这样可以适配各大手机。于是开始对系统源码以及hierarchyViewer进行分析。

###分析
> 1. 通过hierarchyViewer，理清楚SystemUI中notification item的视图结构；查找源码了解到对应layout的布局情况。
> 2. 如果可以保留notification原本的layout，然后整体添加到新的layout中，即可保留系统样式，
> 3. 然后往新的layout中添加显示数据状态的view。

### hierarchyViewer
<img src="/static/img/blog/customnotification/system_ui_notification_item_desc.jpg" width="90%">

找到了notification布局中Notification item layout中，RemoteViews的contentView的id为“status_bar_latest_event_content”

#### 代码实现
```java
public static void senNotification(Context context) {
	...
	    //通过反射找到对应parent的id
		int id = Resources.getSystem().getIdentifier("status_bar_latest_event_content", "id", "android");
		Notification notification;
		//如果找到了id则自定义notification，否则照常
		if (id != 0) {
		    //new一个notification，通过setXXX设置好action，然后build出来
			notification = new NotificationCompat.Builder(context).setContentTitle("这是一个自定义的notifications")
					.setContentText("能保留系统Notification中title和subtitle的style").setContentIntent(pendingIntent)
					.setAutoCancel(true).setSmallIcon(R.mipmap.ic_launcher).setShowWhen(false).build();
			//把系统样式的RemoteViews 克隆一份
			RemoteViews contentView = notification.contentView.clone();
			//接着把parent的child全部清除，等待添加自定义的RemoteViews
			notification.contentView.removeAllViews(id);
			//自定义的parent，将被添加到notification.contentView
			RemoteViews customParentView = new RemoteViews(context.getPackageName(),
					R.layout.layout_custom_notification);
			//把系统样式的RemoteView添加到自定义Parent中
			customParentView.addView(R.id.custom_notification_item_parent, contentView);
			//把现实数据状态的view添加到自定义Parent中
			RemoteViews customChildView = new RemoteViews(context.getPackageName(),
					R.layout.layout_custom_notification_child);
			customChildView.setTextViewText(R.id.child, "Child");
			customParentView.addView(R.id.custom_notification_item_parent, customChildView);
			//把自定义Parent添加到contentView中
			notification.contentView.addView(id, customParentView);
		} else {
		    //正常发送notification
			...
		}
		//常驻notification需要在service中通过发送前台服务实现，此处不赘述，就简单发送一个通知
		notificationManager.notify(1, notification);
	}
```

#### custom_notification_item_parent.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/custom_notification_item_parent"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:gravity="center_vertical">

</RelativeLayout>
```

#### layout_custom_notification_child.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/child"
    android:layout_width="wrap_content"
    android:layout_height="match_parent"
    android:layout_centerVertical="true"
    android:layout_alignParentRight="true"
    android:layout_marginRight="8dp"
    android:gravity="center"
    android:background="@color/theme_white"
    android:textColor="@color/theme_black"
    android:textSize="14sp" />
```

这样，一个风格适配系统的自定义notification就完成了，不过目前有一个弊端还无法解决，就是title和subtitle太长，就会被自定义的child遮挡，不过就之前的需求而言，目前的实现足够满足了。

[代码地址](https://github.com/YannZhao/MVP-ViewModel)


