# ListView和SwipeRefreshView冲突解决

来源：<br/>
[关于SwipeRefreshLayout和ListView滑动冲突问题解决办法](http://www.eoeandroid.com/thread-914273-1-1.html?_dsign=bf09d67b)<br/>
[解决listview与SwipeRefreshLayout滑动冲突问题](http://blog.csdn.net/lijinhua7602/article/details/41114397)<br/>
[滑动ViewPager引起swiperefreshlayout刷新的冲突](http://www.cnblogs.com/kakaleee/p/4697028.html?utm_source=tuicool&utm_medium=referral)<br/>

## 1. 关于SwipeRefreshLayout和ListView滑动冲突问题解决办法

自从Google推出`SwipeRefreshLayout`后相信很多人都开始使用它来实现listView的下拉刷新了，但是在使用的时候，有一个很有趣的现象，当`SwipeRefreshLayout`只有listview一个子view的时候是没有任何问题的，但如果不是得话就会出现问题了，向上滑动listview一切正常，向下滑动的时候就会出现还没有滑倒listview顶部就触发下拉刷新的动作了，看`SwipeRefreshLayout`源码可以看到在`onInterceptTouchEvent`里面有这样的一段代码

```
if (!isEnabled() || mReturningToStart || canChildScrollUp() || mRefreshing) {
	// Fail fast if we're not in a state where a swipe is possible
	return false;
}
```

其中有个`canChildScrollUp`方法，在往下看

```
public boolean canChildScrollUp() {
	if (android.os.Build.VERSION.SDK_INT < 14) {
		if (mTarget instanceof AbsListView) {
			final AbsListView absListView = (AbsListView) mTarget;
			return absListView.getChildCount() > 0
				&& (absListView.getFirstVisiblePosition() > 0 
				|| absListView.getChildAt(0)
					.getTop() < absListView.getPaddingTop());
		} else {
			return ViewCompat.canScrollVertically(mTarget, -1) || mTarget.getScrollY() > 0;
		}
	} else {
            return ViewCompat.canScrollVertically(mTarget, -1);
}
```

决定子view 能否滑动就是在这里了，所以我们只有写一个类继承`SwipeRefreshLayout`，然后重写该方法即可

```
public class SimpleSwipeRefreshLayout extends SwipeRefreshLayout {

    private View view;
    public SimpleSwipeRefreshLayout(Context context) {
        super(context);
    }

    public SimpleSwipeRefreshLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public void setViewGroup(View view) {
        this.view = view;
    }

    @Override
    public boolean canChildScrollUp() {
        if (view != null && view instanceof AbsListView) {
            final AbsListView absListView = (AbsListView) view;
            return absListView.getChildCount() > 0
                    && (absListView.getFirstVisiblePosition() > 0 
                    || absListView.getChildAt(0)
                    	.getTop() < absListView.getPaddingTop());
        }
        return super.canChildScrollUp();
    }
}
```

## 2. 解决listview与SwipeRefreshLayout滑动冲突问题

```
listView.setOnScrollListener(new OnScrollListener() {

@Override
public void onScrollStateChanged(AbsListView view, int scrollState) {
}

@Override
public void onScroll(AbsListView view, int firstVisibleItem,
        int visibleItemCount, int totalItemCount) {
    boolean enable = false;
    if(listView != null && listView.getChildCount() > 0){
        // check if the first item of the list is visible
        boolean firstItemVisible = listView.getFirstVisiblePosition() == 0;
        // check if the top of the first item is visible
        boolean topOfFirstItemVisible = listView.getChildAt(0).getTop() == 0;
        // enabling or disabling the refresh layout
        enable = firstItemVisible && topOfFirstItemVisible;
    }
    swipeRefreshLayout.setEnabled(enable);
}});
```

以上方案在ListView没有数据的时候有问题，所以改正如下：

```
mLvAttentions.setOnScrollListener(new AbsListView.OnScrollListener() {

    @Override
    public void onScrollStateChanged(AbsListView view, int scrollState) {
    }

    @Override
    public void onScroll(AbsListView view, int firstVisibleItem,
                         int visibleItemCount, int totalItemCount) {

        boolean enable = false;
        if(mLvAttentions != null && mLvAttentions.getChildCount() > 0){
            // check if the first item of the list is visible
            boolean firstItemVisible = mLvAttentions.getFirstVisiblePosition() == 0;
            // check if the top of the first item is visible
            boolean topOfFirstItemVisible = mLvAttentions.getChildAt(0).getTop() == 0;
            // enabling or disabling the refresh layout
            enable = firstItemVisible && topOfFirstItemVisible;
        }else if(mLvAttentions != null){
            enable = true;
        }
        mRefresh.setEnabled(enable);
    }});
```

## 3.滑动ViewPager引起swiperefreshlayout刷新的冲突

`ViewPager`是Android中提供的页面切换的控件，`SwipeRefreshLayout`是Android提供的下拉刷新控件，通过`SwipeRefreshLayout`可以很简单的实现下拉刷新的功能，但是如果`SwipeRefreshLayout`的子view中如果包含了ViewPager，会发现滑动ViewPager的时候，很容易引起`SwipeRefreshLayout`的下拉刷新操作为了解决这个冲突可以这样实现

```
viewPager.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View v, MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_MOVE:
                swipeRefreshLayout.setEnabled(false);
            break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                swipeRefreshLayout.setEnabled(true);
            break;
        }
    return false;
    }
});
```

`ViewPager`，设置`OnTouchListener`，里面当ACTION_MOVE的时候设置`SwipeRefreshLayout`不可用，当`ACTION_UP`或者`ACTION_CANCEL`的时候设置`SwipeRefreshLayout`可以，就可以解决这个冲突了

