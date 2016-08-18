# Android中Notification捕捉点击事件的替代方式

来源:[http://blog.csdn.net/liuweiballack/article/details/48009109](http://blog.csdn.net/liuweiballack/article/details/48009109)

在处理程序中的通知消息时，一般都是用Notification类来处理，通过设置PendingIntent来处理点击通知之后的动作。与一般的Intent不同，PendingIntent表示即将要执行的动作，是在用户点击消息之后才进行处理，它里面保存了一个Intent用来执行跳转的操作。

但是有一些需求，要求在用户点击通知之后，还需要执行一些其他的操作,并非单纯的进行activity之间的跳转。因此需要对notification的点击事件进行捕捉，但是Android中并没有提供这样的API供开发者使用。我们无法对系统通知的点击事件进行捕捉，这就需要找到一种其他的方式解决这个问题。

在PendingIntent中提供了一个方法getBroadcast( )，我们可以通过广播的方式进行处理：当用户点击通知时，发送一个广播，当广播接收者收到这个广播时，在进行对应的逻辑处理。

具体代码如下：

定义方法showNotification

```
private void showNotification() {
        // 创建一个NotificationManager的引用
        NotificationManager notificationManager = (NotificationManager) this.getSystemService(Context.NOTIFICATION_SERVICE);

        // 定义Notification的各种属性
        Notification notification = new Notification(R.drawable.ic_launcher, "新消息", System.currentTimeMillis());
        NotificationCompat.Builder builder = new NotificationCompat.Builder(mContext);
        builder.setSmallIcon(R.drawable.ic_launcher);
        notification.flags |= Notification.FLAG_AUTO_CANCEL;

        // 设置通知的事件消息
        CharSequence contentTitle = "Title"; // 通知栏标题
        CharSequence contentText = "Text"; // 通知栏内容

        Intent clickIntent = new Intent(mContext, NotificationClickReceiver.class); //点击通知之后要发送的广播
        int id = (int) (System.currentTimeMillis() / 1000);
        PendingIntent contentIntent = PendingIntent.getBroadcast(this.getApplicationContext(), id, clickIntent, PendingIntent.FLAG_UPDATE_CURRENT);
        notification.setLatestEventInfo(this, contentTitle, contentText, contentIntent);
        notificationManager.notify(id, notification);
}
```

定义NotificationClickReceiver继承 BroadcastReceiver，NotificationClickReceiver需要在AndroidManifest.xml中注册或者使用其他方法注册

```
public class NotificationClickReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        //todo 跳转之前要处理的逻辑
        Intent newIntent = new Intent(context, ActivityMain.class).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(newIntent);
    }
}
```

在用户点击通知之后，NotificationClickReceiver 会接受到通知的点击广播，执行onReceive()方法。

**注意：**<br/>
**1、onReceive（）方法中不能执行耗时操作。**<br/>
**2、在onReceive（）的context中启动activity，需要加上 Intent.FLAG_ACTIVITY_NEW_TASK，否则会崩溃。**

