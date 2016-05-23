# Android插件化基础

来源:[Android插件化基础](http://104.236.134.90/2016/02/02/Android%E6%8F%92%E4%BB%B6%E5%8C%96%E5%9F%BA%E7%A1%80/)

说到Android插件化，简单来说就是如下的操作：

> * 开发者将插件代码封装成Jar或者APK。
> * 宿主下载或者从本地加载Jar或者APK到宿主中。
> * 将宿主调用插件中的算法或者Android特定的Class（如Activity）

第一步的封装Jar和APK。就不用多说了。 下面说说第二步，将Jar或者Apk加载到宿主当中来,使用ClassLoader，可以加载文件系统上的jar、dex、apk。从而将Class加载到宿主之中。 再通过反射进行调用。下面是个简单的例子：

[Demo地址](http://104.236.134.90/git@github.com:BlueBirdZhangHua/ClassLoaderDemo.git)

首先定义一个MyClassLoader继承ClassLoader：

```
public class MyClassLoader extends ClassLoader{
  private String mDirPath;
  public MyClassLoader(String dirPath) {
  	 //这里获取文件路径。 后面getClassData的时候和Class名字一起组成class的文件路径。
     mDirPath = dirPath;
  }
  @Override
  protected Class<?> findClass(String arg0) throws ClassNotFoundException {
  		//读取*.class文件内容，获得该class文件的字节码
        byte[] classData = getClassData(arg0);  
        if (classData == null) {
            throw new ClassNotFoundException();
        }
        //将class的字节码转换成Class类的实例
        return  defineClass(arg0, classData, 0, classData.length);
  }
}
```

上面通过重载findClass方法，我们读取class文件的字节码然后通过调用ClassLoader的defineClass方法来定义Class。
具体使用MyClassLoader的时候我们将文件地址以及类名传给MyClassLoader:

```
MyClassLoader myClassLoader = new MyClassLoader(dirPath);
Class loadedClass = myClassLoader.loadClass("MainTest");
```

这样就得到了Class。后面我们通过反射调用他的相应方法就实现了最简单的插件化。

```
Method method = loadedClass.getMethod("sayHellow", null);
method.invoke(object, null);
```

上面使用了自定义的ClassLoader来实现的加载，功能比较简单，只支持class的加载。

另外也可以通过使用系统的DexClassLoader来直接加载Jar,Apk。

```
File dexOutputDir = this.getDir("dex", 0);  
final String dexOutputPath = dexOutputDir.getAbsolutePath();  
ClassLoader localClassLoader = ClassLoader.getSystemClassLoader();  
DexClassLoader dexClassLoader = new DexClassLoader([Jar或者Apk的路径], dexOutputPath, null, localClassLoader);  
try {  
  Class<?> localClass = dexClassLoader.loadClass(className);  
  Constructor<?> localConstructor = localClass.getConstructor(new Class[] {});  
  Object instance = localConstructor.newInstance();  
} catch (Exception e) {  
  e.printStackTrace();  
}
```

怎么样？通过这样的操作，就可以做一个简单的代码逻辑的插件化了。将你的逻辑处理代码打包成Jar或者APK，放到服务器上。
客户端开启后检查是否有新的版本，如果有就后台下载，覆盖源文件（这样逻辑上应该先把旧的逻辑加载到内存中，这里可以有很多不同的策略）。

下次打开的时候，直接加载新下载的Jar或者APK就动态更新了内部逻辑。

上面只是简单的逻辑代码的插件化，Android资源以及其他具有特性的Class的插件化就复杂很多，处理方法也多种多样。

后面通过对DynamicLoadApk,DirectLoadApk以及ACDD进行分析，由简入深的体验下Android插件化的各种实现方式。