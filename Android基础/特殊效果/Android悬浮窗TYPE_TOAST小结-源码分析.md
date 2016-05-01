# Android悬浮窗TYPE_TOAST小结: 源码分析

来源:[简书](http://www.jianshu.com/p/634cd056b90c)

## 前言
[Android无需权限显示悬浮窗, 兼谈逆向分析app](http://www.jianshu.com/p/167fd5f47d5c)这篇文章阅读量很大, 这篇文章是通过逆向分析UC浏览器的实现和兼容性处理来得到一个悬浮窗的实现小技巧, 但有很多问题没有弄明白, 比如为什么在API 18及以下**TYPE_TOAST**的悬浮窗无法接受触摸事件, 为什么使用**TYPE_TOAST**就不需要权限.

期间[@廖祜秋liaohuqiu_秋百万](http://weibo.com/liaohuqiu)和我有较多探讨, 原文贴的一个[demo android-UCToast](https://github.com/liaohuqiu/android-UCToast)也是他做的, 他也有写[Android 悬浮窗的小结](http://liaohuqiu.net/cn/posts/android-windows-manager/). 这几篇关于悬浮窗的文章, 是我和他共同探索的结果, 非常感谢.

## 思路
老实说一开始我是想看看整个事件的传播过程, 从EventHub开始, 到View.onTouchEvent, 想看看Android系统内事件分发, 不过由于绝大部分代码在Native层, 我并没有搞清楚.

其实要想知道原因很简单, 只要grep一下**TYPE_TOAST**, 把每个用到的地方看一看, 自然就知道了, 但是恰好周末我手上没有源码, 只能在grepcode上面一个一个的查, 所以也花了不少时间.

## 正文
还是从最简单的地方开始, 我们调用了`WindowManager.addView`, **WindowManager**是个接口, 我们使用的是他的实现类**WindowManagerImpl**, 看看它的addView方法:

```
@Override
public void addView(View view, ViewGroup.LayoutParams    params) {
    mGlobal.addView(view, params, mDisplay, mParentWindow);
}
```

mGlobal是**WindowManagerGlobal**的实例, 再看看`WindowManagerGlobal.addView`:

```
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
	......
	final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
	......

	synchronized (mLock) {
		......
		root = new ViewRootImpl(view.getContext(), display);

		view.setLayoutParams(wparams);
		......
	}

	// do this last because it fires off messages to start doing things
	try {
		root.setView(view, wparams, panelParentView);
	} catch (RuntimeException e) {
		// BadTokenException or InvalidDisplayException, clean up.
		......
    }
```

代码中创建了一个`ViewRootImpl`, 调用了它的`setView,` 将我们要添加的view传入. 继续看`ViewRootImpl.setView`:

```
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            ......
            mWindowAttributes.copyFrom(attrs);
            if (mWindowAttributes.packageName == null) {
                mWindowAttributes.packageName = mBasePackageName;
            }
            ......
            try {
                ......
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(),
                        mAttachInfo.mContentInsets, mInputChannel);
            } catch (RemoteException e) {
                ......
                throw new RuntimeException("Adding window failed", e);
            } finally {
                ......
            }
            ......
        }
    }
}
```

对我们的分析来说最关键的代码是

```
res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
        getHostVisibility(), mDisplay.getDisplayId(),
        mAttachInfo.mContentInsets, mInputChannel);
```

`mWindowSession`的类型是`IWindowSession`, mWindow的类型是`IWindow.Stub`, 这句代码就是利用AIDL进行IPC, 实际被调用的是`Session.addToDisplay`:

```
@Override
public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
                        int viewVisibility, int displayId, Rect outContentInsets,
                        InputChannel outInputChannel) {
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
            outContentInsets, outInputChannel);
}
```

`mService`是`WindowManagerServic`e, 继续往下跟:

```
public int addWindow(Session session, IWindow client, int seq,
                     WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
                     Rect outContentInsets, InputChannel outInputChannel) {
    int[] appOp = new int[1];
    int res = mPolicy.checkAddPermission(attrs, appOp);
    if (res != WindowManagerGlobal.ADD_OKAY) {
        return res;
    }
    ......
    final int type = attrs.type;

    synchronized(mWindowMap) {
        ......

        mPolicy.adjustWindowParamsLw(win.mAttrs);
        ......
    }
    ......

    return res;
}
```

`mPolicy`是标记为final的成员变量:

```
final WindowManagerPolicy mPolicy = PolicyManager.makeNewWindowManager();
```

继续看`PolicyManager.makeNewWindowManager`:

```
public final class PolicyManager {
    private static final String POLICY_IMPL_CLASS_NAME =
        "com.android.internal.policy.impl.Policy";

    private static final IPolicy sPolicy;

    static {
        // Pull in the actual implementation of the policy at run-time
        try {
            Class policyClass = Class.forName(POLICY_IMPL_CLASS_NAME);
            sPolicy = (IPolicy)policyClass.newInstance();
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be loaded", ex);
        } catch (InstantiationException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(
                    POLICY_IMPL_CLASS_NAME + " could not be instantiated", ex);
        }
    }

    // Cannot instantiate this class
    private PolicyManager() {}

    ......
    public static WindowManagerPolicy makeNewWindowManager() {
        return sPolicy.makeNewWindowManager();
    }
    ......
}
```

这里`sPolicy`是`com.android.internal.policy.impl.Policy`对象, 再看看它的`makeNewWindowManager`方法返回的是什么:

```
public WindowManagerPolicy makeNewWindowManager() {
    return new PhoneWindowManager();
}
```

现在我们知道`mPolicy`实际上是`PhoneWindowManager`, 那么

```
int res = mPolicy.checkAddPermission(attrs, appOp);
```

实际调用的代码是:

```
@Override
public int checkAddPermission(WindowManager.LayoutParams attrs, int[] outAppOp) {
    int type = attrs.type;

    outAppOp[0] = AppOpsManager.OP_NONE;

    if (type < WindowManager.LayoutParams.FIRST_SYSTEM_WINDOW
            || type > WindowManager.LayoutParams.LAST_SYSTEM_WINDOW) {
        return WindowManagerGlobal.ADD_OKAY;
    }
    String permission = null;
    switch (type) {
        case TYPE_TOAST:
            // XXX right now the app process has complete control over
            // this...  should introduce a token to let the system
            // monitor/control what they are doing.
            break;
        case TYPE_DREAM:
        case TYPE_INPUT_METHOD:
        case TYPE_WALLPAPER:
        case TYPE_PRIVATE_PRESENTATION:
            // The window manager will check these.
            break;
        case TYPE_PHONE:
        case TYPE_PRIORITY_PHONE:
        case TYPE_SYSTEM_ALERT:
        case TYPE_SYSTEM_ERROR:
        case TYPE_SYSTEM_OVERLAY:
            permission = android.Manifest.permission.SYSTEM_ALERT_WINDOW;
            outAppOp[0] = AppOpsManager.OP_SYSTEM_ALERT_WINDOW;
            break;
        default:
            permission = android.Manifest.permission.INTERNAL_SYSTEM_WINDOW;
    }
    if (permission != null) {
        if (mContext.checkCallingOrSelfPermission(permission)
                != PackageManager.PERMISSION_GRANTED) {
            return WindowManagerGlobal.ADD_PERMISSION_DENIED;
        }
    }
    return WindowManagerGlobal.ADD_OKAY;
}
```

我截取的是4.4_r1的代码, 我们最关心的部分其实一直没有变, 那就是**TYPE_TOAST**根本没有做权限检查, 直接break出去了, 最后返回**WindowManagerGlobal.ADD_OKAY**.

不需要权限显示悬浮窗的原因已经找到了, 接着刚才addWindow方法的分析, 继续看下面一句:

``
mPolicy.adjustWindowParamsLw(win.mAttrs);
```

也就是`PhoneWindowManager.adjustWindowParamsLw`, 注意这里我给出了三个版本的实现, 一个是2.0到2.3.7实现的版本, 一个是4.0.1到4.3.1实现的版本, 一个是4.4实现的版本:

```
//Android 2.0 - 2.3.7 PhoneWindowManager
public void adjustWindowParamsLw(WindowManager.LayoutParams attrs) {
    switch (attrs.type) {
        case TYPE_SYSTEM_OVERLAY:
        case TYPE_SECURE_SYSTEM_OVERLAY:
        case TYPE_TOAST:
            // These types of windows can't receive input events.
            attrs.flags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
            break;
    }
}

//Android 4.0.1 - 4.3.1 PhoneWindowManager
public void adjustWindowParamsLw(WindowManager.LayoutParams attrs) {
    switch (attrs.type) {
        case TYPE_SYSTEM_OVERLAY:
        case TYPE_SECURE_SYSTEM_OVERLAY:
        case TYPE_TOAST:
            // These types of windows can't receive input events.
            attrs.flags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
            attrs.flags &= ~WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;
            break;
    }
}


//Android 4.4 PhoneWindowManager
@Override
public void adjustWindowParamsLw(WindowManager.LayoutParams attrs) {
    switch (attrs.type) {
        case TYPE_SYSTEM_OVERLAY:
        case TYPE_SECURE_SYSTEM_OVERLAY:
            // These types of windows can't receive input events.
            attrs.flags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                    | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE;
            attrs.flags &= ~WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;
            break;
    }
}
```

grepcode上没有3.x的代码, 我也没查具体是什么, 没必要考虑3.x.

可以看到, 在4.0.1以前, 当我们使用**TYPE_TOAST**, Android会偷偷给我们加上**FLAG_NOT_FOCUSABLE**和**FLAG_NOT_TOUCHABLE**, 4.0.1开始, 会额外再去掉**FLAG_WATCH_OUTSIDE_TOUCH**, 这样真的是什么事件都没了. 而4.4开始, **TYPE_TOAST**被移除了, 所以从4.4开始, 使用**TYPE_TOAST**的同时还可以接收触摸事件和按键事件了, 而4.4以前只能显示出来, 不能交互.

**API level 18及以下使用TYPE_TOAST无法接收触摸事件的原因**也找到了.

## 尾声

原文发的时候很多事情没搞清楚, 后来文章编辑了十几次, 加上这篇文章, 基本上把所有的疑问都搞明白了. 嗯, 关于这个神奇的悬浮窗的事情应该到这里就结束了.

本人水平有限, 如有错误, 欢迎指正, 以免误导他人
