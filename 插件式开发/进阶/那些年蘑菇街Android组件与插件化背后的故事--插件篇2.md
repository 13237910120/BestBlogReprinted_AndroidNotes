# 那些年蘑菇街Android组件与插件化背后的故事 -- 插件篇(二)

来源:[那些年蘑菇街Android组件与插件化背后的故事 -- 插件篇(二)](http://mogu.io/119-119)

这阵子比较忙，组里的赤(xing)木(xing)大(dui)神(zhang)一直催我写第二篇。今天抽出空来，把这篇补上。

上一篇已经交代了如何把一个插件apk动态加载到内存中，并做到可用。这篇介绍的是，我们如何在“合适”的时间加载某个插件。
试想一个理想化的场景，一个app中，所有的模块都是一个个的插件，我们不可能在应用初始化的时候把所有的插件全部加载，dex optimize时长都能把人弄哭。

那么就得找一个时间点，使我们可以知道，某个插件第一次被用到是什么时候，只要在这个时间加载这个插件就好。
分析一下插件主要有哪些东西构成：

* 首先是逻辑代码，这些函数有些是自己插件内部用的，有些是外部其他插件也要用的。
* 页面，各种Activity。

Ok，我们一个个得解决上面的问题

## 对外函数

在这种情况下，如果一个插件A需要调用插件B的函数，我们可以去检查插件B是否被加载过，如果还未加载，就加载它。
这里不知道大家会不会有个疑问，我们是如何感知到插件A调用了插件B中的函数了呢？

首先有个天然性的隔离就是，插件与插件之间在开发的时候，都是独立工程的，它们之间不应该存在直接的函数调用。

然后，我们需要一个中间管理框架，用来统一处理插件间的函数(服务)调用，这也是蘑菇街组件管理框架所做的很重要的一件事。所有的插件/组件之间的函数(服务)调用，必须过管理框架中转，最后由框架进行实际的调用。

由于管理框架是一个通用框架，并不针对某个插件，所以插件所能提供的对外函数，要使用配置的方式告知外部和管理框架，未在配置定义的函数是不允许被调用的。

于是我们可以拦截到插件间的函数(服务)调用了，这是第一个可以检测到某个插件是否开始被使用的地方。

## 页面，Activity

另外一种常见情况就是插件A需要启动插件B中的一个Activity。为了让开发插件的同学最小化得感知自己在开发一个插件，业务开发的同学依然可以使用任何系统提供的任何启动Activity的方式。但是作为需要知道启动了哪个Activity以便去安装某个插件的管理框架来说，需要对这个操作进行拦截。

又到了read the fuck source code的时间啦~

首先我们从一个常见的startActivity函数调用开始，不论调用了哪个Context的startActivity函数，最后都会调用到ContextImpl的startActivity函数：

```
@Override
public void startActivity(Intent intent, Bundle options) {
    ...
    mMainThread.getInstrumentation().execStartActivity(
        getOuterContext(), mMainThread.getApplicationThread(), null,
        (Activity)null, intent, -1, options);
}
```

看到它最后调用了mMainThread.getInstrumentation().execStartActivity来启动一个Activity，mMainThread是一个ActivityThread类型的对象，getInstrumentation函数返回一个Instrumentation类型的对象。顺理成章得，接下去是Instrumentation类的execStartActivity函数：

```
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    try {
        ...
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
    }
    return null;
}
```

调用到了ActivityManagerNative的startActivity函数，这是一个IPC调用，用于向远端的ActivityManagerService发起一个start activity的请求。当然大家可以拦截Instrumentation的execStartActivity函数，因为通过函数的入参，已经知道了应用层请求启动哪个Activity了。但拦截这个函数有几个弊端，

* 第一就是execStartActivity函数只是启动Activity的某一个函数，类似的函数还有一些，如果我们都把它们拦截了，显得有点冗余了。
* 第二个弊端就是这里只是向ActivityManagerService发起一个IPC请求，具体这个Activity能否被启动，都还是未知数，如果在这里拦截而这个Activity，也会造成一些浪费和不确定性。

另外我们可以非常肯定得认为，在整个Activity启动的过程中，实例化某个具体Activity的操作一定是在当前应用程序进程进行的，不可能由远端的ActivityManagerService来执行，因为只有当前应用程序进行的内存中，才会有某个具体Activity的类存在。

于是我们在ActivityThread类中找到了最后实例化某个Activity类的函数

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

    ...

    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    ...

    return activity;
}
```

最关键的一句，就是调用mInstrumentation.newActivity来返回一个Activity对象，所以这里是最后实例化的地方，Instrumentation类的newActivity也就是我们需要拦截下来的操作。简单明了，只需要复写这个函数然后把自己的Instrumentation对象替换成运行上下文的就可以了~

到这里为止，Activity启动也被管理框架感知到了，自然可以去检测安装Activity所属的插件了。

以上就是蘑菇街组件与插件管理框架中插件安装的逻辑。


