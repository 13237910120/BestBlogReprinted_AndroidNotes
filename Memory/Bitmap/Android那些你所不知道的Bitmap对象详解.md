#  Android 那些你所不知道的Bitmap对象详解

来源:[http://blog.csdn.net/xiaanming/article/details/41084843](http://blog.csdn.net/xiaanming/article/details/41084843)

转载请注明本文出自xiaanming的博客（[http://blog.csdn.net/xiaanming/article/details/41084843](http://blog.csdn.net/xiaanming/article/details/41084843)），请尊重他人的辛勤劳动成果，谢谢！

我们知道Android系统分配给每个应用程序的内存是有限的，Bitmap作为消耗内存大户，我们对Bitmap的管理稍有不当就可能引发OutOfMemoryError，而Bitmap对象在不同的Android版本中存在一些差异，今天就给大家介绍下这些差异，并提供一些在使用Bitmap的需要注意的地方。

**在Android2.3.3(API 10)及之前的版本中，Bitmap对象与其像素数据是分开存储的，Bitmap对象存储在Dalvik heap中，而Bitmap对象的像素数据则存储在Native Memory（本地内存）中或者说Derict Memory（直接内存）中**，这使得存储在Native Memory中的像素数据的释放是不可预知的，我们可以调用recycle()方法来对Native Memory中的像素数据进行释放，前提是你可以清楚的确定Bitmap已不再使用了，如果你调用了Bitmap对象recycle()之后再将Bitmap绘制出来，就会出现"Canvas: trying to use a recycled bitmap"错误，而在Android3.0(API 11)之后，Bitmap的像素数据和Bitmap对象一起存储在Dalvik heap中，所以我们不用手动调用recycle()来释放Bitmap对象，内存的释放都交给垃圾回收器来做，也许你会问，为什么我在显示Bitmap对象的时候还是会出现OutOfMemoryError呢？
在说这个问题之前我顺便提一下，在Android2.2(API 8)之前，使用的是Serial垃圾收集器，从名字可以看出这是一个单线程的收集器，这里的”单线程"的意思并不仅仅是使用一个CPU或者一条收集线程去收集垃圾，更重要的是在它进行垃圾收集时，必须暂停其他所有的工作线程，Android2.3之后，这种收集器就被代替了，使用的是并发的垃圾收集器，这意味着我们的垃圾收集线程和我们的工作线程互不影响。

简单的了解垃圾收集器之后，我们对上面的问题举一个简单的例子，假如系统启动了垃圾回收线程去收集垃圾，而此时我们一下子产生大量的Bitmap对象，此时是有可能会产生OutOfMemoryError，因为垃圾回收器首先要判断某个对象是否还存活(JAVA语言判断对象是否存活使用的是根搜索算法 GC Root Tracing)，然后利用垃圾回收算法来对垃圾进行回收，不同的垃圾回收器具有不同的回收算法，这些都是需要时间的， 发生OutOfMemoryError的时候，我们要明确到底是因为内存泄露(Memory Leak)引发的还是内存溢出(Memory overflow)引发的，如果是内存泄露我们需要利用工具(比如MAT)查明内存泄露的代码并进行改正，如果不存在泄露，换句话来说就是内存中的对象确实还必须活着，那我们可以看看是否可以通过某种途径，减少对象对内存的消耗，比如我们在使用Bitmap的时候，应该根据View的大小利用BitmapFactory.Options计算合适的inSimpleSize来对Bitmap进行相对应的裁剪，以减少Bitmap对内存的使用，如果上面都做好了还是存在OutOfMemoryError(一般这种情况很少发生)的话，那我们只能调大Dalvik heap的大小了，在Android 3.1以及更高的版本中，我们可以在AndroidManifest.xml的application标签中增加一个值等于“true”的android:largeHeap属性来通知Dalvik虚拟机应用程序需要使用较大的Java Heap，但是我们也不鼓励这么做。

## 在Android 2.3及以下管理Bitmap

从上面我们知道，在Android2.3及以下我们推荐使用recycle()方法来释放内存，我们在使用ListView或者GridView的时候，该在什么时候去调用recycle()呢？这里我们用到引用计数，使用一个变量(dispalyRefCount)来记录Bitmap显示情况，如果Bitmap绘制在View上面displayRefCount加一, 否则就减一, 只有在displayResCount为0且Bitmap不为空且Bitmap没有调用过recycle()的时候，我们才需求对该Bitmap对象进行recycle()，所以我们需要用一个类来包装下Bitmap对象，代码如下:

```
package com.example.bitmap;  
  
import android.content.res.Resources;  
import android.graphics.Bitmap;  
import android.graphics.drawable.BitmapDrawable;  
  
public class RecycleBitmapDrawable extends BitmapDrawable {  
    private int displayResCount = 0;  
    private boolean mHasBeenDisplayed;  
  
    public RecycleBitmapDrawable(Resources res, Bitmap bitmap) {  
        super(res, bitmap);  
    }  
      
      
    /** 
     * @param isDisplay 
     */  
    public void setIsDisplayed(boolean isDisplay){  
        synchronized (this) {  
            if(isDisplay){  
                mHasBeenDisplayed = true;  
                displayResCount ++;  
            }else{  
                displayResCount --;  
            }  
        }  
          
        checkState();  
          
    }  
      
    /** 
     * 检查图片的一些状态，判断是否需要调用recycle 
     */  
    private synchronized void checkState() {  
        if (displayResCount <= 0 && mHasBeenDisplayed  
                && hasValidBitmap()) {  
            getBitmap().recycle();  
        }  
    }  
      
      
    /** 
     * 判断Bitmap是否为空且是否调用过recycle() 
     * @return 
     */  
    private synchronized boolean hasValidBitmap() {  
        Bitmap bitmap = getBitmap();  
        return bitmap != null && !bitmap.isRecycled();  
    }  
  
}
```

除了上面这个RecycleBitmapDrawable之外呢，我们还需要一个自定义的ImageView来控制什么时候显示Bitmap以及什么时候隐藏Bitmap对象

```
package com.example.bitmap;

import android.content.Context;
import android.graphics.drawable.Drawable;
import android.graphics.drawable.LayerDrawable;
import android.util.AttributeSet;
import android.widget.ImageView;

public class RecycleImageView extends ImageView {
	public RecycleImageView(Context context) {
		super(context);
	}

	public RecycleImageView(Context context, AttributeSet attrs) {
		super(context, attrs);
	}

	public RecycleImageView(Context context, AttributeSet attrs, int defStyle) {
		super(context, attrs, defStyle);
	}

	@Override
	public void setImageDrawable(Drawable drawable) {
		Drawable previousDrawable = getDrawable();
		super.setImageDrawable(drawable);
		
		//显示新的drawable
		notifyDrawable(drawable, true);

		//回收之前的图片
		notifyDrawable(previousDrawable, false);
	}

	@Override
	protected void onDetachedFromWindow() {
		//当View从窗口脱离的时候,清除drawable
		setImageDrawable(null);

		super.onDetachedFromWindow();
	}

	/**
	 * 通知该drawable显示或者隐藏
	 * 
	 * @param drawable
	 * @param isDisplayed
	 */
	public static void notifyDrawable(Drawable drawable, boolean isDisplayed) {
		if (drawable instanceof RecycleBitmapDrawable) {
			((RecycleBitmapDrawable) drawable).setIsDisplayed(isDisplayed);
		} else if (drawable instanceof LayerDrawable) {
			LayerDrawable layerDrawable = (LayerDrawable) drawable;
			for (int i = 0, z = layerDrawable.getNumberOfLayers(); i < z; i++) {
				notifyDrawable(layerDrawable.getDrawable(i), isDisplayed);
			}
		}
	}

}
```

只需要用RecycleBitmapDrawable包装Bitmap对象，然后设置到ImageView上面就可以啦，具体的内存释放我们不需要管，是不是很方便呢？这是在Android2.3以及以下的版本管理Bitmap的内存。


## 在Android 3.0及以上管理Bitmap

由于在Android3.0及以上的版本中，Bitmap的像素数据也存储在Dalvik heap中，所以内存的管理就直接交给垃圾回收器了，我们并不需要手动的去释放内存，而今天讲的主要是**BitmapFactory.Options.inBitmap**的这个字段，假如这个字段被设置了，我们在解码Bitmap的时候，他会去重用inBitmap设置的Bitmap，减少内存的分配和释放，提高了应用的性能，**然而在Android 4.4之前，BitmapFactory.Options.inBitmap设置的Bitmap必须和我们需要解码的Bitmap的大小一致才行，在Android4.4以后，BitmapFactory.Options.inBitmap设置的Bitmap大于或者等于我们需要解码的Bitmap的大小就OK了**，我们先假设一个场景，还是在使用ListView，GridView去加载大量的图片，为了提高应用的效率，我们通常会做相对应的内存缓存和硬盘缓存，这里我们只说内存缓存，而内存缓存官方推荐使用LruCache, 注意LruCache只是起到缓存数据作用，并没有回收内存。一般我们的代码会这么写

```
package com.example.bitmap;

import java.lang.ref.SoftReference;
import java.util.Collections;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Set;

import android.annotation.TargetApi;
import android.graphics.Bitmap;
import android.graphics.Bitmap.Config;
import android.graphics.BitmapFactory;
import android.graphics.drawable.BitmapDrawable;
import android.os.Build;
import android.os.Build.VERSION_CODES;
import android.support.v4.util.LruCache;

public class ImageCache {
	private final static int MAX_MEMORY = 4 * 102 * 1024;
	private LruCache<String, BitmapDrawable> mMemoryCache;

	private Set<SoftReference<Bitmap>> mReusableBitmaps;

	private void init() {
		if (hasHoneycomb()) {
			mReusableBitmaps = Collections
					.synchronizedSet(new HashSet<SoftReference<Bitmap>>());
		}

		mMemoryCache = new LruCache<String, BitmapDrawable>(MAX_MEMORY) {

			/**
			 * 当保存的BitmapDrawable对象从LruCache中移除出来的时候回调的方法
			 */
			@Override
			protected void entryRemoved(boolean evicted, String key,
					BitmapDrawable oldValue, BitmapDrawable newValue) {

				if (hasHoneycomb()) {
					mReusableBitmaps.add(new SoftReference<Bitmap>(oldValue
							.getBitmap()));
				}
			}

		};
	}

	
	/**
	 * 从mReusableBitmaps中获取满足 能设置到
	 * BitmapFactory.Options.inBitmap上面的Bitmap对象
	 * @param options
	 * @return
	 */
	protected Bitmap getBitmapFromReusableSet(BitmapFactory.Options options) {
		Bitmap bitmap = null;

		if (mReusableBitmaps != null && !mReusableBitmaps.isEmpty()) {
			synchronized (mReusableBitmaps) {
				final Iterator<SoftReference<Bitmap>> iterator 
							= mReusableBitmaps.iterator();
				Bitmap item;

				while (iterator.hasNext()) {
					item = iterator.next().get();

					if (null != item && item.isMutable()) {
						if (canUseForInBitmap(item, options)) {
							bitmap = item;
							iterator.remove();
							break;
						}
					} else {
						iterator.remove();
					}
				}
			}
		}
		return bitmap;
	}

	/**
	 * 判断该Bitmap是否可以设置到BitmapFactory.Options.inBitmap上
	 * 
	 * @param candidate
	 * @param targetOptions
	 * @return
	 */
	@TargetApi(VERSION_CODES.KITKAT)
	public static boolean canUseForInBitmap(Bitmap candidate,
			BitmapFactory.Options targetOptions) {

		// 在Anroid4.4以后，如果要使用inBitmap的话，
		// 只需要解码的Bitmap比inBitmap设置的小就行了，对inSampleSize
		// 没有限制
		if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
			int width = targetOptions.outWidth / targetOptions.inSampleSize;
			int height = targetOptions.outHeight / targetOptions.inSampleSize;
			int byteCount = width * height
					* getBytesPerPixel(candidate.getConfig());
			return byteCount <= candidate.getAllocationByteCount();
		}

		// 在Android
		// 4.4之前，如果想使用inBitmap的话，解码的Bitmap必须
		// 和inBitmap设置的宽高相等，且inSampleSize为1
		return candidate.getWidth() == targetOptions.outWidth
				&& candidate.getHeight() == targetOptions.outHeight
				&& targetOptions.inSampleSize == 1;
	}

	/**
	 * 获取每个像素所占用的Byte数
	 * 
	 * @param config
	 * @return
	 */
	public static int getBytesPerPixel(Config config) {
		if (config == Config.ARGB_8888) {
			return 4;
		} else if (config == Config.RGB_565) {
			return 2;
		} else if (config == Config.ARGB_4444) {
			return 2;
		} else if (config == Config.ALPHA_8) {
			return 1;
		}
		return 1;
	}

	@TargetApi(VERSION_CODES.HONEYCOMB)
	public static boolean hasHoneycomb() {
		return Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB;
	}

}
```

上面只是一些事例性的代码，将从LruCache中移除的BitmapDrawable对象的弱引用保存在一个set中，然后从set中获取满足BitmapFactory.Options.inBitmap条件的Bitmap对象用来提高解码Bitmap性能，使用如下

```
public static Bitmap decodeSampledBitmapFromFile(String filename,  
            int reqWidth, int reqHeight) {  
  
        final BitmapFactory.Options options = new BitmapFactory.Options();  
        ...  
        BitmapFactory.decodeFile(filename, options);  
        ...  
  
        // If we're running on Honeycomb or newer, try to use inBitmap.  
        if (ImageCache.hasHoneycomb()) {  
             options.inMutable = true;  
  
                if (cache != null) {  
                    Bitmap inBitmap = cache.getBitmapFromReusableSet(options);  
  
                    if (inBitmap != null) {  
                        options.inBitmap = inBitmap;  
                    }  
                }  
        }  
        ...  
        return BitmapFactory.decodeFile(filename, options);  
    }
```

通过这篇文章你是不是对Bitmap对象有了更进一步的了解，在应用加载大量的Bitmap对象的时候，如果你做到上面几点，我相信应用发生OutOfMemoryError的概率会很小，并且性能会得到一定的提升，我经常会看到一些同学在评价一个图片加载框架好不好的时候，比较片面的以自己使用过程中是否发生OutOfMemoryError来定论，当然经常性的发生OutOfMemoryError你应该先检查你的代码是否存在问题，一般一些比较成熟的框架是不存在很严重的问题，毕竟它也经过很多的考验才被人熟知的，今天的讲解就到这里了，有疑问的同学可以在下面留言！