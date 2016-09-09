# 关于android webview读取js全局变量或者函数返回值

来源:[http://blog.csdn.net/ldwtill/article/details/9455219](http://blog.csdn.net/ldwtill/article/details/9455219)

**背景：**借助现有接口技术，js可以执行原生java代码中的方法，可以得到方法的返回值，可以让原生java代码在主线程中动态的操作UI；但是借助该接口，原生java代码，采用`webview.loadUrl("javascript: JsFunctionName")`，只能做到执行js中的方法，如果想获取js中定义的全局变量，或者获取某个js函数的返回值，这种方式无法做到，webview也没有提供别的函数来可供使用。

**分析：**为了实现该功能，我们分析application framework的源代码发现，从webview类`loadurl()`方法一路追踪，最终在WebViewCore.java中找到如下代码：

```
private native voidpassToJs(int frame, int node, int x, int y, int gen,
            String currentText, int keyCode, intkeyValue, boolean down,
            boolean cap, boolean fn, boolean sym);
```

在BrowserFrame中，追踪到：

```
private native voidnativeAddJavascriptInterface(int nativeFramePointer,
            Object obj, String interfaceName);
```

至此我们知道Android的webview实现，使用的是开源的webkit浏览器内核，该内核是用c语言(webcore)和c++语言(jscore)实现的，android的webview底层实现最终是调用的webkit内核代码，如果该内核提供了直接读取js全局变量或者函数返回值的方法，那么我们可以使用JNI(JavaNative Interface)的方式来读取出来。

**方案：**

* （1）反射读取方式

在android.webkit包中有个BrowserFrame私有类，该类中有个Native方法：

```
public native StringstringByEvaluatingJavaScriptFromString(String script);
```

这个和苹果中的类似：

```
Public NSStringstringByEvaluatingJavaScriptFromString(NSString script);
```

虽然该类是私有的，但是我们可以利用反射技术来执行这个方法，从而取得js全局变量和函数返回值；

步骤：

1、扩展WebView，派生出MyWebView类，添加

`public String stringByEvaluatingJavaScriptFromString(Stringscript)`方法，该方法体中最终利用反射技术实现；

2、修改布局中的WebView为`com.appeon.test.MyWebView`类型；

3、  在页面load完成的情况下，编码取得JS变量或函数返回值；

* （2）JNI读取方式

除了采用反射方式能访问到私有类BrowserFrame中的`stringByEvaluatingJavaScriptFromString`方法之外，采用JNI技术，也能做到；下面我们采用JNI技术来实现20.4.1中的MyWebView类。

原理：`java->C->java`，具体到这里就是mywebview.java调用bridge.c，bridge.c再调用*BrowserFrame.java*

*（3）扩展webkit方式

直接扩展WebCore,扩展JSBridge，实现JS的数据类型到JAVA数据类型的转换,确切的说是相互转换，这里面JAVA部分可以用反射机制来做到。

*（4）Plug方式

采用插件的方式实现。

*（5）使用android调用js后，通过js调用android本地方法，从而传入值的方式。

代码如下：

```
webView.loadUrl("javascript:androidGetInfo()");//android调用当前页面的androidGetInfo()方法。
```

页面的js：

```
<script>

      var title = '${article.title}';

      var articleId = '${article.id}';

      var articleType = '${article.type}';

      function androidGetInfo(){
	      //调用android 中的getInfo方法。
          window.tlsj.getInfo(title,articleType);
      }

</script>
```

在android手机端代码如下：

```
webView.addJavascriptInterface(newObject() {
	public void getInfo(String _title,String _articleType) {

             title = _title;

             articleType = _articleType;

             Message msg = new Message();

             handler.sendMessage(msg);

          }

      }, "tlsj");
```
 

由此即可将js中的全局变量或者函数返回值传入android手机端

**分析：**

* 反射读取方式存在一定的不稳定性，如当内部不开放函数或变量名发生变化时，反射就会出现问题。

* JNI读取方式需要程序员对c++有一定了解。

* 使用android调用js后，通过js调用android本地方法不够灵活。


**备注：**

js给document设置变量:`document.xxx=xxx;`