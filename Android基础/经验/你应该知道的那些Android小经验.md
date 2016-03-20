# 你应该知道的那些Android小经验

来源:[www.jayfeng.com](http://www.jayfeng.com/2016/03/18/%E4%BD%A0%E5%BA%94%E8%AF%A5%E7%9F%A5%E9%81%93%E7%9A%84%E9%82%A3%E4%BA%9BAndroid%E5%B0%8F%E7%BB%8F%E9%AA%8C/#rd?sukey=a76cdd086edb4fce86852e903fb2fe84a37e0e3334b1c607cb826d09652d1be9f37c6ec1eaf275053796517bad5e6659)

做Android久了，就会踩很多坑，被坑的多了就有经验了，闲暇之余整理了部分，现挑选一些重要或者偏门的“小”经验做个记录。

## 查看SQLite日志

```
adb shell setprop log.tag.SQLiteLog V
adb shell setprop log.tag.SQLiteStatements V
```

因为实现里用了`Log.isLoggable(TAG, Log.VERBOSE)`做了判断，`LessCode`的`LogLess`中也参考了这种机制：[LogLess](https://github.com/openproject/LessCode/blob/master/lesscode-core/src/main/java/com/jayfeng/lesscode/core/LogLess.java)。

使用这种方法就可以在Release版本也能做到查看应用的打印日志了。

## PNG优化

APK打包会自动对PNG进行无损压缩，如果自行无损压缩是无效的。

当然进行有损压缩是可以的：[https://tinypng.com/](https://tinypng.com/)

## Tcpdump抓包

有些模拟器比如genymotion自带了tcpdump，如果没有的话，需要下载tcpdump:
[http://www.strazzere.com/android/tcpdump](http://www.strazzere.com/android/tcpdump)

把tcpdump push到/data/local下，抓包命令：

````
adb shell  /data/local/tcpdump -i any -p -s 0 -w /sdcard/capture.pcap
```

## 查看签名

很多开发者服务都需要绑定签名信息，用下面的命令可以查看签名：

```
keytool -list -v -keystore release.jks
```

注意，这个是需要密码的，可以查看MD5, SHA1，SHA256等等。

## 单例模式(懒汉式)的更好的写法

特别说到这个问题，是因为网上很多这样的代码：

```
public class Singleton {
    private static Singleton instance;
    private Singleton (){}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

这种写法线程不安全，改进一下，加一个同步锁：

```
public class Singleton {
    private static Singleton instance;
    private Singleton (){}
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

网上这样的代码更多，可以很好的工作，但是缺点是效率低。

实际上，早在JDK1.5就引入volatile关键字，所以又有了一种更好的双重校验锁写法：

```
public class Singleton {
    private volatile static Singleton singleton;
    private Singleton (){}
    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

注意，别忘记volatile关键字哦，否则就是10重，100重也可能还是会出问题。

上面是用的最多的，还有一种静态内部类写法更推荐：

```
publlic class Singleton {
    private Singleton() {}
    private static class SingletonLoader {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonLoader.INSTANCE;
    }
}
```

## 多进程Application

是不是经常发现Application里的方法执行了多次？百思不得其解。

因为当有多个进程的时候，Application会执行多次，可以通过pid来判断那些方法只执行一次，避免浪费资源。

## 隐式启动Service

这是Android5.0的一个改动，不支持隐式的Service调用。下面的代码在Android 5.0+上会报错：`Service Intent must be explicit`：

```
Intent serviceIntent = new Intent();
serviceIntent.setAction("com.jayfeng.MyService");
context.startService(serviceIntent);
```

可改成如下：

```
// 指定具体Service类，或者有packageName也行
Intent serviceIntent = new Intent(context, MyService.class);
context.startService(serviceIntent);
```

## fill_parent的寿命

在Android2.2之后，支持使用match_parent。你的布局文件里是不是既有fill_parent和match_parent显得很乱？

如果你现在的minSdkVersion是8+的话，就可以忽略fill_parent，统一使用match_parent了，否则请使用fill_parent。

## ListView的局部刷新

有的列表可能`notifyDataSetChanged()`代价有点高，最好能局部刷新。

**局部刷新的重点是，找到要更新的那项的View，然后再根据业务逻辑更新数据即可。**

```
private void updateItem(int index) {
    int visiblePosition = listView.getFirstVisiblePosition();
    if (index - visiblePosition >= 0) {
        //得到要更新的item的view
        View view = listView.getChildAt(index - visiblePosition);

        // 更新界面（示例参考）
        // TextView nameView = ViewLess.$(view, R.id.name);
        // nameView.setText("update " + index);
        // 更新列表数据（示例参考）
        // list.get(index).setName("Update " + index);
    }
}
```

强调一下，最后那个列表数据别忘记更新， 不然数据源不变，一滚动可能又还原了。

## 系统日志中几个重要的TAG

```
// 查看Activity跳转
adb logcat -v time | grep ActivityManager
// 查看崩溃信息
adb logcat -v time | grep AndroidRuntime
// 查看Dalvik信息，比如GC
adb logcat -v time | grep "D\/Dalvik"
// 查看art信息，比如GC
adb logcat -v time | grep "I\/art"
```

## 一行居中，多行居左的TextView

这个一般用于提示信息对话框，如果文字是一行就居中，多行就居左。

在TextView外套一层wrap_content的ViewGroup即可简单实现：

```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <!-- 套一层wrap_content的ViewGroup -->
    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/hello_world" />
    </LinearLayout>
</RelativeLayout>
```

setCompoundDrawablesWithIntrinsicBounds()

网上一大堆setCompoundDrawables()方法无效不显示的问题，然后解决方法是setBounds，需要计算大小…
不用这么麻烦，用setCompoundDrawablesWithIntrinsicBounds()这个方法最简单！

## 计算程序运行时间

为了计算一段代码运行时间，一般的做法是，在代码的前面加个startTime，在代码的后面把当前时间减去startTime，这个时间差就是运行时间。

这里提供一种写起来更方便的方法，完全无时间逻辑，只是加一个打印log就够了。

```
// 测试setContentView()的时间
Log.d("TAG", "Start");
setContentView(R.layout.activity_http);
Log.d("TAG", "End");
```
没有计算时间的逻辑，这能测出来？

把日志过滤出来，运行命令“adb logcat -v time | grep TAG”：

```
03-18 14:47:25.477 D/TAG     (14600): Start
03-18 14:47:25.478 D/TAG     (14600): End
```
通过`-v time`参数，可以比较日志左边的时间来算出中间的代码运行的时间。

## JAVA引用类型一览表

对象引用：强引用 > 软引用 > 弱引用 > 虚引用。

|引用类型|回收时机|用途|生存时间|
|:--|:--|:--|:--|
|强引用|从来不会|对象的一般状态|JVM停止运行时终止|
|软引用|在内存不足时|对象缓存|内存不足时终止|
|弱引用|在垃圾回收时|对象缓存|GC运行后终止|
|虚引用|在垃圾回收时|对象跟踪|GC运行后终止|

## Context使用场景

为了防止Activity，Service等这样的Context泄漏于一些生命周期更长的对象，可以使用生命周期更长的ApplicationContext，但是不是所有的Context的都能替换为`ApplicationContext`,这是网上流传的一份表格：

||Application|Activity|Service|ContentProvider|BroadcastReceiver|
|:--|:--|:--|:--|:--|:--|
|Show Dialog|否|是|否|否|否|
|Start Activity	|否|是|否|否|否|
|Layout Inflation|否|是|否|否|否|
|Start Service|是|是	|是|是|是|
|Bind Service|是|是|是|是|否|
|Send Broadcast|是|是|是|是|是|
|Regist BroadcastReceiver|是|是|是|是|否|
|Load Resource Value|是|是|是|是|是|

## 图片缓存大小

现在很多图片库需要给图片设置一个最大缓存，但是这个值设置多少合适呢？

高端机和低端机的配置显然应该不同，可以考虑设置一个动态值。

建议设置为应用可用内存的1/8:

```
int memoryCache = (int) (Runtime.getRuntime().maxMemory() / 8);
```

## 系统内置的一些工具类

在AOSP源码全局搜了一下包含Util关键字的类，整理出这个列表供大家参考：

```
// 系统
./android/database/DatabaseUtils.java
./android/transition/TransitionUtils.java
./android/view/animation/AnimationUtils.java
./android/view/ViewAnimationUtils.java
./android/webkit/URLUtil.java
./android/bluetooth/le/BluetoothLeUtils.java
./android/gesture/GestureUtils.java
./android/text/TextUtils.java
./android/text/format/DateUtils.java
./android/os/FileUtils.java
./android/os/CommonTimeUtils.java
./android/net/NetworkUtils.java
./android/util/MathUtils.java
./android/util/TimeUtils.java
./android/util/ExceptionUtils.java
./android/util/DebugUtils.java
./android/drm/DrmUtils.java
./android/media/ThumbnailUtils.java
./android/media/ImageUtils.java
./android/media/Utils.java
./android/opengl/GLUtils.java
./android/opengl/ETC1Util.java
./android/telephony/PhoneNumberUtils.java
// 设计和支持库
./design/src/android/support/design/widget/ViewGroupUtils.java
./design/src/android/support/design/widget/ThemeUtils.java
./design/src/android/support/design/widget/ViewUtils.java
./design/lollipop/android/support/design/widget/ViewUtilsLollipop.java
./design/base/android/support/design/widget/AnimationUtils.java
./design/base/android/support/design/widget/MathUtils.java
./design/honeycomb/android/support/design/widget/ViewGroupUtilsHoneycomb.java
./v7/recyclerview/src/android/support/v7/widget/helper/ItemTouchUIUtil.java
./v7/recyclerview/src/android/support/v7/widget/helper/ItemTouchUIUtilImpl.java
./v7/recyclerview/src/android/support/v7/util/MessageThreadUtil.java
./v7/recyclerview/src/android/support/v7/util/AsyncListUtil.java
./v7/recyclerview/src/android/support/v7/util/ThreadUtil.java
./v7/recyclerview/tests/src/android/support/v7/widget/AsyncListUtilLayoutTest.java
./v7/recyclerview/tests/src/android/support/v7/util/AsyncListUtilTest.java
./v7/recyclerview/tests/src/android/support/v7/util/ThreadUtilTest.java
./v7/appcompat/src/android/support/v7/graphics/drawable/DrawableUtils.java
./v7/appcompat/src/android/support/v7/widget/DrawableUtils.java
./v7/appcompat/src/android/support/v7/widget/ThemeUtils.java
./v7/appcompat/src/android/support/v7/widget/ViewUtils.java
./v4/tests/java/android/support/v4/graphics/ColorUtilsTest.java
./v4/jellybean-mr1/android/support/v4/text/TextUtilsCompatJellybeanMr1.java
./v4/jellybean/android/support/v4/app/BundleUtil.java
./v4/jellybean/android/support/v4/app/NavUtilsJB.java
./v4/java/android/support/v4/app/NavUtils.java
./v4/java/android/support/v4/database/DatabaseUtilsCompat.java
./v4/java/android/support/v4/graphics/ColorUtils.java
./v4/java/android/support/v4/text/TextUtilsCompat.java
./v4/java/android/support/v4/util/TimeUtils.java
./v4/java/android/support/v4/util/DebugUtils.java
./v4/java/android/support/v4/content/res/TypedArrayUtils.java
```

这么多工具类，一定可以找到对你有用的。

## ClipPadding

这个不多说，ListView的ClipPadding设为false，就能为ListView设置各种padding而不会出现丑陋的滑动“禁区”了。

## 强大的dumpsys

dumpsys可以查看系统服务和状态，非常强大，可通过如下查看所有支持的子命令：

```
dumpsys | grep "DUMP OF SERVICE"
```

这里列举几个稍微常用的：

|子命令|备注|
|:--|:--|
|activity|显示所有的activities的信息|
|cpuinfo|显示CPU信息|
|window|显示键盘，窗口和它们的关系|
|meminfo|内存信息（meminfo $package_name or $pid 使用包名或者进程id显示内存信息）|
|alarm|显示Alarm信息|
|statusbar|显示状态栏相关的信息（找出广告通知属于哪个应用）|
|usagestats|每个界面启动的时间|

## bugreport命令

很多人都用过`adb logcat`，但是如果想要更详细的信息，logcat则无能为力。

所以大多数手机厂商测试更多的是用`adb bugreport`来抓log给开发人员分析。

```
// 除了log，还包括启动后的系统状态,包括进程列表，内存信息，VM信息等等
// 而且不像logcat是一直打印的，bugreport命令输出到当前时间就停止结束了。
adb bugreport > main.log
```

## dpi文件夹的换算比例

之前的ldpi基本可以抛弃了，主流的dpi已经从很早之前的mdip转移到了xhdpi了，特别提醒。

|PPI|RESOLUTION|DP|PX|
|:--|:--|:--|:--|
|mdpi(160dp)|320P|1|1|
|hdpi(240dp)|480P|1|1.5|
|xhdpi(320dp)|720P|1|2|
|xxhdpi(480dpi)|1080P|1|3|

## 更新媒体库文件

以前做ROM的时候经常碰到一些第三方软件（某音乐APP）下载了新文件或删除文件之后，但是媒体库并没有更新，因为这个是需要第三方软件主动触发。

```
// 通知媒体库更新单个文件状态
Uri fileUri = Uri.fromFile(file);
sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE,fileUri));
```

媒体库会在手机启动，SD卡插拔的情况下进行全盘扫描，不是实时的而且代价比较大，所以单个文件的刷新很有必要。

## Monkey参数

大家都知道，跑monkey的参数设置有一些要注意的地方，比如太快了不行不切实际，太慢了也不行等等，这里给出一个参考：

```
adb shell monkey -p <packageName> -s 1000 --ignore-crashes --ignore-timeouts --ignore-security-exceptions  --pct-trackball 0 --pct-nav 0 --pct-majornav 0 --pct-anyevent 0  -v --throttle 300 1200000000
```

一边跑monkey，一边抓log吧。

## 小结

无论是大经验还是小经验，有用就是好经验。