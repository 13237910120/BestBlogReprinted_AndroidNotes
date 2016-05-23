# Android动态加载dex技术初探

来源:[blog.csdn.net](http://blog.csdn.net/u013478336/article/details/50734108)

今天不忙，研究了下Android动态加载dex的技术，主要参考：

1、[http://www.cnblogs.com/over140/archive/2011/11/23/2259367.html](http://www.cnblogs.com/over140/archive/2011/11/23/2259367.html)

2、[http://www.fengyoutian.com/web/single/13](http://www.fengyoutian.com/web/single/13)

好歹算是跑通了。下面把实现过程与遇到的问题归纳下，方便后续查找使用。

## 一、综述

Android使用Dalvik虚拟机加载可执行程序，所以不能直接加载基于class的jar，而是需要将class转化为dex字节码，从而执行代码。优化后的字节码文件可以存在一个*.jar中，只要其内部存放的是*.dex即可使用。

将class的jar包转化为dex需要用到命令dx（在`*\android-sdk\build-tools\version[23.0.1]`或`*\android-sdk\platform-tools`下能找到）;命令使用方式为：`dx --dex --output=output.jar origin.jar`，该命令将包含class的origin.jar转化为包含dex的output.jar文件。

Android支持动态加载的两种方式是：DexClassLoader和PathClassLoader，DexClassLoader可加载jar/apk/dex，且支持从SD卡加载；PathClassLoader据说只能加载已经安装在Android系统内APK文件，此说法我尚未验证（[http://blog.csdn.net/quaful/article/details/6096951](http://blog.csdn.net/quaful/article/details/6096951)）；以下这一段是摘录：

//***********************************************************************************//

PathClassLoader 的限制要更多一些，它只能加载已经安装到 Android 系统中的 apk 文件，也就是`/data/app`目录下的 apk 文件。其它位置的文件加载的时候都会出现 ClassNotFoundException. 例如：

```
PathClassLoader cl = new PathClassLoader(jarFile.toString(), 
						"/data/app/", ClassLoader.getSystemClassLoader());
```

为什么有这个限制呢？我认为这其实是当前 Android 的一个 bug, 因为 PathClassLoader 会去读取`/data/dalvik-cache`目录下的经过 Dalvik 优化过的 dex 文件，这个目录的 dex 文件是在安装 apk 包的时候由 Dalvik 生成的。例如，如果包的名字是 com.qihoo360.test，Android 应用安装之后都保存在`/data/app`目录下，即`/data/app/com.qihoo360.test-1.apk`，那么`/data/dalvik-cache`目录下就会生成 `data@app@com.qihoo360.test-1.apk@classes.dex`文件。在调用 PathClassLoader 时，它就会按照这个规则去找 dex 文件，如果你指定的 apk 文件是`/sdcard/test.apk`，它按照这个规则就会去读`/data/dalvik-cache/sdcard@test.apk@classes.dex`文件，显然这个文件不会存在，所以 PathClassLoader 会报错。

//****************************************************************************//

## 二、实验步骤

新建一个Android工程，并进行如下操作：

* 2.1 定义一个接口IShowToast.java

```
package com.example.testdextoast;

import android.content.Context;

public interface IShowToast {
	public int showToast(Context context);
}
```

* 2.2 再定义一个简单实现ShowToastImpl.java

```
package com.example.testdextoast;

import android.content.Context;
import android.widget.Toast;

public class ShowToastImpl implements IShowToast {

	@Override
	public int showToast(Context context) {
		Toast.makeText(context, "我来自另一个dex文件", Toast.LENGTH_LONG).show();
		return 100;
	}
}
```

* 2.3 将且仅将ShowToastImpl.java导出为jar（Eclipse工程右键->Export->Java->Jar file），这时候我们得到一个包含class文件的jar，命名为origin.jar。（此处按文章1说法同时导出接口IShowToast.java会报错：java.lang.IllegalAccessError: Class ref in pre-verified class resolved to unexpected implementation，但是我测试了导出接口IShowToast.java的情况，可以正常使用，没有崩溃问题。）

* 2.4 在命令行下，进入dx所在目录，将jar文件拷到该目录下，执行dx --dex --output=output.jar origin.jar，得到内含class.dex的output.jar

* 2.5 将output.jar文件放到SD卡下adb push output.jar sdcard/output.jar（或者将jar内的dex抽出来放到SD卡亦可）

* 2.6 开始编写调用代码:新建调用的Android工程，将IShowToast.java放入该工程，注意：**此处一定不能改变此文件的包路径，否则dex中的实现类找不到接口，会报错**：

```
[plain] view plain copy print?java.lang.ClassNotFoundException: Didn't find class "com.example.testdextoast.ShowToastImpl" on path: DexPathList[[zip file "/storage/emulated/0/testtoast.jar"],nativeLibraryDirectories=[/vendor/lib, /system/lib]]  
     ...  
     Suppressed: java.lang.NoClassDefFoundError: Failed resolution of: Lcom/example/testdextoast/IShowToast;  
             ... 16 more  
     Caused by: java.lang.ClassNotFoundException: Didn't find class "com.example.testdextoast.IShowToast" on path: DexPathList[[zip file "/storage/emulated/0/testtoast.jar"],nativeLibraryDirectories=[/vendor/lib, /system/lib]]  
             ... 21 more  
             Suppressed: java.lang.ClassNotFoundException: Didn't find class "com.example.testdextoast.IShowToast" on path: DexPathList[[zip file "/data/app/com.example.testshowtoastdex-1/base.apk"],nativeLibraryDirectories=[/vendor/lib, /system/lib]]  
                     ... 22 more  
                     Suppressed: java.lang.ClassNotFoundException: com.example.testdextoast.IShowToast  
                             ... 23 more  
                     Caused by: java.lang.NoClassDefFoundError: Class not found using the boot class loader; no stack available  
     Suppressed: java.lang.ClassNotFoundException: Didn't find class "com.example.testdextoast.ShowToastImpl" on path: DexPathList[[zip file "/data/app/com.example.testshowtoastdex-1/base.apk"],nativeLibraryDirectories=[/vendor/lib, /system/lib]]  
             ... 15 more  
             Suppressed: java.lang.ClassNotFoundException: com.example.testdextoast.ShowToastImpl  
                     ... 16 more  
             Caused by: java.lang.NoClassDefFoundError: Class not found using the boot class loader; no stack available  
```

测试工程代码如下：

```
package com.example.testshowtoastdex;  
  
import java.io.File;  
  
import com.example.testdextoast.IShowToast;  
  
import dalvik.system.DexClassLoader;  
import android.app.Activity;  
import android.os.Bundle;  
import android.os.Environment;  
import android.util.Log;  
  
public class MainActivity extends Activity {  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
          
        File dexOutputDir = getDir("dex1", 0);  
        String dexPath = Environment.getExternalStorageDirectory().toString() + File.separator + "output.jar";  
        DexClassLoader loader = new DexClassLoader(dexPath,   
                dexOutputDir.getAbsolutePath(),   
                null, getClassLoader());  
        try {  
            Class clz = loader.loadClass("com.example.testdextoast.ShowToastImpl");  
            IShowToast impl = (IShowToast) clz.newInstance();  
            impl.showToast(this);  
        } catch (Exception e) {  
            Log.d("TEST111", "error happened", e);  
        }  
    }  
}  
```

此处需要注意DexClassLoader的四个参数：

**参数1 dexPath：**待加载的dex文件路径，如果是外存路径，一定要加上读外存文件的权限（`<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>`），否则会报与上面一样的错误，这点参考文章2中说这个权限可有可无是错误的。(**更正下：Android4.4 KitKat及以后的版本需要此权限，之前的版本不需要权限**)

**参数2 optimizedDirectory：**解压后的dex存放位置，此位置一定要是可读写且仅该应用可读写（安全性考虑），所以只能放在data/data下。本文`getDir("dex1", 0)`会在`/data/data/**package/`下创建一个名叫”app_dex1“的文件夹，其内存放的文件是自动生成output.dex；如果不满足条件，Android会报的错误为：

```
java.lang.IllegalArgumentException: 
	optimizedDirectory not readable/writable: /storage/sdcard0

java.lang.IllegalArgumentException: Optimized data directory /storage/sdcard0 
	is not owned by the current user. 
	Shared storage cannot protect your application from code injection attacks.
```

**参数3 libraryPath：**指向包含本地库(so)的文件夹路径，可以设为null

**参数4 parent：**父级类加载器，一般可以通过`Context.getClassLoader`获取到，也可以通过`ClassLoader.getSystemClassLoader()`取到。

## 三、执行结果

上述步骤完成后就能调用到ShowToastImpl中方法并展示一个Toast，动态加载dex的皮毛已经了解了。另外文章1谈到”在实践中发现，自己新建一个Java工程然后导出jar是无法使用的“，这点经测试是不正确了，在Java工程中导出的class也可以被转化为dex并加载。导出jar的操作步骤见上文。

```
package com.example.testdextoast;  
  
  
public interface IShowToast {  
    public String getToast();  
}  
```

```
public class ShowToastImpl implements IShowToast {

	@Override
	public String getToast() {
		return "我来自另一个dex文件";
	}

}
```

```
package com.example.testshowtoastdex;  

[java] view plain copy print?import java.io.File;  
import com.example.testdextoast.IShowToast;  
import dalvik.system.DexClassLoader;  
import android.app.Activity;  
import android.os.Bundle;  
import android.os.Environment;  
import android.util.Log;  
import android.widget.Toast;  
  
public class MainActivity extends Activity {  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
          
        File dexOutputDir = getDir("dex1", 0);  
        String dexPath = Environment.getExternalStorageDirectory().toString() + File.separator + "output.dex";  
        DexClassLoader loader = new DexClassLoader(dexPath,   
                dexOutputDir.getAbsolutePath(),   
                null, getClassLoader());  
        try {  
            Class clz = loader.loadClass("com.example.testdextoast.ShowToastImpl");  
            IShowToast impl = (IShowToast) clz.newInstance();  
            Toast.makeText(this, "来自Java包1：" + impl.getToast(), Toast.LENGTH_SHORT).show();  
        } catch (Exception e) {  
            Log.d("TEST111", "error happened", e);  
        }  
    }  
}  
```

这段代码弹出的Toast是：”来自Java包1：我来自另一个dex文件“，说明java工程也是可以导出class转为dex的。我估计原文意思是java工程生成的可执行jar文件不能使用。


## 四、补充

1、如何在不安装APK的情况下调用其内部的Activity？

http://blog.zhourunsheng.com/2011/09/%E6%8E%A2%E7%A7%98%E8%85%BE%E8%AE%AFandroid%E6%89%8B%E6%9C%BA%E6%B8%B8%E6%88%8F%E5%B9%B3%E5%8F%B0%E4%B9%8B%E4%B8%8D%E5%AE%89%E8%A3%85%E6%B8%B8%E6%88%8Fapk%E7%9B%B4%E6%8E%A5%E5%90%AF%E5%8A%A8%E6%B3%95/

核心思想是反射目标APK中的Activity，构造完成后将当前应用的Context传入到目标APK中使其具有上下文环境。