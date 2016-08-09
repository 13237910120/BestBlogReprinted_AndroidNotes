# Ubuntu64位系统无法使用Android命令问题解决

来源：

* [解决64位Ubuntu无法使用adb、aapt的32位兼容问题](http://www.linuxdiyf.com/linux/11928.html)
* [Cannot run program "/android-sdk-linux/aapt.exe": error=2, 没有那个文件或目录](http://blog.csdn.net/hunterno4/article/details/8920368)
* [Why can't I run the android emulator?](http://stackoverflow.com/questions/3631709/why-cant-i-run-the-android-emulator)

## 解决64位Ubuntu无法使用adb、aapt的32位兼容问题

 
Ubuntu从13.10就已经去除了对ia32-lib这一32位库的支持，使得很多基于32位库的应用无法正常使用，比如BCompare、adb、aapt等。
新版本的BCompare(4.0+)已经支持新架构的Ubuntu，这里不再赘述，我们只简单的给出adb和aapt兼容解决方案。

1、兼容adb

`sudo apt-get install lib32stdc++6`

2、兼容aapt

`sudo apt-get install zlib1g:i386`

adb和aapt是Android SDK的核心组件，解决这两个组件的兼容问题，就可以顺利的在64bit版本的Linux系统中进行相关开发了。

ps:测试基于ubuntu 15.04 64bit版本,可用。

相关文章：

* 在你的Ubuntu上安装ADB（Android Debugging Bridge）：[http://www.linuxdiyf.com/linux/8478.html](http://www.linuxdiyf.com/linux/8478.html)
* Ubuntu下安装ADB：[http://www.linuxdiyf.com/linux/6114.html](http://www.linuxdiyf.com/linux/6114.html)
* 64位Ubuntu用不了adb：[http://www.linuxdiyf.com/linux/3697.html](http://www.linuxdiyf.com/linux/3697.html)

## Cannot run program "/android-sdk-linux/aapt.exe": error=2, 没有那个文件或目录

在用ant编译打包android的apk文件时报错：`Execute failed: java.io.IOException: Cannot run program "/android-sdk-linux/aapt.exe": error=2,`没有那个文件或目录

首先，确定环境变量没有问题，谷歌之

**解决：**由于系统为Ubuntu 64位系统，而aapt工具需要32位库的支持才能运行
因此执行：`sudo apt-get install ia32-libs`安装32位库


安装好后仍不行，依然是这个报错，细想了下，linux系统没有exe这样的后缀，而`build.xml`是Windows上复制的，需要修改

```
<condition property="exe" value=".exe" else=""><os family="windows" /></condition>
<condition property="bat" value=".bat" else=""><os family="windows" /></condition>
<property name="aapt" value="${android_platform-tools}/aapt${exe}" />  
<property name="aidl" value="${android_platform-tools}/aidl${exe}" />  
<property name="dx" value="${android_platform-tools}/dx${bat}" />  
<property name="apk-builder" value="${android-tools}/apkbuilder${bat}" /> 
<property name="proguard-home" value="${android-tools}/proguard/lib"/> 
```

将build.xml做如上修改，根据不同平台做个判断，当在Windows系统中时，tools下的工具均带有exe、bat后缀，否则则为空，不带后缀。


在linux中终于不报错了，但在jenkins中构建时仍然报这个错,原因在于构建时，使用SVN上传到服务器中的代码中Linux中并不是使用的root用户权限，而是另一个用户的权限，当ant打包时会产生一些新的文件，而这些文件是root权限的,导致在编译过程中出现跨用户。

**解决：**在配置环境变量时确保不同用户均可以找到aapt，尽量让jenkins下的工作空间处于同一用户下，注意不同文件的文件权限。

```
#echo $ANDROID_HOME
#echo $JAVA_HOME
#echo $PATH            //查看当前用户环境变量
```

## Why can't I run the android emulator?

问题:

```
I have installed everything like I was told to by the android website 
	and all I keep getting after I create my avd is

"Failed to start emulator: Cannot run program 
"/home/christopher/Desktop/android-sdk-linux_86//tools/emulator": 
				java.io.IOException: error=2, No such file or directory".
Anybody got any ideas??? I'm running linux if that helps.

```

解决方案：

> If you're running a 64-bit system, you need to install ia32-libs:<br/>
> sudo apt-get install ia32-libs

------


> If you are running Ubuntu 13.10 x64 or the latest Linux Mint x64 then the ia32-libs package is not available anymore. The solution which worked for me without any problems is to:<br/>
>
> sudo apt-get install libc6-i386 lib32stdc++6 lib32gcc1 lib32ncurses5 lib32z1
>
> Hope this will help!

-----

Try this, for me work fine

```
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install libncurses5:i386 libstdc++6:i386 zlib1g:i386
```
