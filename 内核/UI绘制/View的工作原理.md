# View的工作原理

来源:[sparkyuan.me](http://sparkyuan.me/2016/03/11/View%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)

View的绘制流程是从ViewRoot的performTraversals方法开始的，它经过measure、layout和draw三个过程才能最终将一个View绘制出来，其中measure用来测量View的宽和高，layout用来确定View在父容器中的放置位置，而draw则负责将View绘制在屏幕上。

## ViewRoot和DecorView

### ViewRoot
ViewRoot对应ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均通过ViewRoot来完成。
ActivityThread中，Activity创建完成后，会将DecorView添加到Window中，同时创建ViewRootImpl对象，并建立两者的关联。

### DecorView
DecorView作为顶级View，一般情况下它内部包含一个竖直方向的LinearLayout，在这个LinearLayout里面有上下两个部分（具体情况和Android版本及主体有关），上面的是标题栏，下面的是内容栏。在Activity中通过setContentView所设置的布局文件其实就是被加到内容栏之中的，而内容栏的id是content，在代码中可以通过`ViewGroup content = （ViewGroup)findViewById(R.android.id.content)`来得到content对应的layout。

DecorView其实是一个FrameLayout，View层的事件都先经过DecorView，然后才传递给我们的View。

## MeasureSpec

在测量过程中，**系统会将View的LayoutParams根据父容器所施加的规则转换成对应的MeasureSpec，然后再根据这个MeasureSpec来测量出View的宽和高**。测量出来的宽和高不一定等于View最终的宽和高。

MeasureSpec将SpecMode和SpecSize打包成一个int值来避免过多的对象内存分配，高2位代表SpecMode，低30位代表SpecSize，SpecMode是指测量模式，而SpecSize是指在某种测量模式下的规格大小。

SpecMode有三类：

* **UNSPECIFIED：**父容器不对View有任何限制，要多大给多大，这种情况一般用于系统内部，表示一种测量状态
* **EXACTLY：**父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值。它对应于LayoutParams中的match_parent和具体的数值这两种模式
* **AT_MOST：**父容器指定了一个可用大小即SpecSize，View的大小不能大于这个值，具体是什么值要看不同View的具体实现。它对应于LayoutParams中的wrap_content。

## 普通MeasureSpec的创建规则

**对于普通View，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同决定。**

* 子View为精确宽高，无论父容器的MeasureSpec，子View的MeasureSpec都为精确值且遵循LayoutParams中的值。
* 子View为match_parent时，如果父容器是精确模式，则子View也为精确模式且为父容器的剩余空间大小；如果父容器是最大模式，则子View也是wrap_content且不会超过父容器的剩余空间。
* 子View为wrap_content时，无论父View是精确还是最大模式，子View的模式总是最大模式，且不会超过父容器的剩余空间。

## View的工作流程

### measure

ViewGroup的measure方法会遍历每个子元素，并调用子元素内部的measure方法，measure源码如下：

````
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
               getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
   }
