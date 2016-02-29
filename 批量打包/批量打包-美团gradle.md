# 批量打包-美团gradle

来源:[美团](http://tech.meituan.com/mt-apk-adaptation.html)

## 概述

前一篇文章([美团Android自动化之旅—生成渠道包](http://tech.meituan.com/mt-apk-packaging.html))介绍了Android中几种生成渠道包的方式，基本解决了打包慢的问题。

但是，随着渠道越来越多，不同渠道对应用的要求也不尽相同。例如，有的渠道要求美团客户端的应用名为美团，有的渠道要求应用名为美团团购。又比如，有些渠道要求应用不能使用第三方统计工具（如flurry）。总之，每次打包都需要对这些渠道进行适配。

之前的做法是为每个需要适配的渠道创建一个Git分支，发版时再切换到相应的分支，并合并主分支的代码。适配的渠道比较少的话这种方式还可以接受，如果分支比较多，对开发人员来说简直就是噩梦。还好，自从有了Gradle flavor，一切都变得简单了。本文假定读者使用过Gradle，如果还不了解建议先阅读相关文档。

## Flavor

先来看build.gradle文件中的一段代码：

```
android {
    ....

    productFlavors {
        flavor1 {
            minSdkVersion 14
        }
    }
}
```

上例定义了一个flavor：flavor1，并指定了应用的minSdkVersion为14（当然还可以配置更多的属性，具体可参考相关文档）。与此同时，Gradle还会为该flavor关联对应的sourceSet，默认位置为`src/<flavorName>`目录，对应到本例就是`src/flavor1`。

接下来，要做的就是根据具体的需求在build.gradle文件中配置flavor，并添加必要的代码和资源文件。以flavor1为例，运行`gradle assembleFlavor1`命令既可生成所需的适配包。下面主要介绍美团团购Android客户端的一些适配案例。

## 案例
### 使用不同的包名
美团团购Android客户端之前有两个版本：手机版(**com.meituan.group**)和hd版(**com.meituan.group.hd**)，两个版本使用了不同的代码。目前hd版对应的代码已不再维护，希望能直接使用手机版的代码。解决该问题可以有多种方法，不过使用flavor相对比较简单，示例如下：

```
productFlavors {
    hd {
        applicationId "com.meituan.group.hd"
    }
}
```

上面的代码添加了一个名为hd的flavor，并指定了应用的包名为**com.meituan.group.hd**，运行`gradle assembleHd`命令即可生成hd适配包。

### 控制是否自动更新

美团团购Android客户端在启动时会默认检查客户端是否有更新，如果有更新就会提示用户下载。但是有些渠道和应用市场不允许这种默认行为，所以在适配这些渠道时需要禁止自动更新功能。

解决的思路是提供一个配置字段，应用启动的时候检查该字段的值以决定是否开启自动更新功能。使用flavor可以完美的解决这类问题。

Gradle会在generateSources阶段为flavor生成一个BuildConfig.java文件。BuildConfig类默认提供了一些常量字段，比如应用的版本名（VERSION_NAME），应用的包名（PACKAGE_NAME）等。更强大的是，开发者还可以添加自定义的一些字段。下面的示例假设wandoujia市场默认禁止自动更新功能：

```
android {
    defaultConfig {
        buildConfigField "boolean", "AUTO_UPDATES", "true"
    }

    productFlavors {
        wandoujia {
            buildConfigField "boolean", "AUTO_UPDATES", "false"
        }        
    }

}
```

上面的代码会在BuildConfig类中生成AUTO_UPDATES布尔常量，默认值为true，在使用wandoujia flavor时，该值会被设置成false。接下来就可以在代码中使用AUTO_UPDATES常量来判断是否开启自动更新功能了。最后，运行gradle assembleWandoujia命令即可生成默认不开启自动升级功能的渠道包，是不是很简单。


### 使用不同的应用名

最常见的一类适配是修改应用的资源。例如，美团团购Android客户端的应用名是美团，但有的渠道需要把应用名修改为美团团购；还有，客户端经常会和一些应用分发市场合作，需要在应用的启动界面中加上第三方市场的Logo，类似这类适配形式还有很多。
Gradle在构建应用时，会优先使用flavor所属dataSet中的同名资源。所以，解决思路就是在flavor的dataSet中添加同名的字符串资源，以覆盖默认的资源。下面以适配wandoujia渠道的应用名为美团团购为例进行介绍。

首先，在build.gradle配置文件中添加如下flavor：

```
android {
    productFlavors {
        wandoujia { 
        }
    }
}
```

上面的配置会默认src/wandoujia目录为wandoujia flavor的dataSet。

接下来，在src目录内创建wandoujia目录，并添加如下应用名字符串资源（`src/wandoujia/res/values/appname.xml`）：

```
<resources>
    <string name="app_name">美团团购</string>
</resources>
```

默认的应用名字符串资源如下（`src/main/res/values/strings.xml`）:

```
<resources>
    <string name="app_name">美团</string>
</resources>
```

最后，运行`gradle assembleWandoujia`命令即可生成应用名为美团团购的应用了。

### 使用第三方SDK

某些渠道会要求客户端嵌入第三方SDK来满足特定的适配需求。比如360应用市场要求美团团购Android客户端的精品应用模块使用他们提供的SDK。问题的难点在于如何只为特定的渠道添加SDK，其他渠道不引入该SDK。使用flavor可以很好的解决这个问题，下面以为qihu360 flavor引入`com.qihoo360.union.sdk:union:1.0` SDK为例进行说明：

```
android {
    productFlavors {
        qihu360 {
        }
    }
}
...
dependencies {
    provided 'com.qihoo360.union.sdk:union:1.0'
    qihu360Compile 'com.qihoo360.union.sdk:union:1.0'
}
```

上例添加了名为qihu360的flavor，并且指定编译和运行时都依赖`com.qihoo360.union.sdk:union:1.0`。而其他渠道只是在构建的时候依赖该SDK，打包的时候并不会添加它。

接下来，需要在代码中使用反射技术判断应用程序是否添加了该SDK，从而决定是否要显示360 SDK提供的精品应用。部分代码如下：

```
class MyActivity extends Activity {
    private boolean useQihuSdk;

    @override
    public void onCreate(Bundle savedInstanceState) {
        try {
            Class.forName("com.qihoo360.union.sdk.UnionManager");
            useQihuSdk = true;
        } catch (ClassNotFoundException ignored) {

        }
    }
}
```

最后，运行`gradle assembleQihu360`命令即可生成包含360精品应用模块的渠道包了。

## 总结
适配是一项dirty工作，尤其是适配的渠道比较多的时候。上面介绍了几种使用Gradle flavor进行适配的例子，基本解决了繁杂的适配工作。

## 参考资料
* [http://www.gradle.org/](http://www.gradle.org/)
* [Product flavors](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Product-flavors)