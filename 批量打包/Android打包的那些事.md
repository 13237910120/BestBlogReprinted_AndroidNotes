# Android打包的那些事

来源:[www.jayfeng.com](http://www.jayfeng.com/2015/11/07/Android%E6%89%93%E5%8C%85%E7%9A%84%E9%82%A3%E4%BA%9B%E4%BA%8B/)

使用gradle打包apk已经成为当前主流趋势，我也在这个过程中经历了各种需求，并不断结合gradle新的支持，一一改进。在此，把这些相关的东西记录，做一总结。

## 1. 替换AndroidManifest中的占位符
我想把其中的${app_label}替换为@string/app_name

```
android{
    defaultConfig{
        manifestPlaceholders = [app_label:"@string/app_name"]
    }
}
```

如果只想替换debug版本：

```
android{
    buildTypes {
        debug {
	          manifestPlaceholders = [app_label:"@string/app_name_debug"]
        }
        release {
        }
    }
}
```

更多的需求是替换渠道编号：

```
android{
    productFlavors {
        // 把dev产品型号的apk的AndroidManifest中的channel替换dev
        "dev"{
            manifestPlaceholders = [channel:"dev"]
        }
    }
}
```

## 2. 独立配置签名信息
对于签名相关的信息,直接写在gradle当然不好,特别是一些开源项目，可以添加到gradle.properties:

```
RELEASE_KEY_PASSWORD=xxxx
RELEASE_KEY_ALIAS=xxx
RELEASE_STORE_PASSWORD=xxx
RELEASE_STORE_FILE=../.keystore/xxx.jks
```

然后在build.gradle中引用即可：

```
android {
    signingConfigs {
        release {
            storeFile file(RELEASE_STORE_FILE)
            storePassword RELEASE_STORE_PASSWORD
            keyAlias RELEASE_KEY_ALIAS
            keyPassword RELEASE_KEY_PASSWORD
        }
    }
}
```

如果不想提交到版本库，可以添加到local.properties中，然后在build.gradle中读取。

## 3. 多渠道打包

多渠道打包的关键之处在于，定义不同的product flavor, 并把AndroiManifest中的channel渠道编号替换为对应的flavor标识：

```
android {
    productFlavors {
        dev{
            manifestPlaceholders = [channel:"dev"]
        }
        official{
            manifestPlaceholders = [channel:"official"]
        }
        // ... ...
        wandoujia{
            manifestPlaceholders = [channel:"wandoujia"]
        }
        xiaomi{
            manifestPlaceholders = [channel:"xiaomi"]
        }
        "360"{
            manifestPlaceholders = [channel:"360"]
        }
}
```

注意一点，这里的flavor名如果是数字开头，必须用引号引起来。

构建一下，就能生成一系列的Build Variant了:

```
devDebug
devRelease
officialDebug
officialRelease
wandoujiaDebug
wandoujiaRelease
xiaomiDebug
xiaomiRelease
360Debug
360Release
```

其中debug, release是gradle默认自带的两个build type, 下一节还会继续说明。

选择一个，就能编译出对应渠道的apk了。

## 4. 自定义Build Type
前面说到默认的build type有两种debug和release，区别如下：

## 4. 自定义Build Type
前面说到默认的build type有两种debug和release，区别如下：

```
// release版本生成的BuildConfig特性信息
public final class BuildConfig {
  public static final boolean DEBUG = false;
  public static final String BUILD_TYPE = "release";
}
// debug版本生成的BuildConfig特性信息
public final class BuildConfig {
  public static final boolean DEBUG = true;
  public static final String BUILD_TYPE = "debug";
}
```

现在有一种需求，增加一种build type，介于debug和release之间，就是和release版本一样，但是要保留debug状态（如果做过rom开发的话，类似于user debug版本），我们称为preview版本吧。
其实很简单：

```
android {
    signingConfigs {
        debug {
            storeFile file(RELEASE_STORE_FILE)
            storePassword RELEASE_STORE_PASSWORD
            keyAlias RELEASE_KEY_ALIAS
            keyPassword RELEASE_KEY_PASSWORD
        }
        preview {
            storeFile file(RELEASE_STORE_FILE)
            storePassword RELEASE_STORE_PASSWORD
            keyAlias RELEASE_KEY_ALIAS
            keyPassword RELEASE_KEY_PASSWORD
        }
        release {
            storeFile file(RELEASE_STORE_FILE)
            storePassword RELEASE_STORE_PASSWORD
            keyAlias RELEASE_KEY_ALIAS
            keyPassword RELEASE_KEY_PASSWORD
        }
    }

    buildTypes {
        debug {
            manifestPlaceholders = [app_label:"@string/app_name_debug"]
        }
        release {
            manifestPlaceholders = [app_label:"@string/app_name"]
        }
        preview{
            manifestPlaceholders = [app_label:"@string/app_name_preview"]
        }
    }
}
```

另外，build type还有一个好处，如果想要一次性生成所有的preview版本，执行assemblePreview即可，debug和releae版本同理。

## 5. build type中的定制参数
上面我们在不同的build type替换${app_label}为不同的字符串，这样安装到手机上就能明显的区分出不同build type的版本。
除此之外，可能还可以配置一些参数，我这里列几个我在工作中用到的：

```
android {
        debug {
            manifestPlaceholders = [app_label:"@string/app_name_debug"]
            applicationIdSuffix ".debug"
            minifyEnabled false
            signingConfig signingConfigs.debug
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        release {
            manifestPlaceholders = [app_label:"@string/app_name"]
            minifyEnabled true
            shrinkResources true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        preview{
            manifestPlaceholders = [app_label:"@string/app_name_preview"]
            applicationIdSuffix ".preview"
            debuggable true // 保留debug信息
            minifyEnabled true
            shrinkResources true
            signingConfig signingConfigs.preview
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

这些都用的太多了，稍微解释一下：

```
// minifyEnabled 混淆处理
// shrinkResources 去除无用资源
// signingConfig 签名
// proguardFiles 混淆配置
// applicationIdSuffix 增加APP ID的后缀
// debuggable 是否保留调试信息
// ... ...
```

## 6. 多工程全局配置
随着产品渠道的铺开，往往一套代码需要支持多个产品形态，这就需要抽象出主要代码到一个Library，然后基于Library扩展几个App Module。

相信每个module的build.gradle都会有这个代码：

```
android {
    compileSdkVersion 22
    buildToolsVersion "23.0.1"

    defaultConfig {
        minSdkVersion 10
        targetSdkVersion 22
        versionCode 34
        versionName "v2.6.1"
    }
}
```

当升级sdk、build tool、target sdk等，几个module都要更改，非常的麻烦。最重要的是，很容易忘记，最终导致app module之间的差异不统一，也不可控。

强大的gradle插件在1.1.0支持全局变量设定，一举解决了这个问题。

先在project的根目录下的build.gradle定义ext全局变量:

```
ext {
    compileSdkVersion = 22
    buildToolsVersion = "23.0.1"
    minSdkVersion = 10
    targetSdkVersion = 22
    versionCode = 34
    versionName = "v2.6.1"
}
```

然后在各module的build.gradle中引用如下：

```
android {
    compileSdkVersion rootProject.ext.compileSdkVersion
    buildToolsVersion rootProject.ext.buildToolsVersion

    defaultConfig {
        applicationId "com.xxx.xxx"
        minSdkVersion rootProject.ext.minSdkVersion
        targetSdkVersion rootProject.ext.targetSdkVersion
        versionCode rootProject.ext.versionCode
        versionName rootProject.ext.versionName
    }
}
```

然后每次修改project级别的build.gradle即可实现全局统一配置。

## 7. 自定义导出的APK名称
默认android studio生成的apk名称为app-debug.apk或者app-release.apk，当有多个渠道的时候，需要同时编出50个渠道包的时候，就麻烦了，不知道谁是谁了。

这个时候，就需要自定义导出的APK名称了，不同的渠道编出的APK的文件名应该是不一样的。

```
android {
    // rename the apk with the version name
    applicationVariants.all { variant ->
        variant.outputs.each { output ->
            output.outputFile = new File(
                    output.outputFile.parent,
                    "ganchai-${variant.buildType.name}-${variant.versionName}-${variant.productFlavors[0].name}.apk".toLowerCase())
        }
    }
}
```

当apk太多时，如果能把apk按debug，release，preview分一下类就更好了（事实上，对于我这样经常发版的人，一编往往就要编四五十个版本的人，debug和release版本全混在一起没法看，必须分类），简单：

```
android {
    // rename the apk with the version name
    // add output file sub folder by build type
    applicationVariants.all { variant ->
        variant.outputs.each { output ->
            output.outputFile = new File(
                    output.outputFile.parent + "/${variant.buildType.name}",
                    "ganchai-${variant.buildType.name}-${variant.versionName}-${variant.productFlavors[0].name}.apk".toLowerCase())
        }
    }
}
```

现在生成了类似于ganchai-dev-preview-v2.4.0.0.apk这样格式的包了，preview的包自然就放在preview的文件夹下，清晰明了。

## 8. 混淆技巧
混淆能让反编译的代码可读性变的很差，而且还能显著的减少APK包的大小。

### 1). 第一个技巧

相信很多朋友对混淆都觉得麻烦，甚至说，非常乱。因为添加混淆规则需要查询官方说明文档，甚至有的官方文档还没说明。当你引用了太多库后，添加混淆规则将使一场噩梦。
这里介绍一个技巧，不用查官方文档，不用逐个库考虑添加规则。
首先，除了默认的混淆配置(android-sdk/tools/proguard/proguard-android.txt), 自己的代码肯定是要自己配置的：

```
 ## 位于module下的proguard-rules.pro
#####################################
######### 主程序不能混淆的代码 #########
#####################################

-dontwarn xxx.model.**
-keep class xxx.model.** { *; }

 ## 等等，自己的代码自己清楚

#####################################
########### 不优化泛型和反射 ##########
#####################################

-keepattributes Signature
```

接下来是麻烦的第三方库，一般来说，如果是极光推的话，它的包名是cn.jpush, 添加如下代码即可：

```
-dontwarn cn.jpush.**
-keep class cn.jpush.** { *; }
```
其他的第三库也是如此，一个一个添加，太累！其实可以用第三方反编译工具（比如jadx：https://github.com/skylot/jadx ），打开apk后，一眼就能看到引用的所有第三方库的包名，把所有不想混淆或者不确定能不能混淆的，直接都添加又有何不可：

```
#####################################
######### 第三方库或者jar包 ###########
#####################################

-dontwarn cn.jpush.**
-keep class cn.jpush.** { *; }

-dontwarn com.squareup.**
-keep class com.squareup.** { *; }

-dontwarn com.octo.**
-keep class com.octo.** { *; }

-dontwarn de.**
-keep class de.** { *; }

-dontwarn javax.**
-keep class javax.** { *; }

-dontwarn org.**
-keep class org.** { *; }

-dontwarn u.aly.**
-keep class u.aly.** { *; }

-dontwarn uk.**
-keep class uk.** { *; }

-dontwarn com.baidu.**
-keep class com.baidu.** { *; }

-dontwarn com.facebook.**
-keep class com.facebook.** { *; }

-dontwarn com.google.**
-keep class com.google.** { *; }

 ## ... ...
```

### 2). 第二个技巧

一般release版本混淆之后，像友盟这样的统计系统如果有崩溃异常，会记录如下：

```
java.lang.NullPointerException: java.lang.NullPointerException
	at com.xxx.TabMessageFragment$7.run(Unknown Source)
```

这个Unknown Source是很要命的，排除错误无法定位到具体行了，大大降低调试效率。

当然，友盟支持上传Mapping文件，可帮助定位，mapping文件的位置在：

```
project > module
        > build > outputs > {flavor name} > {build type} > mapping.txt
```

如果版本一多，mapping.txt每次都要重新生成，还要上传，终归还是麻烦。

其实，在proguard-rules.pro中添加如下代码即可：

```
-keepattributes SourceFile,LineNumberTable
```

当然apk包会大那么一点点（我这里6M的包，大个200k吧），但是再也不用mapping.txt也能定位到行了，为了这种解脱，这个代价我个人觉得是值的，而且超值！

## 9. 动态设置一些额外信息
假如想把当前的编译时间、编译的机器、最新的commit版本添加到apk，而这些信息又不好写在代码里，强大的gradle给了我创造可能的自信：

```
android {
    defaultConfig {
        resValue "string", "build_time", buildTime()
        resValue "string", "build_host", hostName()
        resValue "string", "build_revision", revision()
    }
}
def buildTime() {
    return new Date().format("yyyy-MM-dd HH:mm:ss")
}
def hostName() {
    return System.getProperty("user.name") + "@" + InetAddress.localHost.hostName
}
def revision() {
    def code = new ByteArrayOutputStream()
    exec {
        commandLine 'git', 'rev-parse', '--short', 'HEAD'
        standardOutput = code
    }
    return code.toString()
}
```

上述代码实现了动态的添加了3个字符串资源: build_time、build_host、build_revision, 然后在其他地方可像如引用字符串一样使用如下：

```
// 在Activity里调用
getString(R.string.build_time)  // 输出2015-11-07 17:01
getString(R.string.build_host)  // 输出jay@deepin，这是我的电脑的用户名和PC名
getString(R.string.build_revision) // 输出3dd5823, 这是最后一次commit的sha值
```

这个地方，如何从命令行读取返回结果，很有意思。

其实这段代码来自我学习VLC源码时偶然看到，深受启发，不敢独享，特摘抄在此。

vlc源码及编译地址：[https://wiki.videolan.org/AndroidCompile](https://wiki.videolan.org/AndroidCompile)， 有兴趣可以过去一观。

## 10. 给自己留个”后门”: 点七下
为了调试方便，我们往往会在debug版本留一个显示我们想看的界面（记得之前微博的一个iOS版本就泄露了一个调试界面），如何进入到一个界面，我们可以仿照android开发者选项的方式，点七下才显示，我们来实现一个：

```
private int clickCount = 0;

private long clickTime = 0;

sevenClickView.setOnClickListener(new View.OnClickListener() {

    @Override
    public void onClick(View view) {
        if (clickTime == 0) {
            clickTime = System.currentTimeMillis();
        }
        if (System.currentTimeMillis() - clickTime > 500) {
            clickCount = 0;
        } else {
            clickCount++;
        }
        clickTime = System.currentTimeMillis();
        if (clickCount > 6) {
            // 点七下条件达到，跳到debug界面
        }
    }
});
```

release版本肯定是不能暴露这个界面的，也不能让人用am在命令行调起，如何防止呢，可以在release版本把这个debug界面的exported设为false。

## 11. 自动化构建
如何使用jenkins打包android和ios，并上传到蒲公英平台，这个可以参考我的另外一篇文章专门介绍: 《[使用jenkins自动化构建android和ios应用》](http://www.jayfeng.com/2015/10/22/%E4%BD%BF%E7%94%A8jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%9E%84%E5%BB%BAandroid%E5%92%8Cios%E5%BA%94%E7%94%A8/)，不过，这篇文章还没写完，实际上在公司里已经一直在用了，哪天心情好了总会写完的，这里不再赘述。

## 12. 小结
android打包因为groovy语言的强大，变的强大的同时必然也变的复杂，今天把我经历的这些门道拿出来说道一下，做一个小小的总结，后续有更新我还会添加。