```

注：

* getDefaultSize()返回MeasureSpec中的specSize，也就是View测量后的大小。
* getSuggestedMinimumWidth()，View如果没有背景，那么返回android:minWidth这个属性指定的值，这个值可以为0；如果设置了背景，则返回背景的最小宽度和minWidth中的最大值。
* getSuggestedMinimumHeight()，与getSuggestedMinimumWidth()类似。
* 直接继承View的自定义控件需要重写onMeasure方法并设置wrap_content时的自身大小，否则在布局中使用wrap_content时就相当于使用match_parent。因为LayoutParams=wrap_content的情况下，MeasureSpec为AT_MOST，所以View的宽和高为父容器当前剩余的空间，这种效果与match_parent一致。具体处理方法要根据需求灵活决定。

### 如何得到View的宽和高

在Activity的onCreate、onStart、onResume方法中均无法正确得到某个View的宽/高信息，这是因为View的measure过程和Activity的生命周期方法不是同步执行的，因此无法保证Activity执行了onCreate、onStart、onResume时某个View就已经测量完毕了，如果View还没有测量完毕，那么获得的宽/高就是0。

可以通过如下四个方法来解决这个问题：

* Activity或者View的onWindowFocusChanged方法（注意该方法会在Activity Pause和resume时被多次调用）
* view.post(new Runnable( {@Overidde public void run(){})})，在run方法中获取。
* ViewTreeObserver中的onGlobalLayoutListener中。
* 手动调用View的measure方法。

示例代码请参考原书P190页

### layout
layout的作用是用来确定子视图在父视图中的位置。源码如下：

```
public void layout(int l, int t, int r, int b) {  
    int oldL = mLeft;  
    int oldT = mTop;  
    int oldB = mBottom;  
    int oldR = mRight;  
    boolean changed = setFrame(l, t, r, b);  
    if (changed || (mPrivateFlags & LAYOUT_REQUIRED) == LAYOUT_REQUIRED) {  
        if (ViewDebug.TRACE_HIERARCHY) {  
            ViewDebug.trace(this, ViewDebug.HierarchyTraceType.ON_LAYOUT);  
        }  
        onLayout(changed, l, t, r, b);  
        mPrivateFlags &= ~LAYOUT_REQUIRED;  
        if (mOnLayoutChangeListeners != null) {  
            ArrayList<OnLayoutChangeListener> listenersCopy =  
                    (ArrayList<OnLayoutChangeListener>) mOnLayoutChangeListeners.clone();  
            int numListeners = listenersCopy.size();  
            for (int i = 0; i < numListeners; ++i) {  
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);  
            }  
        }  
    }  
    mPrivateFlags &= ~FORCE_LAYOUT;  
}
```

通过setFrame()确定四个顶点的位置，进而确定View在父容器中的位置。

在View的默认实现中，View的测量宽/高和最终宽/高是相等的，只不过测量宽/高形成于View的measure过程，而最终宽/高形成于View的layout过程，即两者的赋值时机不同，测量宽/高的赋值时机稍微早一些。多数情况下可以认为View的测量宽/高就等于最终的宽/高，但对于在View的layout中改变了View的left、top、right、bottom四个属性时，得出的测量宽/高有可能和最终的宽/高不一致。

### draw

draw的过程很简单主要有以下几步：

* 绘制背景(background.draw)
* 绘制自己(onDraw)
* 绘制children(dispatchDraw)
* 绘制装饰(onDrawScrollBars)。

源码如下

```
public void draw(Canvas canvas) {  
  
        / * Draw traversal performs several drawing steps which must be executed  
         * in the appropriate order:  
         *  
         *      1. Draw the background if need  
         *      2. If necessary, save the canvas' layers to prepare for fading  
         *      3. Draw view's content  
         *      4. Draw children (dispatchDraw)  
         *      5. If necessary, draw the fading edges and restore layers  
         *      6. Draw decorations (scrollbars for instance)  
         */  
  
       //Step 1, draw the background, if needed  
        if (!dirtyOpaque) {  
            drawBackground(canvas);  
        }  
  
         // skip step 2 & 5 if possible (common case)  
        final int viewFlags = mViewFlags;  
        if (!verticalEdges && !horizontalEdges) {  
            // Step 3, draw the content  
            if (!dirtyOpaque) onDraw(canvas);  
  
            // Step 4, draw the children  
            dispatchDraw(canvas);  
  
            // Step 6, draw decorations (scrollbars)  
            onDrawScrollBars(canvas);  
  
            if (mOverlay != null && !mOverlay.isEmpty()) {  
                mOverlay.getOverlayView().dispatchDraw(canvas);  
            }  
  
            // we're done...  
            return;  
        }  
  
        // Step 2, save the canvas' layers  
        ...  
  
        // Step 3, draw the content  
        if (!dirtyOpaque)   
            onDraw(canvas);  
  
        // Step 4, draw the children  
        dispatchDraw(canvas);  
  
        // Step 5, draw the fade effect and restore layers  
  
        // Step 6, draw decorations (scrollbars)  
        onDrawScrollBars(canvas);  
    }
```

注：

* View有一个特殊的方法setWillNotDraw，如果一个View不需要绘制任何内容，设置这个标记位true后，系统会进行优化。默认情况下，View没有启用这个优化标记位，但是ViewGroup会默认启用这个优化标记位。
* 这个标记位对实际开发的意义是：如果自定义控件继承于ViewGroup并且本身不具备绘制功能时，就可以开启这个标记位从而便于系统进行后续的优化。当明确知道一个ViewGroup需要通过onDraw来绘制内容时，需要显示地关闭WILL_NOT_DRAW这个标记位。

欢迎转载，转载请注明出处[http://sparkyuan.me/](http://sparkyuan.me/)