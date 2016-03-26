# 聚划算android客户端1期教训总结

来源:[Liter's Blog](http://www.vmatianyu.cn/poly-effective-client-1-issues-lessons.html)

## 技术方面

1.一般性控件需要设置onclick事件才会有点击效果（selector）。

2.<item android:drawable=”@drawable/btn_ct_green” />要写在selector的最后才会有点击效果。

3.制作.9格式图片选最小图，否则默认大小撑大控件。

4.如果将一个对象的属性设置为static，那么就算对象实例被回收了，该属性也存在内存，生命周期为app的生命周期。

5.OOM：普通视图和listvew等大数据量展示视图的图片控制分开来。

6.OOM：listview等列表1秒真释放，把大数据量加载后不用的图片释放。

7.OOM：大图片使用前压缩。

8.OOM：减少大图。美工将全部规则图最小化，制作.9格式，以最小程度占用内存。
9.OOM：背景图、大图谨慎使用，不规则大图显式释放。
10.提前得知大小可使用，view.measure(-1, -1);但是view必须得有父view。
11.listview列表呈现多种样式，getViewTypeCount()方法返回全部样式的总数，getItemViewType(int pos)返回的值必须小于getViewTypeCount()，否则报错。
12.公共类、接口、基类设计要职责清晰易理解，最大量减少别人使用时的难度。

13.OOM：webview 内存溢出（OOM），重启一个新进程。

并且设置：需要在onPause时停止Timer，解决由于Timer在，导致WebCoreThread一直在，WebviewCache.db被锁定， 图文详情无法进入的问题webview.pauseTimers();

当Activity返回onResume时WebView.enablePlatformNotifications();webview .resumeTimers();

14.触控范围要作为一个规范来控制到开发的每一步中，有src属性的可以设置padding，没有的为了不失真，套一个layout，最小宽度48dp（9mm）。

15.请求带有时间戳请注意，yy-MM-dd hh:mm:ss是12小时制格式。yy-MM-dd HH:mm:ss是24小时制格式。差别巨大。

16.基础数据类型的封装类型是有预装缓存的，JVM给Byte缓存了-128~127的对象，Integer缓存了-128~127。所以Integer i =k，Integer j =k，，k = 127，i == j为true，k=128则为false。

17.逻辑条件加紧要慎重，放宽松更要慎重；放宽后考虑是否更引发副作用问题，聚划算将id=()、itemId=()，i()都抓下拉起详情，结果频繁无辜拉起。收紧后考虑是否会引起扩展问题

18.最后一刻加上的代码要严格的测试，很多时候就是最后‘以为’加上了‘无关紧要’的东西而导致崩溃掉。

19.Math.abs()取到的不一定是正数，Integer.minValue就是负值。

20.多线程请使用并发容器放置变量，不轻易认为机会少不会冲突，并发量一大什么都有可能。ThreadPool.shutdownNow()之后只是清除等待队列，然后等待活动线程执行完。

21.强转类型之前先先确定对象不为空。

22.android2.3以下版本listView.setDividerHeight()函数调用后，notifyDataSetChanged()便不能记住位置。可使用setSelection记住位置。

23.finish和startActivity位置很重要. 由A跳转向SingleTask的Activity B，A.finish的位置在startActivityB之前，退出B按home回到(home键退出或back键finishB)应用界面仍然是B，无论B是否是action.MAIN，overridePendingTransition需要在finish或者startActivity之后才有效。

24.区域事件拦截：比如只要ViewA获取点击事件而组织其父控件和其他子控件触发事件，可重写activity的dispatchTouchEvent()函数，调用ViewA.getHitRect(rect),初始化一个Rect，判断event的getX和getY如果在rec之内，拦截ACTION_DOWN返回true，其余ACTION调用ViewA. dispatchTouchEvent() 即可拦截事件。

25.一次有效触摸，当ACTION_DOWN返回ture时，其他事件也不会在得到响应。当event在rect之外时，可以通过event.setAction(MotionEvent.ACTION_DOWN);activity.onTouchEvent(event);来重新触发事件。

26.WebView：缓存与不缓存，很关键。尤其在活动、计时、含session界面。

27.WebView：当webview占用大量内存时，可以将WebView全部启动在另一个进程中。

28.WebView：当多个重定向干扰或不能后退到上一页时，不使用webview.goBack()，自己用栈Stack维护Url，其关键在于区分是否是重定向，目前采用java调用js获取、分析网页内容判断是否重定向，如果不是再将url放入stack，反之不入栈。

29.无线电波状态机：应用运行在前台考虑避免延迟阻塞，运行在后台关注电量浪费。优化网络连接：预取数据，批量传输与连接（包含携带、顺带其他数据），减少连接次数（规避高频心跳）。

30.当listview含有Header时，在onItemClick事件中请这样获取ItemObject：Object obj = parent.getAdapter().getItem(position); 先判空，再强转为需要的对象。

31.WebView： 注意对下载文件的支持；shouldOverrideUrlLoading返回false，会自动加载该页，返回true不会加载网页，需要自己处理（之前返回true，调用WebView.load(url)结果造成重定向网页不能回退的问题，自己花了很大代价才解决，直接返回false会自动加载）。

32.使用一个函数，尤其别人写的函数，不管怎么诚恳的承诺参数不会为null，请尽量做非空判断。除以一个变量之前，先确定其不为0.

33.如果程序自自动，或者后台耗流量，首先检测manifest中静态注册的广播，它会拉起程序。

34.findbugs结合使用ADT（16以后）自带的lint检测程序中的问题，lint可以检测出未使用的图片和更具android特性的问题。

35.View onMeasure之后，width不一定有值，如果设置了LayoutParagrams那么view.getLayoutParams().width将有设定值。

36.Gallery特性改善：一次触摸只切换一张图片：复写onFlying直接返回true；使触摸更加灵敏：复写onScroll 调用super.onScroll(e1, e2, distanceX * 1.5f, distanceY)，使distanceX 变大就更加灵敏。

37.Gallery视觉优化：setStaticTransformationsEnabled(true)之后，getChildStaticTransformation方法生效，默认方法会使图片alpha值改变变而视觉不清，复写可以利用Camera产生xyz和角度的改变，从而优化视觉体验，比如打造3D画廊。

38.可共用的对象属性用static来保持一份节省资源，每个实例或者对象单独享用的属性切记不要static。

39.改变一个类的私有属性：

Field field = ViewGroup.class.getDeclaredField(“hsl”);
field.setAccessible(true); field.set(listView, 0);

40.Which client is best?

Apache HTTP client has fewer bugs on Eclair and Froyo. It is the best choice for these releases.

For Gingerbread and better, HttpURLConnection is the best choice. Its simple API and small size makes it great fit for Android. Transparent compression and response caching reduce network use, improve speed and save battery. New applications should use HttpURLConnection; it is where we will be spending our energy going forward.


## 产品层面

1.特效的运用要考虑用户的心理趋向和感受，炫的效果很酷，可能会让人产生某种心理导向。

2.特效的运用要考虑将来功能的扩展，一味的渲染也不一定很好，可能会为后续功能带来难度。

3.一个产品能帮助用户解决至少一个问题，那解决问题会有主流程，引导要强化，比如按钮大，颜色显眼，字体稍大，放置位置移动少。

4.产品不能为照顾运营而偏离主题，或者严重影响美观和主要功能，聚划算妹纸儿们说广告banner曝光度不够，设计变为由轮转变为平铺，但是广告良莠不齐，视觉效果不好，并且主页list仅能显示1、2个商品，影响了主题功能。

## 项目方面

1.计划进度，前紧后慢，提前实施。

2.不拘泥与细节，不干扰整体进度。

3.个人有担当，能者多劳，最大程度保证进度。

4.有问题不能包着，有些项目问题不是一个人的问题，不是一人能担得住的。

5.宝不能压在责任无关人身上，别人的资源、承诺要充分利用，但是最终还是靠自己突破。

6.干一样活的人放在一起，沟通迅速，快速解决问题。

7.早上碰一下，清晰任务；晚上碰一下一下，检查进度。

8.bug先把基础性、功能性问题解决，适配、显示、兼容问题靠后。

9.不同网络机型下的跑测，不同网络环境下的测试提前入测。

10.写完一天代码，尽量坚持用findbugs查找潜在危险。

11.同一版本多次发布，或多个版本发布后，记录下的crash问题如果没有版本属性就很难定位,混淆map也对应不上。发板后立即封代码，保证bug日志中的bug发生行数和源码一致。

12.开发新功能、打包、发布过程规范化，规范减少问题发生概率。

13.日志规范：

* 1). 冗长、惯例日志打到Verbose级别，例如网络返回信息，经常携带大量的信息，会严                                    重干扰视觉；
* 2). 细粒度信息打到Debug级别，对出问题时调试有帮助作用，比如某个参数的值，我并    不总想知道是多少，但某些情况我就得看一下；
* 3). 粗粒度级别信息打到Info级别，比如应用业务主逻辑，运行中重要的过程或关键点；
* 4). 会出现 潜在错误的信息 或者 可以接受的错误 打到Warn级别，比如某个地方我想警告一下说这里是有问题的，或者可能引出其他什么问题；
* 5). 错误或者致命信息，理应放到Error级别，血红的颜色很显眼~
 

## 管理方面

1.会议总结问题，就一定要着手解决，不解决会议时间就白浪费了。

2.对于人的期望，快速的给予回馈，帮助其向希望的方向发展。

更多经验分享请看我的另一篇分享:  [http://www.vmatianyu.cn/summarization-of-technical-experience.html](http://www.vmatianyu.cn/summarization-of-technical-experience.html
)

个人开源站点:[http://vmatianyu.cn](http://vmatianyu.cn)

