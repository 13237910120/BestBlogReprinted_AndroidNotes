# 硬件加速 setLayerType

来源:[硬件加速 setLayerType](http://blog.csdn.net/lsw305264677/article/details/51383181)

从Android 3.0开始，Android的2D渲染管线可以更好的支持硬件加速。硬件加速使用GPU进行View上的绘制操作。

硬件加速可以在以下四个级别开启或关闭：

* Application
* Activity
* Window
* View

## Application级别
往您的应用程序AndroidManifest.xml文件为application标签添加如下的属性即可为整个应用程序开启硬件加速：

```
<application android:hardwareAccelerated="true" ...>
```

## Activity级别

您还可以控制每个activity是否开启硬件加速，只需在activity元素中添加android:hardwareAccelerated属性即可办到。比如下面的例子，在application级别开启硬件加速，但在某个activity上关闭硬件加速。

```
<application android:hardwareAccelerated="true">    
	<activity ... />    
	<activity android:hardwareAccelerated="false" />
</application>
```

## Window级别

如果您需要更小粒度的控制，可以使用如下代码开启某个window的硬件加速：

```
getWindow().setFlags(WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED,
					    WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);
```

注：目前还不能在window级别关闭硬件加速。

## View级别

您可以在运行时用以下的代码关闭单个view的硬件加速：

```
myView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
```

注：您不能在view级别开启硬件加速

## 为什么需要这么多级别的控制？

很明显，硬件加速能够带来性能提升，android为什么要弄出这么多级别的控制，而不是默认就是全部硬件加速呢？原因是并非所有的2D绘图操作支持硬件加速，如果您的程序中使用了自定义视图或者绘图调用，程序可能会工作不正常。如果您的程序中只是用了标准的视图和Drawable，放心大胆的开启硬件加速吧！具体是哪些绘图操作不支持硬件加速呢?以下是已知不支持硬件加速的绘图操作：

### Canvas

* clipPath()
* clipRegion()
* drawPicture()
* drawPosText()
* drawTextOnPath()
* drawVertices()

### Paint

* setLinearText()
* setMaskFilter()
* setRasterizer()

另外还有一些绘图操作，开启和不开启硬件加速，效果不一样：

### Canvas

* clipRect()： XOR, Difference和ReverseDifference裁剪模式被忽略，3D变换将不会应用在裁剪的矩形上。
* drawBitmapMesh()：colors数组被忽略
* drawLines()：反锯齿不支持
* setDrawFilter()：可以设置，但无效果

### Paint

* setDither()： 忽略
* setFilterBitmap()：过滤永远开启
* setShadowLayer()：只能用在文本上

### ComposeShader

* ComposeShader只能包含不同类型的shader (比如一个BitmapShader和一个LinearGradient，但不能是两个BitmapShader实例)s
* ComposeShader不能包含ComposeShader
如果应用程序受到这些影响，您可以在受影响的部分调用

`setLayerType(View.LAYER_TYPE_SOFTWARE, null)`，这样在其它地方仍然可以享受硬件加速带来的好处

## Android的绘制模型

开启硬件加速后，Android框架将采用新的绘制模型。基于软件的绘制模型和基于硬件的绘制模型有和不同呢？

### 基于软件的绘制模型

在软件绘制模型下，视图按照如下两个步骤绘制：

1. Invalidate the hierarchy（注：hierarchy怎么翻译？）
2. Draw the hierarchy

应用程序调用invalidate()更新UI的某一部分，失效(invalidation)消息将会在整个视图层中传递，计算每个需要重绘的区域（即脏区域）。然后Android系统将会重绘所有和脏区域有交集的view。很明显，这种绘图模式存在缺点：

1. 每个绘制操作中会执行不必要的代码。比如如果应用程序调用invalidate()重绘button，而button又位于另一个view之上，即使该view没有变化，也会进行重绘。
2. 可能会掩盖一些应用程序的bug。因为android系统会重绘与脏区域有交集的view，所以view的内容可能会在没有调用invalidate()的情况下重绘。这可能会导致一个view依赖于其它view的失效才得到正确的行为。

### 基于硬件的绘制模型

Android系统仍然使用invalidate()和draw()来绘制view，但在处理绘制上有所不同。Android系统记录绘制命令到显示列表，而不是立即执行绘制命令。另一个优化就是Android系统只需记录和更新标记为脏（通过invalidate()）的view。新的绘制模型包含三个步骤：

1. Invalidate the hierarchy
2. 记录和更新显示列表
3. 绘制显示列表


####  LAYER_TYPE_SOFTWARE

无论硬件加速是否打开，都会有一张Bitmap（software layer），并在上面对WebView进行软渲染。
好处：

在进行动画，使用software可以只画一次View树，很省。

什么时候不要用：

View树经常更新时不要用。尤其是在硬件加速打开时，每次更新消耗的时间更多。因为渲染完这张Bitmap后还需要再把这张Bitmap渲染到hardware layer上面去。


#### LAYER_TYPE_HARDWARE

硬件加速关闭时，作用同software。

硬件加速打开时会在FBO（Framebuffer Object）上面做渲染，在进行动画时，View树也只需要画一次。


两者区别：

1、一个是渲染到Bitmap，一个是渲染到FB上。<br/>
2、hardware可能会有一些操作不支持。

两者相同：

都是开了一个buffer，把View画到这个buffer上面去。


#### LAYER_TYPE_NONE

这个就比较简单了，不为这个View树建立单独的layer

**PS:GLSurfaceView和WebView默认Layertype都是none。**


* GLSurfaceView:

给GLSurfaceView设置为software或者hardware后，发现什么也画不出来了。得出结论：**GLSurfaceView的Layer type只能是none**


* WebView:

以前使用WebView时碰到过一个问题，如果在WebView上面使用Animation，WebView的绘画区域不动。当时的解决方案是在进行动画之前对WebView进行截屏（drawingcache）。按上面的道理试了一下，设置一个hardware或者software的layer就OK了。

现在又碰到了另外一个问题，打开硬件加速后，在一些机器上面（我的是3.2）WebView有时会出现某一块区域白屏的问题。默认的layer type是none，改为hardware也不行，设置为software就解决了。当然关闭硬件加速也好了，可是那样的话程序整体就比较慢了。所以最终方案是整体硬件加速，出问题的WebView设置software

补充于2012.4.21：

加上这一句，可以让3D的绘制更快一些：

> getHolder().setType(SurfaceHolder.SURFACE_TYPE_HARDWARE);

补充于2012.4.22

先说问题：

> 在硬件加速开启的情况下GLSurfaceView一旦被从View树上摘下来，会使整个窗口背景变黑，即使设置layer type为software也不管用。


> 经过两天的排查，发现了原因，我的程序是在Ｃ层由drawFrame（属于GLThread线程）来驱动进行绘画，当GLSurfaceView被摘下来时，GLSurfaceView的destroy方法被调用，我在destroy方法（属于UI线程）中直接调用 了GLThread线程的结束方法。而GLSurfaceView.creat,sizeChanged,destroyed在UI线程，Render.create,sizeChanged,drawFrame在GLThread线程。因此，出现了UI线程直接调用GLThread线程的方法的问题。最终通过GLSurfaceView.queueEvent向GLThread线程发送Runnable，问题得到解决。

看来，还是软渲染的容错能力比较强，一开硬件加速，底层就比较脆弱了。

结论：**一定要搞清楚哪个是UI线程，哪个是GLThread线程。**

补上几个寻找问题过程中发现的知识点：

hardware acclerator是对整个窗口进行加速，在硬件加速打开时View.isHardwareAcclerator返回true。但每个View可能被渲染到的Canvas是不同的，比如View可能被通过setLayer设置了Layer，这时，Canvas.isHardwareAccelerator返回false

Android提供了三种硬件加速是否打开的控制级别，分别是Application,Activity,Window,View。这个可以参考Dev Guide


