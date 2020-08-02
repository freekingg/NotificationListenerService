title: NotificationListenerService的那些事儿
date: 2017-02-09 21:00:04
categories: Android Blog
tags: [Android]
---
最近在公司时接到一个需求：需要实时监听设备的通知栏消息，并可以捕获到通知的内容，然后进行对应的操作。刚看到这个需求的时候，脑子里第一反应就是使用 `AccessibilityService` 。 `AccessibilityService` 支持的事件监听类型中有 `TYPE_NOTIFICATION_STATE_CHANGED` ，该事件类型就是用来监听通知栏消息状态改变的，众多的抢红包插件利用的就是这个原理。

之后在 Github 上看到了 [qianghongbao](https://github.com/lendylongli/qianghongbao) 这个抢红包的项目，发现代码里面有一个 [QHBNotificationService](https://github.com/lendylongli/qianghongbao/blob/master/app/src/main/java/com/codeboy/qianghongbao/QHBNotificationService.java) 继承了 `NotificationListenerService` ，这个 `NotificationListenerService` 极大地引起了我的兴趣。查了一下资料，发现 `NotificationListenerService` 是在 Android 4.3 （API 18）时被加入的，作用就是用来监听通知栏消息。并且官方建议在 Android 4.3 及以上使用 `NotificationListenerService` 来监听通知栏消息，以此取代 `AccessibilityService` 。

![Notification Listener](/uploads/20170209/20170209220914.png)

`NotificationListenerService` 的使用范围也挺广的，比如我们熟知的抢红包，智能手表同步通知，通知栏去广告工具等，都是利用它来完成的。所以，我也想赶时髦地好好利用这把“利器”。最后方案也就出来了：在 Android 4.3 以下（API < 18）使用 `AccessibilityService` 来读取新通知，在 Android 4.3 及以上（API >= 18）使用 `NotificationListenerService` 来满足需求。

这也正是本篇博客诞生的“起源”。

NotificationListenerService
===========================
在这里，我们就做一个小需求：实时检测微信的新通知，如果该通知是微信红包的话，就进入微信聊天页面。

准备好了吗，我们开始吧！

首先创建一个 `WeChatNotificationListenerService` 继承 `NotificationListenerService` 。然后在 `AndroidManifest.xml` 中进行声明相关权限和 `<intent-filter>` ：

``` xml
<service android:name="com.yuqirong.listenwechatnotification.WeChatNotificationListenerService"
          android:label="@string/app_name"
          android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
     <intent-filter>
         <action android:name="android.service.notification.NotificationListenerService" />
     </intent-filter>
</service>
```

然后一般会重写下面这三个方法：

*  `onNotificationPosted(StatusBarNotification sbn)` ：当有新通知到来时会回调；
*  `onNotificationRemoved(StatusBarNotification sbn)` ：当有通知移除时会回调；
*  `onListenerConnected()` ：当 `NotificationListenerService` 是可用的并且和通知管理器连接成功时回调。

onNotificationPosted(StatusBarNotification sbn)
-----------------------------------------------
下面我们来看看 `NotificationListenerService` 中的重点： `onNotificationPosted(StatusBarNotification sbn)` 方法。

``` java
@Override
public void onNotificationPosted(StatusBarNotification sbn) {
    // 如果该通知的包名不是微信，那么 pass 掉
    if (!"com.tencent.mm".equals(sbn.getPackageName())) {
        return;
    }
    Notification notification = sbn.getNotification();
    if (notification == null) {
        return;
    }
    PendingIntent pendingIntent = null;
    // 当 API > 18 时，使用 extras 获取通知的详细信息
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        Bundle extras = notification.extras;
        if (extras != null) {
            // 获取通知标题
            String title = extras.getString(Notification.EXTRA_TITLE, "");
            // 获取通知内容
            String content = extras.getString(Notification.EXTRA_TEXT, "");
            if (!TextUtils.isEmpty(content) && content.contains("[微信红包]")) {
                pendingIntent = notification.contentIntent;
            }
        }
    } else {
        // 当 API = 18 时，利用反射获取内容字段
        List<String> textList = getText(notification);
        if (textList != null && textList.size() > 0) {
            for (String text : textList) {
                if (!TextUtils.isEmpty(text) && text.contains("[微信红包]")) {
                    pendingIntent = notification.contentIntent;
                    break;
                }
            }
        }
    }
    // 发送 pendingIntent 以此打开微信
    try {
        if (pendingIntent != null) {
            pendingIntent.send();
        }
    } catch (PendingIntent.CanceledException e) {
        e.printStackTrace();
    }
}
```

从上面的代码可知，对于分析 `Notification` 的内容分为了两种：

* 当 API > 18 时，利用 `Notification.extras` 来获取通知内容。`extras` 是在 API 19 时被加入的；
* 当 API = 18 时，利用反射获取 `Notification` 中的内容。具体的代码在下方。

``` java
public List<String> getText(Notification notification) {
    if (null == notification) {
        return null;
    }
    RemoteViews views = notification.bigContentView;
    if (views == null) {
        views = notification.contentView;
    }
    if (views == null) {
        return null;
    }
    // Use reflection to examine the m_actions member of the given RemoteViews object.
    // It's not pretty, but it works.
    List<String> text = new ArrayList<>();
    try {
        Field field = views.getClass().getDeclaredField("mActions");
        field.setAccessible(true);
        @SuppressWarnings("unchecked")
        ArrayList<Parcelable> actions = (ArrayList<Parcelable>) field.get(views);
        // Find the setText() and setTime() reflection actions
        for (Parcelable p : actions) {
            Parcel parcel = Parcel.obtain();
            p.writeToParcel(parcel, 0);
            parcel.setDataPosition(0);
            // The tag tells which type of action it is (2 is ReflectionAction, from the source)
            int tag = parcel.readInt();
            if (tag != 2) continue;
            // View ID
            parcel.readInt();
            String methodName = parcel.readString();
            if (null == methodName) {
                continue;
            } else if (methodName.equals("setText")) {
                // Parameter type (10 = Character Sequence)
                parcel.readInt();
                // Store the actual string
                String t = TextUtils.CHAR_SEQUENCE_CREATOR.createFromParcel(parcel).toString().trim();
                text.add(t);
            }
            parcel.recycle();
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return text;
}
```

凭着 `onNotificationPosted(StatusBarNotification sbn)` 方法就已经可以完成监听微信通知并打开的动作了。下面我们来看一下其他关于 `NotificationListenerService` 的二三事。

取消通知
-------
有了监听，`NotificationListenerService` 自然提供了可以取消通知的方法。取消通知的方法有：

* `cancelNotification(String key)` ：是 API >= 21 才可以使用的。利用 `StatusBarNotification` 的 `getKey()` 方法来获取 `key` 并取消通知。
* `cancelNotification(String pkg, String tag, int id)` ：在 API < 21 时可以使用，在 API >= 21 时使用此方法来取消通知将无效，被废弃。

最后，取消通知的方法：

``` java
public void cancelNotification(StatusBarNotification sbn) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        cancelNotification(sbn.getKey());
    } else {
        cancelNotification(sbn.getPackageName(), sbn.getTag(), sbn.getId());
    }
}
```

检测通知监听服务是否被授权
-----------------------

``` java
public boolean isNotificationListenerEnabled(Context context) {
    Set<String> packageNames = NotificationManagerCompat.getEnabledListenerPackages(this);
    if (packageNames.contains(context.getPackageName())) {
        return true;
    }
    return false;
}
```

打开通知监听设置页面
------------------

``` java
public void openNotificationListenSettings() {
    try {
        Intent intent;
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.LOLLIPOP_MR1) {
            intent = new Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS);
        } else {
            intent = new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS");
        }
        startActivity(intent);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

被杀后再次启动时，监听不生效的问题
------------------------------
这个问题来源于知乎问题： [NotificationListenerService不能监听到通知，研究了一天不知道是什么原因？](https://www.zhihu.com/question/33540416)

从问题的回答中可以了解到，是因为 `NotificationListenerService` 被杀后再次启动时，并没有去 `bindService` ，所以导致监听效果无效。

最后，在回答中还给出了解决方案：利用 `NotificationListenerService` 先 disable 再 enable ，重新触发系统的 rebind 操作。代码如下：

``` java
private void toggleNotificationListenerService() {
    PackageManager pm = getPackageManager();
    pm.setComponentEnabledSetting(new ComponentName(this, com.fanwei.alipaynotification.ui.AlipayNotificationListenerService.class),
            PackageManager.COMPONENT_ENABLED_STATE_DISABLED, PackageManager.DONT_KILL_APP);
    pm.setComponentEnabledSetting(new ComponentName(this, com.fanwei.alipaynotification.ui.AlipayNotificationListenerService.class),
            PackageManager.COMPONENT_ENABLED_STATE_ENABLED, PackageManager.DONT_KILL_APP);
}
```

该方法使用前提是 `NotificationListenerService` 已经被用户授予了权限，否则无效。另外，在自己的小米手机上实测，重新完成 rebind 操作需要等待 10 多秒（我的手机测试过大概在 13 秒左右）。幸运的是，官方也已经发现了这个问题，在 API 24 中提供了 `requestRebind(ComponentName componentName)` 方法来支持重新绑定。

AccessibilityService
====================
讲完了 `NotificationListenerService` 之后，按照前面说的那样，在 API < 18 的时候使用 `AccessibilityService` 。

同样，创建一个 `WeChatAccessibilityService` ，并且在 `AndroidManifest.xml` 中进行声明：

``` java
<service
    android:name="com.yuqirong.listenwechatnotification.WeChatAccessibilityService"
    android:label="@string/app_name"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessible_service_config" />
</service>
```

声明之后，还要对 `WeChatAccessibilityService` 进行配置。需要在 res 目录下新建一个 xml 文件夹，在里面新建一个 accessible_service_config.xml 文件：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeNotificationStateChanged"
    android:accessibilityFeedbackType="feedbackAllMask"
    android:accessibilityFlags="flagIncludeNotImportantViews"
    android:canRetrieveWindowContent="true"
    android:description="@string/app_name"
    android:notificationTimeout="100"
    android:packageNames="com.tencent.mm" />
```

最后就是代码了：

``` java
public class WeChatAccessibilityService extends AccessibilityService {

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        if (Build.VERSION.SDK_INT < 18) {
            Notification notification = (Notification) event.getParcelableData();
            List<String> textList = getText(notification);
            if (textList != null && textList.size() > 0) {
                for (String text : textList) {
                    if (!TextUtils.isEmpty(text) &&
                            text.contains("[微信红包]")) {
                        final PendingIntent pendingIntent = notification.contentIntent;
                        try {
                            if (pendingIntent != null) {
                                pendingIntent.send();
                            }
                        } catch (PendingIntent.CanceledException e) {
                            e.printStackTrace();
                        }
                    }
                    break;
                }
            }
        }
    }

    @Override
    public void onInterrupt() {

    }
    
}
```

看了一圈 `WeChatAccessibilityService` 的代码，发现和 `WeChatNotificationListenerService` 在 API < 18 时处理的逻辑是一样的，`getText(notification)` 方法就是上面那个，在这里就不复制粘贴了，基本没什么好讲的了。

有了 `WeChatAccessibilityService` 之后，在 API < 18 的情况下也能监听通知啦。\\(^ο^)/

我们终于实现了当初许下的那个需求了。 cry ...

总结
====
除了监听通知之外，`AccessibilityService` 还可以进行模拟点击、检测界面变化等功能。具体的可以在 GitHub 上搜索抢红包有关的 Repo 进行深入学习。

而 `NotificationListenerService` 的监听通知功能更加强大，也更加专业。在一些设备上，如果 `NotificationListenerService` 被授予了权限，那么可以做到该监听进程不死的效果，也算是另类的进程保活。

今天就到这儿了，拜拜！！

源码下载：[ListenWeChatNotification.rar](/uploads/20170209/ListenWeChatNotification.rar)

References
==========
* [Android 4.3 APIs](https://developer.android.google.cn/about/versions/android-4.3.html#NotificationListener)
* [NotificationListenerService](https://developer.android.google.cn/reference/android/service/notification/NotificationListenerService.html)
* [NotificationListenerService不能监听到通知，研究了一天不知道是什么原因？](https://www.zhihu.com/question/33540416)