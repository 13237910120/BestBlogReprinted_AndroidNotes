# 官方-用Lint优化Android代码

来源:[http://android.jobbole.com/84931/?utm_source=blog.jobbole.com&utm_medium=relatedPosts](http://android.jobbole.com/84931/?utm_source=blog.jobbole.com&utm_medium=relatedPosts)

> 原文出处： [developer](https://developer.android.com/studio/write/lint.html)   译文出处：[开源中国](https://www.oschina.net/translate/improve-your-code-with-lint?print)

  
除了测试 Android 的应用程序是否满足功能要求外，确定你的代码没有结构问题也相当重要。代码架构不完善会影响 Android 应用程序的可靠性和运行效率，同时也会使代码更难维护。比如，如果你的XML资源包含未使用的命名文件，这不仅占用了空间，还会招致不必要的处理工作。其他的结构问题，如使用过时代码，或者使用了不被目标 API 版本支持的 API 调用，这都可能会导致代码无法正确运行。

## 概览

Android Studio 提供了一款名叫 Lint 的代码扫描工具，它可以帮助你轻松识别并纠正你的代码结构质量问题，而且并不需要运行app或编写测试用例。该工具检查到的每个问题都会生成一份包含描述信息和严重等级的报告，因此你可以快速优先处理需要做的紧急改进。你还可以配置一个问题的严重程度以忽略与你项目不相关的问题，或提升严重级别。这个工具有一个命令行接口，因此你可以轻易将它集成到你的自动测试进程中。

Lint 工具为潜在的bug和优化改进会检查你的Android项目代码文件中的正确性、安全性、性能、可用性、可访问性以及国际化。你可以从命令行或Android Studio中运行Lint。

**注意：**在Android Studio中，当你的代码在Android Studio中编译时额外的[IntelliJ code inspections](https://www.jetbrains.com/help/idea/2016.1/code-inspection.html)会运行以简化代码审查。

图1展示了Lint工具是如何处理应用程序的源代码文件的。

![](4/1.png)

*图1、 Lint工具的代码扫描工作流程*

* 应用程序源代码文件

源代码文件包含了那些构成你的Android工程的文件，包括 Java 和 XML文件，图标，以及 ProGuard 配置文件。

* lint.xml 文件

这是一个配置文件，你可以用它来指定任何你想要排除出去的Lint检查，还有就是对问题的严重级别进行自定义。

* Lint 工具

这是一个静态的代码扫描工具，你可以通过命令行或者 Android Studio来把它运行起来。Lint 工具会检查哪些可能会影响到Android应用程序质量及性能的结构方面的代码问题。我们强烈建议你在发布应用程序之前，先把Lint检测出来的错误修正。

* Lint检查的结果

你可以在控制台或者 Android Studio 的事件日志（Event Log）中查看来自Lint的运行结果。每一个问题都会以其在源代码文件中检测到错误的位置进行标识，并附上针对问题的描述信息。

Lint 工具是作为 Android SDK工具版本16或者更高版本的一部分，已经自动被安装好了的。

## 在 Android Studio 中运行Lint

在 Android Studio 中, 配置好的 Lint 及 IDE 检查会在你构建应用时自动运行。IDE检查是跟着 Lint 检查一起配置的，运行[IntelliJ 代码检查](https://www.jetbrains.com/help/idea/2016.1/code-inspection.html)就可以精简代码的审查工作。

注意: 要查看和修改检查的问题严重界别的话，可以使用 F**ile > Settings > Editor > Inspections** 菜单来打开检查配置（*Inspection Configuration*s）页，里面有一个支持检查项的清单。

使用 Android Studio 的话, 你还可以针对特定的构建变量运行 Lint 检查, 或者也可以是来自 build.gradle 文件的所有变量。需要将 lintOptions 属性添加到构建文件的 android 设置中。下面这段代码来自于一个 Gradle 构建文件，它显示了如何将 quiet 选项设置成 true，将 abortOnError 选项设置成 false。

```
android {
    lintOptions {
       // set to true to turn off analysis progress reporting by lint
       quiet true
       // if true, stop the gradle build if errors are found
       abortOnError false
       // if true, only report errors
       ignoreWarnings true
       }
       ...
    }
```

要在 Android Studio 中手动运行检查的话，可以从 application 或者 鼠标右键出现的菜单入手，选择 **Analyze > Inspect Code** 就可以了。这样会出现一个指定检查范围（*Specify Inspections Scope*）对话框，这样你就可以指定想要进行检查的范围和方面了。

每次检查的结果都会显示在检查工具窗口中。你可以通过将鼠标悬停在一个检查的错误上来显示内联的错误概要信息, 或者通过选定它来展示出错误完整的问题描述信息。

## 从命令行运行 lint

要针对一个工程目录中一系列文件运行 Lint，可以使用如下命令:

> lint [flags] <project directory>


例如，你可以输入如下命令来扫面 myproject 路径及其子路径下的文件。问题ID MissingPrefix 会告诉 Lint 只扫描那些丢失了 Android 命名空间前缀的 XML 属性。

> lint --check MissingPrefix myproject

要查看工具所支持的标识和命令行参数清单，使用如下命令：

> lint --help

### lint 输出示例

下面的示例展示了 Lint在针对一个叫做 Earthquake 的工程运行时，控制台的输出。

```
$ lint Earthquake
 
Scanning Earthquake: ...............................................................................................................................
Scanning Earthquake (Phase 2): .......
AndroidManifest.xml:23: Warning: <uses-sdk> tag appears after 
			<application> tag [ManifestOrder]
  <uses-sdk android:minSdkVersion="7" />
  ^
AndroidManifest.xml:23: Warning: <uses-sdk> tag should specify a 
	target API level (the highest verified version;
	 when running on later versions, 
	 compatibility behaviors may be enabled) with 
	 			android:targetSdkVersion="?" [UsesMinSdkAttributes]
  <uses-sdk android:minSdkVersion="7" />
 
  ^
res/layout/preferences.xml: Warning: The resource 
		R.layout.preferences appears to be unused [UnusedResources]
res: Warning: Missing density variation folders in res: 
			drawable-xhdpi [IconMissingDensityFolder]
0 errors, 4 warnings
```

上面的输出列明了这个工程有个四个警告，没有错误。三处警告(ManifestOrder, UsesMinSdkAttributes, and UnusedResources)是在工程的 AndroidManifest.xml文件中找到的。剩下的这处警告(IconMissingDensityFolder)是在 Preferences.xml 布局文件中找到的。

### 对 lint 进行配置

默认情况下，当你运行 Lint 扫描时，工具会检查所有 Lint 支持扫描的问题。你也可以限制 Lint 只对哪些问题进行检查，并制定这些问题的严重级别。例如，你可以不让 Lint 去检查哪些跟你的工程没多大关系的问题，还可以不让 Lint 报告那些严重级别较低的不怎么重要的问题。

你可以在不同的级别上对 Lint 检查进行配置：

* 全局地，针对整个工程
* 每个工程模块
* 每个生产模块
* 每个测试模块
* 每个打开的文件
* 每个类层级
* 每个版本控制系统（VCS）范围内

### 在 Android Studio 中对Lint进行配置

在你使用的是Android Studio时，其内置的 Lint 工具会对你的代码进行检查。你可以使用两种方式来查看警告和错误信息：

* 代码编辑器中的一个弹出文本. 当 Lint 发现一处问题时，它会让问题代码显示成高亮的黄色，或者针对更加严重的问题，让代码带上红色的下划线。
* 在你选择 **Analyze > Inspect Code** 所打开的 Lint 检查结果窗口中。

要对默认的 Lint 检查进行设置的话:

1. 在 Android Studio 中打开你的项目。
2. 选择 **File > Other Settings > Default Settings**。
3. 在默认选项（*Default Preferences* ）对话框中选择 **Editor > Inspections**。
4. 在 **Profile** 区域选择 **Default** 或者 **Project Default** 来设置[扫描范围](https://www.jetbrains.com/help/idea/2016.1/specify-inspection-scope-dialog.html?origin=old_help)是针对 Android Studio中的所有工程，或仅仅只针对对应的工程。
5. 将一个类别展开，然后按需修改 Lint 设置。你可以选择单个检查，或者整个所有的类别。
6. 点击 OK 。

要制作一份 Lint 检查的列表显示在检查结果（Inspection Results）窗口的话:

1. 在 Android Studio 中打开你的工程并选择工程中你想要进行测试那一部分。
2. 选择 **Analyze > Inspect Code**。
3. 在指定检查范围（*Specify Inspection Scope*）对话框中，选择要进行检查的[范围](https://www.jetbrains.com/help/idea/2016.1/specify-inspection-scope-dialog.html?origin=old_help)和方面。范围会指定你想要分析的文件，而方面则是指定你想要执行的那些 Lint 检查。
4. 如果你想要对 Lint 设置进行修改的话，就点击检查（*Inspections*）对话框中的 …，选择性的点击 **Manage** 来定义一个新的方面，指定你自己所想的Lint设置，然后点击 OK。在检查（ Inspections）对话框中，你可以进行字符串搜索来找到 Lint 检查。注意在检查窗口中如果修改了一个方面的 Lint设置，并不会改变上一个过程中所讲到的默认设置。不过它确实改变了在检查对话框中显示的那些设置。
5. 点击 **OK**。

检查结果会按类型组织在一起显示在检查结果（*Inspection Results*）窗口中。

### 对 lint 文件进行配置

你可以在 lint.xml 文件中指定 Lint 检查的参数。如果你是手动创建的这个文件，就把它放到Android工程的根路径下面。如果你是在 Android Studio 中对 Lint 参数进行的配置的话，lint.xml已经自动为你创建并添加到 Android 工程了。

lint.xml 文件包含了一个封闭起来的<lint>父标记，里面包含了一个或者多个 <issue> 子元素。每个 <issue> 被指定了一个唯一的id属性值，它是由 Lint 来定义的。

```
<?xml version="1.0" encoding="UTF-8"?>    
<lint>       
  <!-- list of issues to configure -->
</lint>
```

通过在 <issue>标记中的severity 进行设置，你可以不让 Lint 对某个问题进行检查，或者修改一个问题的严重等级。

提示: 想要查看 Lint 工具所支持的问题的完整清单及其对应的问题ID的话，可以运行 lint –list 命令。

#### 示例 lint.xml 文件

下面的示例显示了一个 lint.xml 文件的内容。

```
<?xml version="1.0" encoding="UTF-8"?>
<lint>   
  <!-- Disable the given check in this project -->   
  <issue id="IconMissingDensityFolder" severity="ignore" />    
  <!-- Ignore the ObsoleteLayoutParam issue in the specified files -->   
  <issue id="ObsoleteLayoutParam">        
    <ignore path="res/layout/activation.xml" />        
    <ignore path="res/layout-xlarge/activation.xml" />    
  </issue>   
  <!-- Ignore the UselessLeaf issue in the specified file -->   
  <issue id="UselessLeaf">        
    <ignore path="res/layout/main.xml" />    
  </issue>   
  <!-- Change the severity of hardcoded strings to "error" -->
  <issue id="HardcodedText" severity="error" />
</lint>
```

### 在 Java 和 XML 源代码文件中配置 Lint 检查

你可以在 Java 和 XML 文件中将 Lint 检查禁用掉。

提示: 如果你使用的是 Android Studio 的话，你可以使用 **File > Settings > Project Settings > Inspections** 功能来管理对 Java 和XML 源代码文件的 Lint 检查。

#### 在 Java 中配置 lint 检查

要在 Android 工程中将针对一个 Java 类或者方法的 Lint 检查禁用的话，向 Java 代码添加 `@SuppressLint` 注解就行了。

下面的示例展示了你如何才能够将 onCreate 方法中针对 NewApi 问题的 Lint 检查关闭掉。Lint 工具还是会在这个类的其它方法中对 NewApi 问题进行检查。

```
@SuppressLint("NewApi")
@Override
public void onCreate(Bundle savedInstanceState) {    
  super.onCreate(savedInstanceState);   
  setContentView(R.layout.main);
```

下面的示例显示了如何去关闭针对 FeedProvider类中的 ParserError 问题的 Lint 检查：

```
@SuppressLint("ParserError")
public class FeedProvider extends ContentProvider {
```

要限制在 Java 文件中针对所有 Lint 问题的检查，可以像下面这样使用 all 关键词：

```
@SuppressLint("all")
```

#### 在 XML 配置 lint 检查

你可以使用`tools:ignore`属性将针对 XML 文件特定区域的 Lint 检查禁用掉。为了让 Lint 工具认出这个属性，如下所示的命名空间必须被引入到你的 XML 文件中：

```
namespace xmlns:tools="http://schemas.android.com/tools"
```

下面的示例展示了你如何才能够将针对一个 XML 布局文件中的 <LinearLayout>元素的 UnusedResources 问题的 Lint 检查禁用掉。ignore 属性会由在其中声明了该属性的父元素其下的子元素所继承。在这个实例中，对于子元素 <TextView> 而言， Lint 检查也是被禁用的。


```
<LinearLayout
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  tools:ignore="UnusedResources" >    
  <TextView android:text="@string/auto_update_prompt" />
</LinearLayout>
```

要禁用多个问题，可以在一个逗号分隔的字符串中把问题都列出来。例如：

```
tools:ignore="NewApi,StringFormatInvalid"
```

要在一个XML元素中限制对所有 Lint 问题的检查，可以像下面这样使用 all 关键词：

```
tools:ignore="all
```

