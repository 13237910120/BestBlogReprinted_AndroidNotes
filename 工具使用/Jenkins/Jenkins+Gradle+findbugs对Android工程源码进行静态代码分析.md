# Jenkins+Gradle+findbugs对Android工程源码进行静态代码分析

来源:[测试蜗牛，一步一个脚印](http://blog.csdn.net/hwhua1986/article/details/49278467)

## 环境说明

```
Gradle 2.6.
OS：windows server 2008
Jenkins 1.620
Findbugs 3.0.1
```

## 前提

Jenkins需要提前安装好FindBugs Plug-in插件

## 一、Jenkins配置如下：

* 1、新建job

![](8/1.png)

* 2、配置svn

![](8/2.png)

3、配置构建操作

先配置build构建操作

![](8/3.png)

为什么要配置build任务呢？从下图的findbugs的task任务可以看出findbugs要检查class文件，class文件需要编译java文件才能生成。

![](8/4.png)

再配置findbugs检查任务。如下图：
 
![](8/5.png)

备注：

> Tasks指的是build.gradle里面的task名称<br/>
> 配置info参数是用来看调试日志，也可以配置debug级别。主要用来查看构建失败的原因。

* 4、配置分析报告

![](8/6.png)

## 二、gradle.build的配置如下
 
* 1、添加依赖

```
buildscript {
   repositories {
               mavenCentral()
    }
   dependencies {
       classpath 'com.Android.tools.build:gradle:1.0.0+'
    }
}
```

备注：版本包可以通过中央仓库[http://mvnrepository.com/artifact/]http://mvnrepository.com/artifact/)查看，如图

![](8/7.png)

* 2、增加findbugs的task

```
apply plugin: "findbugs"
 
repositories {
  mavenCentral()
}
 
task findbugs(type: FindBugs) {
  //toolVersion = "3.0.1"
       ignoreFailures= true
       effort= "max"
       reportLevel= "low"
   classes = files("$project.buildDir/intermediates/classes")
       source= fileTree('build/intermediates/classes/debug/com/sn/')
       classpath= files()
       reports{
   xml {
     destination "build/findbugs.xml"
    }
  }
}
```

## 三、构建结果查询

![](8/8.png)
![](8/9.png)

## 四、build.gradle的所有代码如下

```
buildscript {
    repositories {
		 mavenCentral()
    }
    dependencies {
        classpath  'com.android.tools.build:gradle:1.0.0+'
		//classpath  'io.fabric.tools:gradle:1.+'
		//classpath  'com.google.code.findbugs:findbugs:3.0.1'
		//classpath  'com.puppycrawl.tools:checkstyle:6.11.2'
		//classpath  'net.sourceforge.pmd:pmd:5.4.0'
    }
}
apply plugin: 'android'

dependencies {
    compile fileTree(dir: 'libs', include: '*.jar')
}

android {
    compileSdkVersion 20
    buildToolsVersion "20.0.0"

	//忽略编码错误
	lintOptions {  
	     abortOnError false  
	}  

	//设置版本号
	defaultConfig {
        versionCode 1
        versionName "1.0"
        minSdkVersion 8
        targetSdkVersion 18
    }
    
    //引用so包
	sourceSets{
        main{
            jniLibs.srcDir(['libs'])
            jniLibs.srcDir(['obj'])
        }
    }

	//设置编译编码
	tasks.withType(JavaCompile) { 
		options.encoding = 'UTF-8' 
	}
	
	//autograph
    signingConfigs{
        //keystore info
        myConfig {
            storeFile file("bgkey")
            storePassword "sinocare@ydyl"
            keyAlias "com.sn.bloodglucose"
            keyPassword "sinocare@ydyl"
        }
    }

  //混淆
    buildTypes{  
        release{  
            signingConfig signingConfigs.myConfig  
            minifyEnabled false  
        }  
    }  

    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }

        // Move the tests to tests/java, tests/res, etc...
        instrumentTest.setRoot('tests')

        // Move the build types to build-types/<type>
        // For instance, build-types/debug/java, build-types/debug/AndroidManifest.xml, ...
        // This moves them out of them default location under src/<type>/... which would
        // conflict with src/ being used by the main source set.
        // Adding new build types or product flavors should be accompanied
        // by a similar customization.
        debug.setRoot('build-types/debug')
        release.setRoot('build-types/release')
    }
}

apply plugin: "findbugs"

repositories {
  mavenCentral()
}

task findbugs(type: FindBugs) {
   //toolVersion = "2.0.1"
	ignoreFailures = true
	effort = "max"
	reportLevel = "low"
    classes = files("$project.buildDir/intermediates/classes")
	source = fileTree('build/intermediates/classes/debug/com/sn/')
	classpath = files()
	reports {
    xml {
      destination "build/findbugs.xml"
    }
  }
}


apply plugin: "checkstyle"

repositories {
  mavenCentral()
}
task checkstyle(type: Checkstyle) {
   ignoreFailures = true
	//config = files("build/config/checkstyle/checkstyle.xml")
	source = fileTree('build/intermediates/classes/debug/com/sn/')
	classpath = files()
	reports {
    xml {
      destination "build/checkstyle-result.xml"
    }
  }
}


apply plugin: "pmd"

repositories {
  mavenCentral()
}
task pmd(type: Pmd) {
    ignoreFailures = true
   	source = fileTree('src/com/sn/')
	//ruleSetConfig = resources.file("${project.rootDir}/config/pmd/PmdRuleSets.xml")
	//ruleSetFiles = files("config/pmd/PmdRuleSets.xml")
	ruleSetFiles = files("${project.rootDir}/config/pmd/PmdRuleSets.xml")
	ruleSets = ["java-android"]
	reports {
    xml {
      destination "build/pmd.xml"
    }
  }
}
```


## 备注

FindBugs Gradle任务详细配置:

* [https://docs.gradle.org/current/dsl/org.gradle.api.plugins.quality.FindBugsExtension.html#org.gradle.api.plugins.quality.FindBugsExtension:excludeFilter](https://docs.gradle.org/current/dsl/org.gradle.api.plugins.quality.FindBugsExtension.html#org.gradle.api.plugins.quality.FindBugsExtension:excludeFilter)

* [https://docs.gradle.org/current/userguide/findbugs_plugin.html#useFindBugsPlugin](https://docs.gradle.org/current/userguide/findbugs_plugin.html#useFindBugsPlugin)

