# Jenkins+Gradle+checkstyle对Android工程源码进行静态代码分析

来源:[测试蜗牛，一步一个脚印](http://blog.csdn.net/hwhua1986/article/details/49278773)

## 环境说明

```
Gradle 2.6.
OS：windows server 2008
Jenkins 1.620
checkstyle 6.11.6
```

## 前提

Jenkins需要提前安装好Checkstyle Plug-in插件

## 一、Jenkins配置如下：

* 1、  新建job

![](7/1.png)

* 2、  配置svn

![](7/2.png)

* 3、  配置构建操作

![](7/3.png)

备注：
> Tasks指的是build.gradle里面的task名称<br/>
> 配置info参数是用来看调试日志，也可以配置debug级别。主要用来查看构建失败的原因。

* 4、  配置分析报告

![](7/4.png)

## 二、gradle.build的配置如下
 
* 1、添加checkstyle的依赖

```
buildscript {
   repositories {
               mavenCentral()
    }
   dependencies {
       classpath 'com.Android.tools.build:gradle:1.0.0+'
              //classpath  'io.fabric.tools:gradle:1.+'
              //classpath  'com.google.code.findbugs:findbugs:3.0.1'
              //classpath  'com.puppycrawl.tools:checkstyle:6.11.2'
              //classpath  'net.sourceforge.pmd:pmd:5.4.0'
    }
}
```

备注：版本包可以通过中央仓库（http://mvnrepository.com/artifact/）查看，如图

![](7/5.png)

版本列表：

![](7/6.png)

* 2、增加checkstyle的task
 
```
apply plugin: "checkstyle"
 
repositories {
  mavenCentral()
}

task checkstyle(type: Checkstyle) {
  //toolVersion = "2.0.1"
  ignoreFailures = true
       //config= files("$rootProject.projectDir/config/checkstyle/checkstyle.xml")
       source= fileTree('build/intermediates/classes/debug/com/sn/')
       classpath= files()
       reports{
   xml {
     destination "build/checkstyle-result.xml"
    }
  }
}
```

备注：其中检查规则文件`checkstyle.xml`需要创建，步骤如下

![](7/7.png)

Xml内容代码：

```
<?xml version="1.0"encoding="UTF-8"?>  
<!DOCTYPE module PUBLIC  
   "-//Puppy Crawl//DTD Check Configuration 1.3//EN"  
    "http://www.puppycrawl.com/dtds/configuration_1_3.dtd">  
<module name="Checker">  
<modulenamemodulename="FileTabCharacter"/>  
  <module name="TreeWalker">  
   <module name="UnusedImports"/>  
</module>  
```

## 三、构建结果查看

![](7/8.png)
![](7/9.png)

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


