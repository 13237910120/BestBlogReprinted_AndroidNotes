# 更优雅的 Android 发布自动版本号方案

来源:[稀土掘金](http://gold.xitu.io/entry/56d7885defa6310054aecd5c/view?from=groupmessage&isappinstalled=0)

以前看到一些自动化版本号打包的文章。如果您的项目是用 Git 管理的，并且恰巧又是使用 Gradle 编译（应该绝大部分都是这样的了吧？），本文试图找到一种更加优雅的自动版本管理方法。

## 1 背景

我们都知道，Android 应用的版本管理是依赖 AndroidManifest.xml 中的两个属性：

* **android:versionCode**：版本号，是一个大于 0 的整数，相当于 Build Number，随着版本的更新，这个必须是递增的。大的版本号，覆盖更新小的版本号；
* **android:versionName**：版本名，是一个字符串，例如 "1.2.0"，这个是给人看的版本名，系统并不关心这个值，但是合理的版本名，对后期的维护和 bug 修复也非常重要。

在使用了 Android Studio 或者 Gradle 编译以后，我们通常是在`build.gradle`里面定义这两个值，如下：

```
android {  
    ...
    defaultConfig {
        ...
        versionCode 1
        versionName "1.0"
    }
}
```

## 2 自动版本号

在这篇文章中[6 tips to speed up your Gradle build](https://medium.com/@shelajev/6-tips-to-speed-up-your-gradle-build-3d98791d3df9) 发现了，可以使用 Git 中 commit 的数量来作为版本号（versionCode）。方案如下：

```
def cmd = 'git rev-list HEAD --first-parent --count'  
def gitVersion = cmd.execute().text.trim().toInteger()

android {  
  defaultConfig {
    versionCode gitVersion
  }
}
```

这里关键是这一行 git 命令`git rev-list HEAD --first-parent --count`，表示获取当前分支的 commit 数量。

这是一个绝妙的方案。因为在项目开发中，我们的往 Git 库中提交的 Commit 的数量应该是只增不减的（当然，在极少的情况下有例外），而且对应 Commit 的数量直接对应代码当前的版本状态，只要你做了代码修改，版本号就应该增加。有些解决方案中，每次 Build 就会增加一次版本号，个人感觉并不合适，如果是相同的代码，发布出去版本号应该保持一致，而不在于你编译多少次。

另外，有些人可能会担心，每次版本发布，可能会包含几百个新的 commit，这样的话 versionCode 会不会增长太快了，最后导致不够用了。其实，完全没有必要担心，versionCode 是 int 类型，最大值是 2^31-1，也就是 21 亿多，Android 源码中，改动最活跃的 framework/base 所有分支到目前为止也就 20 万多个 commit，所以完全够用了。

## 3 自动版本名

前面通过一条简单的命令实现了自动化的 *versionCode*，现在我们看怎么自动化 *versionName*。

在正常的发布流程中，在发布新版本的时候，都会在版本库中打 tag。一般情况下，tag 名就是版本名，而且也建议这么做，因为如果某个版本出现 bug，也可以正好 checkout 这个 tag 来查看代码。所以，现在的问题就是怎么自动获得 git 库中最新的最新 tag？原来，git 早就提供了命令`git describe`，它的功能就是获取从当期 commit 到距离它最近的 tag 的描述。默认都是 annoted tag，如果要指所有的类型的 tag 的话，就加`--tags`参数。

此命令的详细介绍在这里：[git-describe](http://git-scm.com/docs/git-describe)。举例一个简单的例子，假如你的当前代码状态如下：

```
--A--B-...-C-->
     |     |
   v1.0  v1.1
```

执行`git describe`的结果是：v1.1，如果是如下的情况：

```
--A--B-...-C--D-->
     |     |
   v1.0  v1.1
```

执行`git describe`的结果是：`v1.1-1-gXXXXXX`，其中 1 表示当前代码距离最近的 tag v1.1 一个 commit，最新的 commit 的 id 是`XXXXXX`。

可见，`describe`命令很好的描述了当前的分支的版本状态，我们可以直接使用这个它的输出作为版本号。在`build.gradle`中的使用如下：

```
def cmd = 'git describe --tags'  
def version = cmd.execute().text.trim()

android {  
  defaultConfig {
    versionName version
  }
}
```

这样就可以自动抽取 git 中的 tag 为版本名了。有些同学可能接受不了这样版本名字`v1.1-1-gXXXXXX`，这里也可以稍微做一些修改，使版本号更好看，如下：

```
def pattern = "-(\\d+)-g"  
def matcher = version =~ pattern

if (matcher) {  
    version = version.substring(0, matcher.start()) + "." + matcher[0][1]
} else {
    version = version + ".0"
}
```

这样的话，上面的版本名就变为了`v1.0.0`和`v1.1.1`了。

## 4 优化

前面的那篇文章中说了，为了尽可能减少 gradle 脚本的运算，提高开发速度，我们可以把这样的自动版本的计算放到 release 编译中去。最后的写法如下：

```
def gitVersionCode() {  
    def cmd = 'git rev-list HEAD --first-parent --count'
    cmd.execute().text.trim().toInteger()
}

def gitVersionTag() {  
    def cmd = 'git describe --tags'
    def version = cmd.execute().text.trim()

    def pattern = "-(\\d+)-g"
    def matcher = version =~ pattern

    if (matcher) {
        version = version.substring(0, matcher.start()) + "." + matcher[0][1]
    } else {
        version = version + ".0"
    }

    return version
}

android {  
    compileSdkVersion 23
    buildToolsVersion "23.0.2"

    defaultConfig {
        applicationId "com.race604.example"
        minSdkVersion 15
        targetSdkVersion 23
        versionCode 1
        versionName '1.0'
    }
    buildTypes {
        debug {
            // 为了不和 release 版本冲突
            applicationIdSuffix ".debug"
        }
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    applicationVariants.all { variant ->
        if (variant.buildType.name.equals('release')) {
            variant.mergedFlavor.versionCode = gitVersionCode()
            variant.mergedFlavor.versionName = gitVersionTag()
        }
    }
}
```

至此，结合 git 和 gradle 我们就实现了自动版本号。
