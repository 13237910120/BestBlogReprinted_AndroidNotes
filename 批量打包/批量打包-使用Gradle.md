# 使用gradle批量打包

## 1.signingConfigs(签名配置)

在`build.gradle`文件的`andorid`块中添加`signingConfigs`

例子：

```
signingConfigs {
    debugConfigs{
        storeFile file('/Users/laowang/keystore/debug/debug.keystore')
        keyAlias 'androiddebugkey'
        keyPassword 'android'
        storePassword 'android'

    }
    releaseConfigs {
        keyAlias 'laowang'
        keyPassword 'laowang'
        storeFile file('/Users/laowang/keystore/release/release.keystore')
        storePassword 'laowang'

    }
}
```

* 这一步也可以在图形界面中完成，具体步骤如下：

```
File -> 
Project Structure -> 
选择具体的Moudle -> 
选择signing选项卡 -> 
底部可以选择添加或者删除配置,右侧可以配置具体的配置内容
```

## 2、buildTypes(构建类型配置)
在`build.gradle`文件的`andorid`块中添加`buildTypes`
```
buildTypes {
    release {
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
    qihu360{
        signingConfig signingConfigs.releaseConfigs
        minifyEnabled true
        zipAlignEnabled true
    }
    bd{
        signingConfig signingConfigs.releaseConfigs
        minifyEnabled true
        zipAlignEnabled true
    }
    anzhi{
        signingConfig signingConfigs.releaseConfigs
        minifyEnabled true
        zipAlignEnabled true
    }
    wandoujia{
        signingConfig signingConfigs.releaseConfigs
        minifyEnabled true
        zipAlignEnabled true
    }
}
```
* 这一步也可以在图形界面中完成，具体步骤如下：
```
File -> 
Project Structure -> 
选择具体的Moudle -> 
选择Build Type选项卡 -> 
底部可以选择添加或者删除配置,右侧可以配置具体的配置内容
```

## 3、在项目根文件夹下生成保存要替换文件的目录

在项目根文件夹下生成保存要替换文件的目录，并添加具体的渠道子文件夹，把Manifest.xml文件拷贝到各个目录下，并修改相应的渠道号(注意某些版本需要对meta-data添加tools:replace="android:value")

例子：

```
/
.../app/
....../channels/
........./bd/
........./qihu360/
........./anzhi/
........./wandoujia/
```

Manifest.xml

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.xinye.test">

    <application android:allowBackup="true" android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher" android:theme="@style/AppTheme">
        <!-- 在各个不同的文件夹下，value替换成相应的渠道号 -->
        <meta-data android:name="CHANNEL" android:value="anzhi"
            tools:replace="android:value"/>

        <activity android:name=".TestActivity"
            android:theme="@android:style/Theme.NoTitleBar.Fullscreen"
            android:screenOrientation="portrait">

            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>

        </activity>
    </application>

</manifest>
```

## 4、sourceSets(设置替换文件夹)

在`build.gradle`文件的`andorid`块中添加`sourceSet`


```
sourceSets{
    bd.setRoot('channels/bd')
    qihu360.setRoot('channels/qh360')
    anzhi.setRoot('channels/anzhi')
    wandoujia.setRoot('channels/wandoujia')
}
```

## 5、构建

点击`grale -> 具体的moudle -> Task -> build -> build`进行构建

可以在`/具体的模块/build/outputs/apk`文件夹下看到生成的apk文件


## 6、附加内容

读取Manifest文件中的渠道号的代码

```
public static String getApplicationMetadata(Context context,String metaDataKey) {
    ApplicationInfo info = null;
    try {
        PackageManager pm = context.getPackageManager();

        info = pm.getApplicationInfo(context.getPackageName(),
            PackageManager.GET_META_DATA);

        return String.valueOf(info.metaData.get(metaDataKey));
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```