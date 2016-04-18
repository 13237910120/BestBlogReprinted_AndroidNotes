# Android开发中，那些让你相见恨晚的方法、类或接口

来源:[liukun的个人博客](http://liukun.engineer/2016/04/11/Android%E5%BC%80%E5%8F%91%E4%B8%AD%EF%BC%8C%E9%82%A3%E4%BA%9B%E8%AE%A9%E4%BD%A0%E7%9B%B8%E8%A7%81%E6%81%A8%E6%99%9A%E7%9A%84%E6%96%B9%E6%B3%95%E3%80%81%E7%B1%BB%E6%88%96%E6%8E%A5%E5%8F%A3/)

>PS:本文类容来自我在[知乎上对Android开发中，有哪些让你觉得相见恨晚的方法、类或接口？](https://www.zhihu.com/question/33636939/answer/57239990?group_id=612750833369153536)这一问题的回答，目前就总结这些，日后若有新的发现，随时补充。欢淫点赞。

* getParent().requestDisallowInterceptTouchEvent(true);剥夺父view 对touch 事件的处理权，谁用谁知道。

* ArgbEvaluator.evaluate(float fraction, Object startValue, Object endValue); 用于根据一个起始颜色值和一个结束颜色值以及一个偏移量生成一个新的颜色，分分钟实现类似于微信底部栏滑动颜色渐变。

* Canvas中clipRect、clipPath和clipRegion 剪切区域的API。

> 第3个有个坑，就是有些clip貌似不支持硬件加速…(也有可能我测试机不多，安卓api demo里面的canvas clip那个Activity是在Activity注册的时候加了使用软件绘图的属性，当年照着demo写clipPath一直没有效果…后来发现要关闭硬件加速才行，被这个坑了好久… 还有就是刚出的design包里的FAB，手动设置成软件绘图在4.x下才会正常显示阴影…)

* Bitmap.extractAlpha ();返回一个新的Bitmap，capture原始图片的alpha 值。有的时候我们需要动态的修改一个元素的背景图片又不希望使用多张图片的时候，通过这个方法，结合Canvas 和Paint 可以动态的修改一个纯色Bitmap的颜色。

* HandlerThread，代替不停new Thread 开子线程的重复体力写法。

* IntentService,一个可以干完活后自己去死且不需要我们去管理子线程的Service。

* Palette，5.0加入的可以提取一个Bitmap 中突出颜色的类，结合上面的Bitmap.extractAlpha，你懂的。

* Executors. newSingleThreadExecutor();这个是java 的，之前不知道它，自己花很大功夫去研究了单线程顺序执行的任务队列。。

* android:animateLayoutChanges=”true”，LinearLayout中添加View 的动画的办法，支持通过setLayoutTransition()自定义动画。

* ViewDragHelper，自定义一个子View可拖拽的ViewGroup 时，处理各种事件很累吧，嗯? what the fuck!!

* GradientDrawable，之前接手公司的项目，发现有个阴影效果还不错，以为是切的图片，一看代码，什么鬼= =！

* AsyncQueryHandler，如果做系统工具类的开发，比如联系人短信辅助工具等，肯定免不了和ContentProvider打交道，如果数据量不是很大的情况下，随便搞，如果数据量大的情况下，了解下这个类是很有必要的，需要注意的是，这玩意儿吃异常..

* ViewFlipper，实现多个view的切换(循环)，可自定义动画效果，且可针对单个切换指定动画。

* 有朋友提到了在自定义View时有些方法在开启硬件加速的时候没有效果的问题，在API16之后确实有很多方法不支持硬件加速，通常我们关闭硬件加速都是在清单文件中通过，其实android也提供了针对特定View关闭硬件加速的方法,调用View.setLayerType(View.LAYER_TYPE_SOFTWARE, null);即可。

* android util包中的Pair类，可以方便的用来存储一”组”数据。注意不是key value。

* PointF，graphics包中的一个类，我们经常见到在处理Touch事件的时候分别定义一个downX，一个downY用来存储一个坐标，如果坐标少还好，如果要记录的坐标过多那代码就不好看了。用PointF(float x, float y);来描述一个坐标点会清楚很多。

* StateListDrawable，定义Selector通常的办法都是xml文件，但是有的时候我们的图片资源可能是从服务器动态获取的，比如很多app所谓的皮肤，这种时候就只能通StateListDrawable
来完成了，各种addState即可。

* android:descendantFocusability，ListView的item中CheckBox等元素抢焦点导致item点击事件无法响应时，除了给对应的元素设置 focusable,更简单的是在item根布局加上android:descendantFocusability=”blocksDescendants”

* android:duplicateParentState=”true”，让子View跟随其Parent的状态，如pressed等。常见的使用场景是某些时候一个按钮很小，我们想要扩大其点击区域的时候通常会再给其包裹一层布局，将点击事件写到Parent上，这时候如果希望被包裹按钮的点击效果对应的Selector继续生效的话，这时候duplicateParentState就派上用场了。

* includeFontPadding=”false”，TextView默认上下是有一定的padding的，有时候我们可能不需要上下这部分留白，加上它即可。

* Messenger，面试的时候通常都会被问到进程间通信，一般情况下大家都是开始背书，AIDL巴拉巴拉。。有一天在鸿神的博客看到这个，嗯，如他所说，又可以装一下了。

* TextView.setError();用于验证用户输入。

* ViewConfiguration.getScaledTouchSlop();触发移动事件的最小距离，自定义View处理touch事件的时候，有的时候需要判断用户是否真的存在movie，系统提供了这样的方法。

* ValueAnimator.reverse(); 顺畅的取消动画效果。

* ViewStub，有的时候一块区域需要根据情况显示不同的布局，通常我们都会通过setVisibility的方法来显示和隐藏不同的布局，但是这样默认是全部加载的，用ViewStub可以更好的提升性能。

* onTrimMemory，在Activity中重写此方法，会在内存紧张的时候回调（支持多个级别），便于我们主动的进行资源释放，避免OOM。

* EditTxt.setImeOptions， 使用EditText弹出软键盘时，修改回车键的显示内容(一直很讨厌用回车键来交互，所以之前一直不知道这玩意儿)

* TextView.setCompoundDrawablePadding，代码设置TextView的drawable padding。

* ImageSwitcher，可以用来做图片切换的一个类，类似于幻灯片。

* WeakHashMap，直接使用HashMap有时候会带来内存溢出的风险，使用WaekHashMap实例化Map。当使用者不再有对象引用的时候，WeakHashMap将自动被移除对应Key值的对象。


-----------

## 评论内容1

作者：StephenLee
链接：https://www.zhihu.com/question/33636939/answer/57171337
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

抛砖引玉一下，怎么都没什么人？

* 1、Throwable接口中的getStackTrace()方法（或者Thread类的getStackTrace()方法），根据这个方法可以得到函数的逐层调用地址，其返回值为StackTraceElement[]；

* 2、StackTraceElement类，其中四个方法getClassName()，getFileName()，getLineNumber()，getMethodName()在调试程序打印Log时非常有用；

* 3、UncaughtExceptionHandler接口，再好的代码异常难免，利用此接口可以对未捕获的异常善后；
使用参见：Android使用UncaughtExceptionHandler捕获全局异常

* 4、Resources类中的getIdentifier(name, defType, defPackage)方法，根据资源名称获取其ID，做UI时经常用到；

* 5、View中的isShown()方法，以前都是用view.getVisibility() == View.VISIBLE来判断的(╯□╰)；（谢评论提醒，这里面其实有一个坑：【android】view.isShown ()的用法）

* 6、Arrays类中的一系列关于数组操作的工具方法：binarySearch()，asList()，equals()，sort()，toString()，copyOfRange()等；
Collections类中的一系列关于集合操作的工具方法：sort()，reverse()等；

* 7、android.text.format.Formatter类中formatFileSize(Context, long)方法，用来格式化文件Size（B → KB → MB → GB）；

* 8、android.media.ThumbnailUtils类，用来获取媒体（图片、视频）缩略图；

* 9、String类中的format(String, Object...)方法，用来格式化strings.xml中的字符串（多谢 @droider An 提示：Context类中getString(int, Object... )方法用起来更加方便）；

* 10、View类中的三个方法：callOnClick()，performClick()，performLongClick()，用于触发View的点击事件；

* 11、TextUtils类中的isEmpty(CharSequence)方法，判断字符串是否为null或""；

* 12、TextView类中的append(CharSequence)方法，添加文本。一些特殊文本直接用+连接会变成String；

* 13、View类中的getDrawingCache()等一系列方法，目前只知道可以用来截图；

* 14、DecimalFormat类，用于字串格式化包括指定位数、百分数、科学计数法等；

* 15、System类中的arraycopy(src, srcPos, dest, destPos, length)方法，用来copy数组；

* 16、Fragment类中的onHiddenChanged(boolean)方法，使用FragmentTransaction中的hide()，show()时貌似Fragment的其它生命周期方法都不会被调用，太坑爹！

* 17、Activity类中的onWindowFocusChanged(boolean)，onNewIntent(intent)等回调方法；

* 18、View类中的getLocationInWindow(int[])方法和getLocationOnScreen(int[])方法，获取View在窗口/屏幕中的位置；

* 19、TextView类中的setTransformationMethod(TransformationMethod)方法，可用来实现“显示密码”功能；

* 20、TextWatcher接口，用来监听文本输入框内容的改变，可用来实现一系列具有特殊功能的文本输入框；

* 21、View类中的setSelected(boolean)方法结合android:state_selected=""用来实现图片选中效果；

* 22、Surface设置透明：SurfaceView.setZOrderOnTop(true);
SurfaceView.getHolder().setFormat(PixelFormat.TRANSLUCENT);但是会挡住其它控件；

* 23、ListView或GridView类中的setFastScrollEnabled(boolean)方法，用来设置快速滚动滑块是否可见，当然前提是item够多；

* 24、PageTransformer接口，用来自定义ViewPager页面切换动画，用setPageTransformer(boolean, PageTransformer)方法来进行设置；

* 25、apache提供的一系列jar包：commons-lang.jar，commons-collections.jar，commons-beanutils.jar等，里面很多方法可能是你曾经用几十几百行代码实现过的，但是执行效率或许要差很多，比如：ArrayUtils，StringUtils……；

* 26、AndroidTestCase类，Android单元测试，在AndroidStudio中使用非常方便；

* 27、TextView类的setKeyListener(KeyListener)方法；
其中DigitsKeyListener类，使用getInstance(String accepted)方法即可指定EditText可输入字符集；

* 28、ActivityLifecycleCallbacks接口，用于在Application类中监听各Activity的状态变化；

* 29、Context类中的createPackageContext(packageName, flags)方法，可用来获取指定包名应用程序的Context对象。


-------

## 评论内容2

> 作者：Rocko<br/>
> 链接：https://www.zhihu.com/question/33636939/answer/57297329<br/>
> 来源：知乎<br/>
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。<br/>

首先呼应题目，Log.wtf()

### Part 1：
* Activity.startActivities() 常用于在应用程序中间启动其他的Activity。
* TextUtils.isEmpty() 简单的工具类,用于检测是否为空。
* Html.fromHtml() 用于生成一个Html,参数可以是一个字符串.个人认为它不是很快,所以我不怎么经常去用.（我说不经常用它是为了重点突出这句话：请多手动构建 Spannable 来替换 Html.fromHtml），但是它对渲染从 web 上获取的文字还是很不错的。
* TextView.setError() 在验证用户输入的时候很棒。
* Build.VERSION_CODES 这个标明了当前的版本号,在处理兼容性问题的时候经常会用到.点进去可以看到各个版本的不同特性。
* Log.getStackTraceString() 方便的日志类工具,方法Log.v()、Log.d()、Log.i()、Log.w()和Log.e()都是将信息打印到LogCat中，有时候需要将出错的信息插入到数据库或一个自定义的日志文件中，那么这种情况就需要将出错的信息以字符串的形式返回来，也就是使用static String getStackTraceString(Throwable tr)方法的时候。
* LayoutInflater.from() 顾名思义,用于Inflate一个layout,参数是layout的id.这个经常写Adapter的人会用的比较多。
* ViewConfiguration.getScaledTouchSlop() 使用 ViewConfiguration 中提供的值以保证所有触摸的交互都是统一的。这个方法获取的值表示:用户的手滑动这个距离后,才判定为正在进行滑动.当然这个值也可以自己来决定.但是为了一致性,还是使用标准的值较好。
* PhoneNumberUtils.convertKeypadLettersToDigits 顾名思义.将字母转换为数字,类似于T9输入法,
* Context.getCacheDir() 获取缓存数据文件夹的路径,很简单但是知道的人不多,这个路径通常在SD卡上(这里的SD卡指的是广义上的SD卡,包括外部存储和内部存储)Adnroid/data/您的应用程序包名/cache/ 下面.测试的时候,可以去这里面看是否缓存成功.缓存在这里的好处是:不用自己再去手动创建文件夹,不用担心用户把自己创建的文件夹删掉,在应用程序卸载的时候,这里会被清空,使用第三方的清理工具的时候,这里也会被清空。
* ArgbEvaluator 用于处理颜色的渐变。就像 Chris Banes 说的一样，这个类会进行很多自动装箱的操作，所以最好还是去掉它的逻辑自己去实现它。这个没用过,不明其所以然,回头再补充.
* ContextThemeWrapper 方便在运行的时候修改主题。
* Space space是Android 4.0中新增的一个控件，它实际上可以用来分隔不同的控件，其中形成一个空白的区域.这是一个轻量级的视图组件，它可以跳过Draw，对于需要占位符的任何场景来说都是很棒的。
* ValueAnimator.reverse() 这个方法可以很顺利地取消正在运行的动画。

---------------------------------------------------------------------------------------------------------------------------------
### Part 2:
* DateUtils.formatDateTime() 用来进行区域格式化工作，输出格式化和本地化的时间或者日期。
* AlarmManager.setInexactRepeating 通过闹铃分组的方式省电，即使你只调用了一个闹钟，这也是一个好的选择，（可以确保在使用完毕时自动调用 AlarmManager.cancel ()。原文说的比较抽象，这里详细说一下:setInexactRepeating指的是设置非准确闹钟，使用方法:alarmManager.setInexactRepeating(AlarmManager.RTC， startTime，intervalL， pendingIntent)，非准确闹钟只能保证大致的时间间隔，但是不一定准确，可能出现设置间隔为30分钟，但是实际上一次间隔20分钟，另一次间隔40分钟。它的最大的好处是可以合并闹钟事件，比如间隔设置每30分钟一次，不唤醒休眠，在休眠8小时后已经积累了16个闹钟事件，而在手机被唤醒的时候，非准时闹钟可以把16个事件合并为一个， 所以这么看来，非准时闹钟一般来说比较节约能源。
* Formatter.formatFileSize() 一个区域化的文件大小格式化工具。通俗来说就是把大小转换为MB，G，KB之类的字符串。
* ActionBar.hide()/.show() 顾名思义，隐藏和显示ActionBar，可以优雅地在全屏和带Actionbar之间转换。
* Linkify.addLinks() 在Text上添加链接。很实用。
* StaticLayout 在自定义 View 中渲染文字的时候很实用。
* Activity.onBackPressed() 很方便的管理back键的方法，有时候需要自己控制返回键的事件的时候，可以重写一下。比如加入 “点两下back键退出” 功能。
* GestureDetector 用来监听和相应对应的手势事件，比如点击，长按，慢滑动，快滑动，用起来很简单，比你自己实现要方便许多。
* DrawFilter 可以让你在不调用onDrew方法的情况下，操作canvas，比了个如，你可以在创建自定义 View 的时候设置一个 DrawFilter，给父 View 里面的所有 View 设置反别名。
* ActivityManager.getMemoryClass() 告诉你你的机器还有多少内存，在计算缓存大小的时候会比较有用。
* ViewStub 它是一个初始化不做任何事情的 View，但是之后可以载入一个布局文件。在慢加载 View 中很适合做占位符。唯一的缺点就是不支持标签，所以如果你不太小心的话，可能会在视图结构中加入不需要的嵌套。
* SystemClock.sleep() 这个方法在保证一定时间的 sleep 时很方便，通常我用来进行 debug 和模拟网络延时。
* DisplayMetrics.density 这个方法你可以获取设备像素密度，大部分时候最好让系统来自动进行缩放资源之类的操作，但是有时候控制的效果会更好一些.(尤其是在自定义View的时候)。
* Pair.create() 方便构建类和构造器的方法。

---------------------------------------------------------------------------------------------------------------------------------
### Part 3:
* UrlQuerySanitizer——使用这个工具可以方便对 URL 进行检查。
* Fragment.setArguments——因为在构建 Fragment 的时候不能加参数，所以这是个很好的东西，可以在创建 Fragment 之前设置参数（即使在 configuration 改变的时候仍然会导致销毁/重建）。
* DialogFragment.setShowsDialog ()—— 这是一个很巧妙的方式，DialogFragment 可以作为正常的 Fragment 显示！这里可以让 Fragment 承担双重任务。我通常在创建 Fragment 的时候把 onCreateView ()和 onCreateDialog ()都加上，就可以创建一个具有双重目的的 Fragment。
* FragmentManager.enableDebugLogging ()——在需要观察 Fragment 状态的时候会有帮助。
* LocalBroadcastManager——这个会比全局的 broadcast 更加安全，简单，快速。像 otto 这样的 Event buses 机制对你的应用场景更加有用。
* PhoneNumberUtils.formatNumber ()——顾名思义，这是对数字进行格式化操作的时候用的。
* Region.op()——我发现在对比两个渲染之前的区域的时候很实用，如果你有两条路径，那么怎么知道它们是不是会重叠呢？使用这个方法就可以做到。
* Application.registerActivityLifecycleCallbacks——虽然缺少官方文档解释，不过我想它就是注册 Activity 的生命周期的一些回调方法（顾名思义），就是一个方便的工具。
* versionNameSuffix——这个 gradle 设置可以让你在基于不同构建类型的 manifest 中修改版本名这个属性，例如，如果需要在在 debug 版本中以”-SNAPSHOT”结尾，那么就可以轻松的看出当前是 debug 版还是 release 版。
* CursorJoiner——如果你是只使用一个数据库的话，使用 SQL 中的 join 就可以了，但是如果收到的数据是来自两个独立的 ContentProvider，那么 CursorJoiner 就很实用了。
* Genymotion——一个非常快的 Android 模拟器，本人一直在用。
* -nodpi——在没有特别定义的情况下，很多修饰符(-mdpi,-hdpi,-xdpi等等)都会默认自动缩放 assets/dimensions，有时候我们需要保持显示一致，这种情况下就可以使用 -nodpi。
* BroadcastRecevier.setDebugUnregister ()——又一个方便的调试工具。
* Activity.recreate ()——强制让 Activity 重建。
* PackageManager.checkSignatures ()——如果同时安装了两个 app 的话，可以用这个方法检查。如果不进行签名检查的话，其他人可以轻易通过使用一样的包名来模仿你的 app。

---------------------------------------------------------------------------------------------------------------------------------
### Part 4:
* Activity.isChangingConfigurations ()——如果在 Activity 中 configuration 会经常改变的话，使用这个方法就可以不用手动做保存状态的工作了。
* SearchRecentSuggestionsProvider——可以创建最近提示效果的 provider，是一个简单快速的方法。
* ViewTreeObserver——这是一个很棒的工具。可以进入到 VIew 里面，并监控 View 结构的各种状态，通常我都用来做 View 的测量操作（自定义视图中经常用到）。
* org.gradle.daemon=true——这句话可以帮助减少 Gradle 构建的时间，仅在命令行编译的时候用到，因为 Android Studio 已经这样使用了。
* DatabaseUtils——一个包含各种数据库操作的使用工具。
* android:weightSum (LinearLayout)——如果想使用 layout weights，但是却不想填充整个 LinearLayout 的话，就可以用 weightSum 来定义总的 weight 大小。
* android:duplicateParentState (View)——此方法可以使得子 View 可以复制父 View 的状态。比如如果一个 ViewGroup 是可点击的，那么可以用这个方法在它被点击的时候让它的子 View 都改变状态。
* android:clipChildren (ViewGroup)——如果此属性设置为不可用，那么 ViewGroup 的子 View 在绘制的时候会超出它的范围，在做动画的时候需要用到。
* android:fillViewport (ScrollView)——在这片文章中有详细介绍文章链接，可以解决在 ScrollView 中当内容不足的时候填不满屏幕的问题。
* android:tileMode (BitmapDrawable)——可以指定图片使用重复填充的模式。
* android:enterFadeDuration/android:exitFadeDuration (Drawables)——此属性在 Drawable 具有多种状态的时候，可以定义它展示前的淡入淡出效果。
* android:scaleType (ImageView)——定义在 ImageView 中怎么缩放/剪裁图片，一般用的比较多的是“centerCrop”和“centerInside”。
* Merge——此标签可以在另一个布局文件中包含别的布局文件，而不用再新建一个 ViewGroup，对于自定义 ViewGroup 的时候也需要用到；可以通过载入一个带有标签的布局文件来自动定义它的子部件。
* AtomicFile——通过使用备份文件进行文件的原子化操作。这个知识点之前我也写过，不过最好还是有出一个官方的版本比较好。

---------------------------------------------------------------------------------------------------------------------------------
### Part 5:
* ViewDragHelper ——视图拖动是一个比较复杂的问题。这个类可以帮助解决不少问题。如果你需要一个例子，DrawerLayout就是利用它实现扫滑。Flavient Laurent 还写了一些关于这方面的优秀文章。
* PopupWindow——Android到处都在使用PopupWindow ，甚至你都没有意识到（标题导航条ActionBar，自动补全AutoComplete，编辑框错误提醒Edittext Errors）。这个类是创建浮层内容的主要方法。
* Actionbar.getThemrContext()——导航栏的主题化是很复杂的（不同于Activity其他部分的主题化）。你可以得到一个上下文（Context），用这个上下文创建的自定义组件可以得到正确的主题。
* ThumbnailUtils——帮助创建缩略图。通常我都是用现有的图片加载库（比如，Picasso 或者 Volley），不过这个ThumbnaiUtils可以创建视频缩略图。译者注：该API从V8才开始支持。
* Context.getExternalFilesDir()———— 申请了SD卡写权限后，你可以在SD的任何地方写数据，把你的数据写在设计好的合适位置会更加有礼貌。这样数据可以及时被清理，也会有更好的用户体验。此外，Android 4.0 Kitkat中在这个文件夹下写数据是不需要权限的，每个用户有自己的独立的数据存储路径。译者注：该API从V8才开始支持。
* SparseArray——Map的高效优化版本。推荐了解姐妹类SparseBooleanArray、SparseIntArray和SparseLongArray。
* PackageManager.setComponentEnabledSetting()——可以用来启动或者禁用程序清单中的组件。对于关闭不需要的功能组件是非常赞的，比如关掉一个当前不用的广播接收器。
* SQLiteDatabase.yieldIfContendedSafely()——让你暂时停止一个数据库事务， 这样你可以就不会占用太多的系统资源。
* Environment.getExternalStoragePublicDirectory()——还是那句话，用户期望在SD卡上得到统一的用户体验。用这个方法可以获得在用户设备上放置指定类型文件（音乐、图片等）的正确目录。
* View.generateViewId()——每次我都想要推荐动态生成控件的ID。需要注意的是，不要和已经存在的控件ID或者其他已经生成的控件ID重复。
* ActivityManager.clearApplicationUserData()—— 一键清理你的app产生的用户数据，可能是做用户退出登录功能，有史以来最简单的方式了。
* Context.createConfigurationContext() ——自定义你的配置环境信息。我通常会遇到这样的问题：强制让一部分显示在某个特定的环境下（倒不是我一直这样瞎整，说来话长，你很难理解）。用这个实现起来可以稍微简单一点。
* ActivityOptions ——方便的定义两个Activity切换的动画。 使用ActivityOptionsCompat 可以很好解决旧版本的兼容问题。
* AdapterViewFlipper.fyiWillBeAdvancedByHostKThx()——仅仅因为很好玩，没有其他原因。在整个安卓开源项目中（AOSP the Android ——pen Source Project Android开放源代码项目）中还有其他很有意思的东西（比如
GRAVITY_DEATH_STAR_I）。不过，都不像这个这样，这个确实有用
* ViewParent.requestDisallowInterceptTouchEvent() ——Android系统触摸事件机制大多时候能够默认处理，不过有时候你需要使用这个方法来剥夺父级控件的控制权。


译文出自 [@Gracker](https://www.zhihu.com/people/dafca61abd3171ed5bf8b55ab023f7cf) 的博客，[Android Performance](https://link.zhihu.com/?target=http%3A//androidperformance.com/)：<br/>

Part1: [[译]Android小技巧(1)](https://link.zhihu.com/?target=http%3A//androidperformance.com/2014/05/28/android-tips-round-up-1/)<br/>
Part2: [[译]Android小技巧(2)](https://link.zhihu.com/?target=http%3A//androidperformance.com/2014/05/31/android-tips-round-up-2/)<br/>
Part3：[[译]Android小技巧(3)](https://link.zhihu.com/?target=http%3A//androidperformance.com/2015/03/15/android-tips-round-up-3/)<br/>
Part4: [[译]Android小技巧(4)](https://link.zhihu.com/?target=http%3A//blog.danlew.net/2014/05/12/android-tips-round-up-part-4/)<br/>
Part5: [[译]Android小技巧(5)](https://link.zhihu.com/?target=http%3A//androidperformance.com/2015/03/15/android-tips-round-up-5/)<br/>

原文出自 Dan Lew 的博客，有 5 篇，强烈推荐。

[Android Tips Round-Up, Part 1](https://link.zhihu.com/?target=http%3A//blog.danlew.net/2014/03/30/android-tips-round-up-part-1/)<br/>
[Android Tips Round-Up, Part 2](https://link.zhihu.com/?target=http%3A//blog.danlew.net/2014/04/14/android-tips-round-up-part-2/)<br/>
[Android Tips Round-Up, Part 3](https://link.zhihu.com/?target=http%3A//blog.danlew.net/2014/04/28/android-tips-round-up-part-3/)<br/>
[Android Tips Round-Up, Part 4](https://link.zhihu.com/?target=http%3A//blog.danlew.net/2014/05/12/android-tips-round-up-part-4/)<br/>
[Android Tips Round-Up, Part 5](https://link.zhihu.com/?target=http%3A//blog.danlew.net/2014/05/28/android-tips-round-up-part-5/)<br/>

最后做个福利广告 [zhengxiaopeng/android-dev-bookmarks · GitHub](https://link.zhihu.com/?target=https%3A//github.com/zhengxiaopeng/android-dev-bookmarks)

-------

### 补充

* 1、android:clipChildren 和 android:clipToPadding：clipToPadding就是说控件的绘制区域是否在padding里面的，true的情况下如果你设置了padding那么绘制的区域就往里 缩，clipChildren是指子控件是否超过padding区域，这两个属性默认是true的，所以在设置了padding情况下，默认滚动是在 padding内部的，要达到上面的效果主要把这两个属性设置了false那么这样子控件就能画到padding的区域了。使用场景如：ActionBar（透明）下显示Listview而第一项要在actionbar下。参见 [android:clipToPadding和android:clipChildren。](https://link.zhihu.com/?target=http%3A//www.alloyteam.com/2014/10/androidcliptopadding-he-androidclipchildren/)
* 2、Fragment 的 setUserVisibleHint 方法，可实现 fragment 对用户可见时才加载资源（延迟加载）。
* 3、自定义 View 时重写 hasOverlappingRendering 方法指定 View 是否有 Overlapping 的情况，提高渲染性能。
* 4、AutoScrollHelper，在可滚动视图中长按边缘实现滚动，[Android View.OnTouchListener 的子类](https://link.zhihu.com/?target=http%3A//zhengxiaopeng.com/2015/04/26/Android-View-OnTouchListener-%25E7%259A%2584%25E5%25AD%2590%25E7%25B1%25BB/)。
* 5、TouchSlop，系统所能识别出的被认为是最小的滑动距离，ViewConfiguration.get(context).getScaledTouchSlop()。
* 6、VelocityTracker，可用于 View 滑动事件速度跟踪。
* 7、AlphabetIndexer，字母索引辅助类。
* 8、Messenger，AIDL 实现的封装，比手写 AIDL 更方便。
* 9、ArrayMap，比 HashMap 更高的内存效率，但比 HashMap 慢，不适合有大量数据的场景。
* 10、Property，抽象类，封装出对象中的一个易变的属性值，使用场景如在使用属性动画时对动画属性的操作。
* 11、SortedList，v7 包中，见名知意。

---------

## 技术小黑屋，http://droidyue.com/

* HandlerThread 单一线程 + 任务队列 处理轻量的异步任务 详见 [详解 Android 中的 HandlerThread](https://link.zhihu.com/?target=http%3A//droidyue.com/blog/2015/11/08/make-use-of-handlerthread/)
* Proguard assumenosideeffects 编译器屏蔽日志 详见 [关于Android Log的一些思考](https://link.zhihu.com/?target=http%3A//droidyue.com/blog/2015/11/01/thinking-about-android-log/)
* [Android性能调优利器StrictMode](https://link.zhihu.com/?target=http%3A//droidyue.com/blog/2015/09/26/android-tuning-tool-strictmode/)
* Android中线程优先级控制 android.os.Process.setThreadPriority 详见 [剖析Android中进程与线程调度之nice](https://link.zhihu.com/?target=http%3A//droidyue.com/blog/2015/09/05/android-process-and-thread-schedule-nice/)
* Android Lint 一个静态分析工具 详见 [使用Android lint发现并解决高版本API问题](https://link.zhihu.com/?target=http%3A//droidyue.com/blog/2015/07/25/use-android-lint-to-find-higher-api-usage/)
* (更新) 使用UncaughtExceptionHandler 处理应用崩溃，收集信息甚至可以不显示崩溃对话框 [详细：Android处理崩溃的一些实践](https://link.zhihu.com/?target=http%3A//droidyue.com/blog/2015/12/06/practise-about-crash-in-android/)

-------

## 张明云，http://zmywly8866.github.io/

* 清除画布上的内容：canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR);
* 在自定义View的onDetachedFromWindow方法中清理与View相关的资源；
* Fragment在onAttach方法中接收回调：

```
@Override
public void onAttach(Activity activity) {
	super.onAttach(activity);
	try {
		mPageSelectedListener = (PageSelectedListener) activity;
		mMenuBtnOnclickListener = (MenuBtnOnClickListener) activity;
		mCommitBtnOnClickListener = (CommitBtnOnClickListener) activity;
	} catch (ClassCastException e) {
		throw new ClassCastException(activity.toString() 
				+ "must implements listener");
	}
}
```

* 使用[ClipDrawable](https://link.zhihu.com/?target=http%3A//developer.android.com/intl/zh-cn/reference/android/graphics/drawable/ClipDrawable.html)实现进度条功能；
* 自定义view中的getContext()，再也不需要专门创建一个mContext全局对象了；
* 自定义手写view的时候，在手指移动的过程中通过[MotionEvent | Android Developers](https://link.zhihu.com/?target=http%3A//developer.android.com/intl/zh-cn/reference/android/view/MotionEvent.html)对象的getHistorySize()获得缓存的历史点，绘制出来的曲线要平滑很多。
* 复写Activity的onUserLeaveHint方法，确保用户离开界面时能够立即暂停界面中的一些任务，关于onUserLeaveHint的更多作用可以谷歌：[android - Google 搜索](https://link.zhihu.com/?target=https%3A//www.google.com/search%3Fq%3Dandroid%2520%26gws_rd%3Dssl%23newwindow%3D1%26safe%3Dstrict%26q%3Dandroid%2BonUserLeaveHint)
* 值得借鉴的点击两次退出应用的实现：[Android关于双击退出应用的问题](https://link.zhihu.com/?target=http%3A//segmentfault.com/q/1010000002921663)

没那么麻烦，直接用toast的getView().getParent() 判断是不是空就ok了。API 16 测试通过

```
public class MainActivity extends Activity {


    private Toast toast;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        toast = Toast.makeText(getApplicationContext(), "确定退出？", 0);

    }
    public void onBackPressed() {
        quitToast();
    }
    /*
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        System.out.println(keyCode + "...." + event.getKeyCode());
        if(keyCode == KeyEvent.KEYCODE_BACK){
            quitToast();
        }
        return super.onKeyDown(keyCode, event);
    }
    */
    private void quitToast() {
        if(null == toast.getView().getParent()){
            toast.show();
        }else{
            System.exit(0);
        }
    }
}
```

------

## [Allan](https://www.zhihu.com/people/allan.chen)，Android 爱好者 -> 开发者

* android:animateLayoutChanges

一直以为 Lollipop Dialer 接通画面里面那些酷炫的动画（文字部分）是很复杂的做出来的，后来发现其实只有一行。
[视频 演示](https://dl.pushbulletusercontent.com/N75Bx03taJzFVjjLkMpzzyqGKT8m5PpH/cm_trltexxLMY48Gyilun07312015153119.mp4)

只需要加好 android:animateLayoutChanges="true" 然后 setVisibility 就可以了

----

## [林申](https://www.zhihu.com/question/33636939/answer/59913428):魅族科技（中国）有限公司 软件工程师

竟然没有人说这个：

[LocalBroadcastManager](https://link.zhihu.com/?target=http%3A//developer.android.com/intl/zh-cn/reference/android/support/v4/content/LocalBroadcastManager.html)

* You know that the data you are broadcasting won't leave your app, so don't need to worry about leaking private data.
* It is not possible for other applications to send these broadcasts to your app, so you don't need to worry about having security holes they can exploit.
* It is more efficient than sending a global broadcast through the system.

简单来说，就是 LocalBroadcastManager 可以在App的范围内发广播和收广播，不会被global的receiver收到，对数据比较敏感且不用共享的可以用这个。

果然 google support 包里面藏了太多的好东西。

-----

## [夏青](https://www.zhihu.com/people/xia-qing-19-57)，Android工程师 谣言粉碎机

说几个简单但是基础的吧

* selector用这个来做样式的多态真是没有太方便了，以前傻傻的自己分析事件来变换

* HierarchyViewer这个工具用来了解界面实现方式，找到每个view布局和对应id实在太方便了，还可以配合dumpsys命令用来调试

* ListView ViewHolder的使用，虽说现在RecyclerView已经把ViewHolder包含进去了，但是还是要说说这个方法对于复用的意义

* moveTaskToBack 看到很多论坛写模拟home按键的方式是用Intent.setAction(Intent.ACTION_MAIN)来实现所谓模拟home键的方式(实际上是调用launcher)，其实很多场景里面要这么做的主要目的就是为了让当前的APP隐藏而非退出，但是用Intent方法其实并不是那么优雅，刚才说了这个方法的实质是调用launcher，会导致所有应用全部转到后台，最近在做Android多任务相关工作，这么做对于系统开发者和其他应用造成不小的困扰，其实在activity层面调用moveTaskToBack就可以搞定了

* setSystemUiVisibility和setStatusBarColor要实现status bar的透明或者颜色用这两个接口就可以了，透明只需要设置SystemUiVisibility为View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_STABLE，颜色的话调用后面那个接口就可以，看到某某云音乐用系统id status_bar_height高度各种计算，自己绘制来实现对于status bar的染色也是最了

* loader和ContentObserver 实现数据于控制机制分离的非常好的结构，Android原生邮件系统使用了大量的这样的模式，来处理繁杂的邮件相关的信息内容，加载邮件服务器内容到数据库以及UI上的显示更新完全是两条路

-----

## [尹東](https://www.zhihu.com/people/ydcool)，INTP

TimingLogger,SDK自带打印时间戳工具，简直神器。

摘自官方文档：

```
A utility class to help log timings splits throughout a method call. Typical usage is:

TimingLogger timings = new TimingLogger(TAG, "methodA"); 
// ... do some work A ... timings.addSplit("work A");
// ... do some work B ... timings.addSplit("work B");
// ... do some work C ... timings.addSplit("work C"); timings.dumpToLog(); 

The dumpToLog call would add the following to the log:

D/TAG ( 3459): methodA: begin 
D/TAG ( 3459): methodA: 9 ms, work A 
D/TAG ( 3459): methodA: 1 ms, work B 
D/TAG ( 3459): methodA: 6 ms, work C 
D/TAG ( 3459): methodA: end, 16 ms 
```

原文 [http://www.ydcool.me/archives/11/](http://www.ydcool.me/archives/11/)

-----

## [master郑]()，做好产品

我也添一些，如有雷同纯属巧合~

* 1.通过 WindowManager.addView 在其他app界面添加一个view时，经常会无法显示，特别在miui，emui固件上，需要指定type为LayoutParams.TYPE_TOAST。

* 2.View.getLocationOnScreen(new int[])，获取view在屏幕上的位置

* 3.Paint.setXfermode(porterDuffXfermode)，在ApiDemo里面有专门的介绍，实现了穿透，叠加，覆盖等多种绘制效果，非常实用

* 4.直接获取当前系统壁纸的fd

```
IBinder binder = ServiceManager.getService("wallpaper");
IWallpaperManager wm = IWallpaperManager.Stub.asInterface(binder);
Bundle params = new Bundle();
ParcelFileDescriptor fd = wm.getWallpaper(stub, params);
```

直接获取当前系统壁纸的fd，避免壁纸过大造成oom问题。这种方式有适配问题，需注意。

* 5.通过View.getDrawingCache()可以获取截图，但是需要setDrawingCacheEnabled(true)频繁使用可能会oom，还有一种方法直接用canvas

```
Bitmap bm = Bitmap.createBitmap((int) (w * scale), (int) (h*scale), Bitmap.Config.ARGB_8888);
Canvas canvas = new Canvas();
canvas.setBitmap(bm);
View.draw(canvas);
return bm；
```

* 6.说到几个oom，顺带说下有一种偷懒又有效的解决办法，在manifest上加android:largeHeap="true"

* 7.用一个牛逼的来结尾，AccessibilityService。由于强大所以需要手动

----

## [咕咚](https://www.zhihu.com/people/maoruibin)，热爱代码，喜欢倒腾的程序员 …

AtomicInteger，一个提供原子操作的Integer的类。在Java语言中，++i和i++操作并不是线程安全的，在使用的时候，不可避免的会用到synchronized关键字。而AtomicInteger则通过一种线程安全的加减操作接口。这个类在AsyncTask中用到了。

----

## [田元](https://www.zhihu.com/people/tian-yuan-17-25)，算不上程序员。

* 1、android.support.design.widget.TextInputLayout，给EditText带个套吧⊙▽⊙
* 2、AndroidMainfest.xml activity的一些标签，比如android:windowSoftInputMode

```
<activity android:name=".Main" 
android:label="@string/app_name" 
android:windowSoftInputMode="stateHidden" > 
<intent-filter> 
<action android:name="android.intent.action.MAIN" /> 
<category android:name="android.intent.category.LAUNCHER" /> 
</intent-filter> 
</activity> 
```

activity launch后默认隐藏键盘，这在activity里面有EditText等元素又不想一开始就弹出软键盘的情况下有用，在此之前就知道android:name 和android.label这两个属性_(:3」∠)_

* 3、getSystemService函数，获取各种系统service,而且不用担心性能问题，都是直接返回各种manager。
* 4、Parcelable接口
原来受MFC等c++类库影响，比较习惯继承serialiabe接口这种方式，但后来知道了Parcelable的实现方式就喜欢上了。
* 5、android.support.v4.widget.DrawerLayout 原生大方的抽屉控件。
* 6、android.support.v7.widget.Toolbar 定制性极强的viewGroup

-----

## [peerless2012](https://www.zhihu.com/people/peerless2012)

* 5、View中的isShown()方法，以前都是用view.getVisibility() == View.VISIBLE来判断的

这个方法是个坑，并不是像描述的那样，如果view是个 ViewGroup，如果某个子view是不可见的，其他可见，这个方法照样会返回fasle


## [陈启超]

```
sendOrderedBroadcast (Intent intent, String receiverPermission, 
			BroadcastReceiver resultReceiver, Handler scheduler, 
			int initialCode, String initialData, Bundle initialExtras)
```

可以设置一个最终广播接收者 resultReceiver。即使有优先级高的广播接收者使用了 abortBroadcast() 拦截了广播，最终的广播接收着依然可以接收到广播。

参考：[Context | Android Developers](https://link.zhihu.com/?target=http%3A//developer.android.com/reference/android/content/Context.html%23sendOrderedBroadcast)(android.content.Intent, java.lang.String, android.content.BroadcastReceiver, android.os.Handler, int, java.lang.String, android.os.Bundle)

-----

## [李照](https://www.zhihu.com/people/li-zhao-91)

再补充一个监听主线程是否空闲，在View丢帧的情况下使用，效果立竿见影，在主线程中

Looper.myQueue().addIdleHandler(mIdleHandler)

------

## [千里山南](https://www.zhihu.com/people/qian-li-shan-nan)，Android Dever

* LinearLayout#setDividerDrawable
* View#inflate()
* URLUtil#isNetworkUrl及其他URL校验的方法
* Debug#startMethodTracing
* Debug#waitForDebug
* TimeUnit.SECONDS#sleep
* View.addOnAttachStateChangeListener

还有一些稍后补充

另外推荐一本小册子[《50 Android Hacks》摘要如下50 Android Hack 读书笔记](https://link.zhihu.com/?target=http%3A//my.oschina.net/u/189899/blog/378804)

-----

## 其他

* Throwable类中的getStackTrace()配合Arrays.toString()方法，可打印函数的逐层调用，方便找出引发异常的代码。

* [Cool Android Apis 整理（一）](http://oakzmm.com/2015/08/04/cool-Android-api/)
* [Cool Android Apis 整理（二）](http://oakzmm.com/2015/08/11/cool-Android-api-2/)
* [Cool Android Apis 整理（三）](http://oakzmm.com/2015/09/07/cool-android-api-3/)