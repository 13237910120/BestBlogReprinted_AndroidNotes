# Android View中getViewTreeObserver().addOnGlobalLayoutListener()

来源:[http://blog.csdn.net/linghu_java/article/details/46544811](http://blog.csdn.net/linghu_java/article/details/46544811)

我们知道在`onCreate`中`View.getWidth`和`View.getHeight`无法获得一个view的高度和宽度，这是因为View组件布局要在`onResume`回调后完成。所以现在需要使用`getViewTreeObserver().addOnGlobalLayoutListener()`来获得宽度或者高度。这是获得一个view的宽度和高度的方法之一。

`OnGlobalLayoutListener`是`ViewTreeObserver`的内部类，当一个视图树的布局发生改变时，可以被`ViewTreeObserver`监听到，这是一个注册监听视图树的观察者(observer)，在视图树的全局事件改变时得到通知。`ViewTreeObserver`不能直接实例化，而是通过`getViewTreeObserver()`获得。

除了`OnGlobalLayoutListener` ，`ViewTreeObserver`还有如下内部类：



* `interface ViewTreeObserver.OnGlobalFocusChangeListener`

当在一个视图树中的焦点状态发生改变时，所要调用的回调函数的接口类

* `interface ViewTreeObserver.OnGlobalLayoutListener*

当在一个视图树中全局布局发生改变或者视图树中的某个视图的可视状态发生改变时，所要调用的回调函数的接口类

* `interface ViewTreeObserver.OnPreDrawListener`

当一个视图树将要绘制时，所要调用的回调函数的接口类

* `interface ViewTreeObserver.OnScrollChangedListener`

当一个视图树中的一些组件发生滚动时，所要调用的回调函数的接口类

* `interface ViewTreeObserver.OnTouchModeChangeListener`

当一个视图树的触摸模式发生改变时，所要调用的回调函数的接口类

其中，我们可以利用`OnGlobalLayoutListener`来获得一个视图的真实高度。

```
int mHeaderViewHeight;  
mHeaderView.getViewTreeObserver().addOnGlobalLayoutListener(  
        new OnGlobalLayoutListener() {  
            @Override  
            public void onGlobalLayout() {  
                                                                                                                                                                                                                                        
                mHeaderViewHeight = mHeaderView.getHeight();  
                getViewTreeObserver()  
                        .removeGlobalOnLayoutListener(this);  
            }  
        }); 
```

但是需要注意的是`OnGlobalLayoutListener`可能会被多次触发，因此在得到了高度之后，要将`OnGlobalLayoutListener`注销掉。

有时候需要在`onCreate`方法中知道某个View组件的宽度和高度等信息，而直接调用View组件的`getWidth()`、`getHeight()`、`getMeasuredWidth()`、`getMeasuredHeight()`、`getTop()`、`getLeft()`等方法是无法获取到真实值的，只会得到0。这是因为View组件布局要在`onResume`回调后完成。下面提供实现方法，`onGlobalLayout`回调会在view布局完成时自动调用:

类似：

```
// This listener is used to get the final width of the GridView and then calculate the  
// number of columns and the width of each column. The width of each column is variable  
// as the GridView has stretchMode=columnWidth. The column width is used to set the height  
// of each view so we get nice square thumbnails.  
mGridView.getViewTreeObserver().addOnGlobalLayoutListener( //view 布局完成时调用，每次view改变时都会调用  
        new ViewTreeObserver.OnGlobalLayoutListener() {  
            @Override  
            public void onGlobalLayout() {  
                if (mAdapter.getNumColumns() == 0) {  
                        final int numColumns = (int) Math.floor(  
                                 mGridView.getWidth() / (mImageThumbSize + mImageThumbSpacing));  
                    if (numColumns > 0) {  
                        final int columnWidth =  
                            (mGridView.getWidth() / numColumns) - mImageThumbSpacing;  
                        mAdapter.setNumColumns(numColumns);   //设置 列数  
                            mAdapter.setItemHeight(columnWidth);  //设置 高度  
                    }  
                }  
            }  
        });  
```

在gridview布局完成后，根据girdview的宽和高设置adapter列数和每个item高度

