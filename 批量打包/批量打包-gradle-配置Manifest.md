# 批量打包-gradle-配置Manifest

节选来源:[stormzhang.com](http://stormzhang.com/devtools/2015/01/15/android-studio-tutorial6/)

由于国内Android市场众多渠道，为了统计每个渠道的下载及其它数据统计，就需要我们针对每个渠道单独打包，如果让你打几十个市场的包岂不烦死了，不过有了Gradle，这再也不是事了。

## 友盟多渠道打包
废话不多说，以友盟统计为例，在AndroidManifest.xml里面会有这么一段：

```
<meta-data
    android:name="UMENG_CHANNEL"
    android:value="Channel_ID" />
```

里面的Channel_ID就是渠道标示。我们的目标就是在编译的时候这个值能够自动变化。

* 第一步 在AndroidManifest.xml里配置PlaceHolder

```
<meta-data
    android:name="UMENG_CHANNEL"
    android:value="${UMENG_CHANNEL_VALUE}" />
```

* 第二步 在build.gradle设置productFlavors

```
android {  
    productFlavors {
        xiaomi {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "xiaomi"]
        }
        _360 {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "_360"]
        }
        baidu {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "baidu"]
        }
        wandoujia {
            manifestPlaceholders = [UMENG_CHANNEL_VALUE: "wandoujia"]
        }
    }  
}
```

或者批量修改为渠道名

```
android {  
    productFlavors {
        xiaomi {}
        _360 {}
        baidu {}
        wandoujia {}
    }  

    productFlavors.all { 
        flavor -> flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name] 
    }
}
```

很简单清晰有没有？直接执行 `./gradlew assembleRelease` ， 然后就可以静静的喝杯咖啡等待打包完成吧。


