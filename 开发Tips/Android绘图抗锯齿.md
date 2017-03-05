# 抗锯齿方法两种（其一：paint.setAntiAlias(ture);paint.setBitmapFilter(true))

来源:[CSDN](http://blog.csdn.net/yixinyouni1314/article/details/7774164)

在Android中，目前，我知道有两种出现锯齿的情况。 

* 当我们用Canvas绘制位图的时候，如果对位图进行了选择，则位图会出现锯齿。 
* 在用View的RotateAnimation做动画时候，如果View当中包含有大量的图形，也会出现锯齿。

我们分别以这两种情况加以考虑。

* 用Canvas绘制位的的情况。在用Canvas绘制位图时，一般地，我们使用drawBitmap函数家族，在这些函数中，都有一个Paint参数，要做到防止锯齿，我们就要使用到这个参数。如下：
   * 首先在你的构造函数中，需要创建一个Paint。 Paint mPaint = new Paint（）； 
   * 然后，您需要设置两个参数: 第一个函数是用来防止边缘的锯齿，第二个函数是用来对位图进行滤波处理。
      * 1)mPaint.setAntiAlias(); 
      * 2)mPaint.setBitmapFilter(true)。
   
   * 最后，在画图的时候，调用drawBitmap函数，只需要将整个Paint传入即可。 

* 有时候，当你做RotateAnimation时，你会发现，讨厌的锯齿又出现了。这个时候，由于你不能控制位图的绘制，只能用其他方法来实现防止锯齿。另外，如果你画的位图很多。不想每个位图的绘制都传入一个Paint。还有的时候，你不可能控制每个窗口的绘制的时候，您就需要用下面的方法来处理——对整个Canvas进行处理。 

   * 在您的构造函数中，创建一个Paint滤波器。
    
     > PaintFlagsDrawFilter mSetfil = new PaintFlagsDrawFilter(0, Paint.FILTER_BITMAP_FLAG);
     
     第一个参数是你要清除的标志位，第二个参数是你要设置的标志位。此处设置为对位图进行滤波。 
     
   * 当你在画图的时候，如果是View则在`onDraw`当中，如果是ViewGroup则在`dispatchDraw`中调用如下函数。 
   
      > canvas.setDrawFilter( mSetfil );

* 最后，另外，在Drawable类及其子类中，也有函数`setFilterBitmap`可以用来对Bitmap进行滤波处理，这样，当你选择Drawable时，会有抗锯齿的效果。