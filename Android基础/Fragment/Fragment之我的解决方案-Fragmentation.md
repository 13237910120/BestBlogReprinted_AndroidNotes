# Fragment之我的解决方案：Fragmentation

来源:[www.jianshu.com](http://www.jianshu.com/p/38f7994faa6b)

* 1、[Fragment全解析系列（一）：那些年踩过的坑](http://www.jianshu.com/p/d9143a92ad94)
* 2、[Fragment全解析系列（二）：正确的使用姿势](http://www.jianshu.com/p/fd71d65f0ec6)
* 3、[Fragment之我的解决方案：Fragmentation](http://www.jianshu.com/p/38f7994faa6b)

附：[SwipeBackFragment的实现分析](http://www.jianshu.com/p/626229ca4dc2)

如果你通读了本系列的前两篇，我相信你可以写出大部分场景都能正常运行的Fragment了。如果你想了解更多，那么你可以看看我封装的这个库：Fragmentation。

本篇主要介绍这个库，解决了一些BUG，使用简单，提供实时查看栈视图等实用功能。

源码地址：[Github](https://github.com/YoKeyword/Fragmentation)，欢迎Fork，提Issues 。

[Demo网盘下载](http://yun.baidu.com/share/link?shareid=2711640160&uk=1647198224&third=0)

Demo演示：单Activity+多Fragment

![](fragment-summary/2.gif)

## Fragmentation

为"单Activity ＋ 多Fragment的架构","多模块Activity + 多Fragment的架构"而生，帮你简化使用过程，修复了官方Fragment库存在的一些BUG。

## 特性

* 1、为重度使用Fragment而生
* 2、提供了方便的管理Fragment的方法
* 3、有效解决Fragment重叠问题
* 4、实时查看Fragment的(包括嵌套Fragment)栈视图，方便Fragment嵌套时的调试
* 5、增加启动模式、startForResult等类似Activity方法
* 6、修复官方库里pop(tag/id)出栈多个Fragment时的一些BUG
* 7、完美解决进出栈动画的一些BUG，更自由的管理Fragment的动画
* 8、支持SwipeBack滑动边缘退出(需要使用Fragmentation_SwipeBack库,详情[README](https://github.com/YoKeyword/Fragmentation/blob/master/fragmentation_swipeback/README.md))

## 如何使用

* 1、项目下app的build.gradle中依赖：

```
compile 'me.yokeyword:fragmentation:0.5.4'
// appcompat v7包也是必须的
// 如果想使用SwipeBack 滑动边缘退出Fragment/Activity功能，请再添加下面的库
// compile 'me.yokeyword:fragmentation-swipeback:0.3.0'
```

* 2、Activity继承SupportActivity：

```
public class MainActivity extends SupportActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(...);
        if (savedInstanceState == null) {
            start(HomeFragment.newInstance());
        }
    }

    /**
    *  设置Container的id，必须实现
    */
    @Override
    public int setContainerId() {
        return R.id.fl_container;
    }

    /**
     *  设置全局动画，在SupportFragment可以自由更改其动画
     */
    @Override
    protected FragmentAnimator onCreateFragmentAnimator() {
        // 默认竖向(和安卓5.0以上的动画相同)
        return super.onCreateFragmentAnimator();
        // 设置横向(和安卓4.x动画相同)
        // return new DefaultHorizontalAnimator();
        // 设置自定义动画
        // return new FragmentAnimator(enter,exit,popEnter,popExit);
    }
```

* 3、Fragment继承SupportFragment

## API

### SupportActivity

打开 栈视图 的提示框，在复杂嵌套的时候，可以通过这个来清洗的理清不同阶级的栈视图。

```
// 弹出 栈视图 提示框
showFragmentStackHierarchyView();
```

除此之外包含大部分SupportFragment的方法，请自行查看。

### SupportFragment

1、启动相关：

```
// 启动新的Fragment
start(SupportFragment fragment)
// 以某种启动模式，启动新的Fragment
start(SupportFragment fragment, int launchMode)
// 启动新的Fragment，并能接收到新Fragment的数据返回
startForResult(SupportFragment fragment,int requestCode)
// 启动目标Fragment，并关闭当前Fragment
startWithFinish(SupportFragment fragment)
```

2、关闭Fragment：

```
// 关闭当前Fragment
pop();

// 关闭某一个Fragment栈内之上的Fragments
popTo(Class fragmentClass, boolean includeSelf);

// 如果想出栈后，紧接着开始.beginTransaction()开始一个事务，请使用下面的方法：
// 在 第二篇 文章内的 Fragment事务部分的问题 有解释原因
popTo(Class fragmentClass, boolean includeSelf, Runnable afterTransaction)
```

3、在SupportFragment内，支持监听返回按钮事件：

```
@Override
public boolean onBackPressedSupport() {
   // 返回false,则继续传递返回事件； 返回true则不继续传递
    return false;
}
```

4、 定义当前Fragment的动画，复写onCreateFragmentAnimation方法：

```
@Override
protected FragmentAnimator onCreateFragmentAnimation() {
    // 获取在SupportActivity里设置的全局动画对象
    FragmentAnimator fragmentAnimator = _mActivity.getFragmentAnimator();
    fragmentAnimator.setEnter(0);
    fragmentAnimator.setExit(0);
    return fragmentAnimator;

    // 也可以直接通过
    // return new FragmentAnimator(enter,exit,popEnter,popExit)设置一个全新的动画
}
```

5、获取当前Activity/Fragment内栈顶(子)Fragment：

```
getTopFragment();
```

6、获取栈内某个Fragment对象：

```
findFragment(Class fragmentClass);

// 获取某个子Fragment对象
findChildFragment(Class fragmentClass);
```

## 更多

隐藏/显示 输入法:

```
// 隐藏软键盘 一般用在onHiden里
hideSoftInput();
// 显示软键盘
showSoftInput(View view);
```

下面是DetailFragment  startForResult ModifyTitleFragment的代码：

DetailFragment.class里:

```
startForResult(ModifyDetailFragment.newInstance(mTitle), REQ_CODE);
@Override
public void onFragmentResult(int requestCode, int resultCode, Bundle data) {
    super.onFragmentResult(requestCode, resultCode, data);
    if (requestCode == REQ_CODE && resultCode == RESULT_OK ) {
    // 在此通过Bundle data 获取返回的数据
    }
}
```

ModifyTitleFragment.class里:

```
Bundle bundle = new Bundle();
bundle.putString("title", "xxxx");
setFramgentResult(RESULT_OK, bundle);
```

下面是以一个singleTask模式start一个Fragment的标准代码：

```
HomeFragment fragment = findFragment(HomeFragment.class);
if (fragment == null) {
    fragment = HomeFragment.newInstance();
}else{
    Bundle newBundle = new Bundle();
    // 传递的bundle数据，会调用目标Fragment的onNewBundle(Bundle newBundle)方法
    fragment.putNewBundle(newBundle);
}
// homeFragment以SingleTask方式启动
start(fragment, SupportFragment.SINGLETASK);

// 在HomeFragment.class中：
@Override
protected void onNewBundle(Bundle newBundle){
    // 在此可以接收到数据
}
```

关于Fragmentation帮你恢复Fragment，你需要知道的2个概念：

> "同级"式：比如QQ的主界面，“消息”、“联系人”、“动态”，这3个Fragment属于同级关系<br/>
> “流程”式：比如登录->注册/忘记密码->填写信息->跳转到主页Activity

对于Activity内的“流程”式Fragments（比如登录->注册/忘记密码->填写信息->跳转到主页Activity），Fragmentation帮助你处理了栈内的恢复，保证Fragment不会重叠，你不需要再自己处理了。

但是如果你的Activity内的Fragments是“同级”的，那么需要你复写onHandleSaveInstanceState()使用findFragmentByTag(tag)或getFragments()去恢复处理。

```
@Override
protected void onHandleSaveInstancState(Bundle savedInstanceState) {
        // 复写的时候 下面的super一定要删掉
        // super.onHandleSaveInstancState(savedInstanceState);
        // 在此处 通过findFragmentByTag或getFraments来恢复，详情参考第二篇文章
}
```

而如果你有Fragment嵌套，那么不管是“同级”式还是“流程”式，你都需要自己去恢复处理。
