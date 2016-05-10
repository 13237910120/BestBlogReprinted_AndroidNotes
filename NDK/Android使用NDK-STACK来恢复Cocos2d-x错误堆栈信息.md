# Android使用NDK-STACK来恢复Cocos2d-x错误堆栈信息

来源:[blog.csdn.net](http://blog.csdn.net/enjoy517905407/article/details/38557275)

之前在网上偶然看到可以使用ndk-stack来恢复cocos2d-x的错误堆栈信息，这虽不能算是通过Android调试C++代码，可还是能定位到具体哪一行代码出错，这样比起之前用输出log来定位错误来说，已经算是是从地狱走到天堂的大门了。

好了，下面是具体操作，其实很简单。

1.首先把logcat中的错误信息复制保存在文本中作为ndk-stack的输入，比如H:\logcat.txt。复制信息时要从有一行*号的信息开始，即从

```
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
```

开始复制全部绿色的错误信息，然后粘贴在H:\logcat.txt中。

2.打开cmd,输入`$NDK/ndk-stack -sym $PROJECT_PATH/obj/local/armeabi -dump H:logcat.txt`

```
-dump选项将指定logcat保存为文件作为输入；

$NDK是NDK安装目录；

$PROJECT_PATH是项目路径。
```

3.最后是显示出C++的错误堆栈信息，从中可以找出错误代码的具体行号。