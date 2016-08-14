# Android DownloadManager 的使用

来源:[http://blog.csdn.net/carrey1989/article/details/8060155#](http://blog.csdn.net/carrey1989/article/details/8060155#)

从Android 2.3（API level 9）开始Android用系统服务(Service)的方式提供了Download Manager来优化处理长时间的下载操作。Download Manager处理HTTP

连接并监控连接中的状态变化以及系统重启来确保每一个下载任务顺利完成。

在大多数涉及到下载的情况中使用Download Manager都是不错的选择，特别是当用户切换不同的应用以后下载需要在后台继续进行，以及当下载任务顺利完

成非常重要的情况（DownloadManager对于断点续传功能支持很好）。

要想使用Download Manager，使用`getSystemService`方法请求系统的`DOWNLOAD_SERVICE`服务，代码片段如下：

```
String serviceString = Context.DOWNLOAD_SERVICE; 
DownloadManager downloadManager; 
downloadManager = (DownloadManager) getSystemService(serviceString); 
```

## 下载文件

要请求一个下载操作，需要创建一个DownloadManager.Request对象，将要请求下载的文件的Uri传递给Download Manager的`enqueue`方法，代码片段如下所示：

```
String serviceString = Context.DOWNLOAD_SERVICE;  
DownloadManager downloadManager;  
downloadManager = (DownloadManager)getSystemService(serviceString);  
  
Uri uri = Uri.parse("http://developer.android.com/shareables/icon_templates-v4.0.zip");  
DownloadManager.Request request = new Request(uri);  
long reference = downloadManager.enqueue(request);  
```

在这里返回的reference变量是系统为当前的下载请求分配的一个唯一的**ID**，我们可以通过这个**ID**重新获得这个下载任务，进行一些自己想要进行的操作或者查询下载的状态以及取消下载等等。

我们可以通过`addRequestHeader`方法为`DownloadManager.Request`对象request添加HTTP头，也可以通过`setMimeType`方法重写从服务器返回的`mime type`。

我们还可以指定在什么连接状态下执行下载操作。`setAllowedNetworkTypes`方法可以用来限定在WiFi还是手机网络下进行下载，`setAllowedOverRoaming`方法可以用来阻止手机在漫游状态下下载。

下面的代码片段用于指定一个较大的文件只能在WiFi下进行下载：

```
request.setAllowedNetworkTypes(Request.NETWORK_WIFI); 
```

Android API level 11 介绍了`getRecommendedMaxBytesOverMobile`类方法（静态方法），返回一个当前手机网络连接下的最大建议字节数，可以来判断下载是否应该限定在WiFi条件下。

调用enqueue方法之后，只要数据连接可用并且Download Manager可用，下载就会开始。

要在下载完成的时候获得一个系统通知（notification）,注册一个广播接受者来接收**ACTION_DOWNLOAD_COMPLETE**广播，这个广播会包含一个**EXTRA_DOWNLOAD_ID**信息在intent中包含了已经完成的这个下载的ID,代码片段如下所示：

```
IntentFilter filter = new IntentFilter(DownloadManager.ACTION_DOWNLOAD_COMPLETE);  
      
BroadcastReceiver receiver = new BroadcastReceiver() {  
  @Override  
  public void onReceive(Context context, Intent intent) {  
    long reference = intent.getLongExtra(DownloadManager.EXTRA_DOWNLOAD_ID, -1);  
    if (myDownloadReference == reference) {  
       
    }  
  }  
};  
registerReceiver(receiver, filter);  
```

使用Download Manager的`openDownloadedFile`方法可以打开一个已经下载完成的文件，返回一个`ParcelFileDescriptor`对象。我们可以通过Download Manager来查询下载文件的保存地址，如果在下载时制定了路径和文件名，我们也可以直接操作文件。

我们可以为**ACTION_NOTIFICATION_CLICKED** action注册一个广播接受者，当用户从通知栏点击了一个下载项目或者从Downloads app点击可一个下载的项目的时候，系统就会发出一个点击下载项的广播。

代码片段如下：

```
IntentFilter filter = new IntentFilter(DownloadManager.ACTION_NOTIFICATION_CLICKED);  
  
BroadcastReceiver receiver = new BroadcastReceiver() {  
  @Override  
  public void onReceive(Context context, Intent intent) {  
    String extraID = DownloadManager.EXTRA_NOTIFICATION_CLICK_DOWNLOAD_IDS;  
    long[] references = intent.getLongArrayExtra(extraID);  
    for (long reference : references)  
      if (reference == myDownloadReference) {  
        // Do something with downloading file.  
      }  
  }  
};  
  
registerReceiver(receiver, filter);  
```

## 定制Download Manager Notifications的样式

默认情况下，通知栏中会显示被Download Manager管理的每一个download每一个Notification会显示当前的下载进度和文件的名字，如下图所示(图没了)：

通过Download Manager可以为每一个download request定制**Notification**的样式，包括完全隐藏Notification。下面的代码片段显示了通过`setTitle`和`setDescription`方法来定制显示在文件下载**Notification**中显示的文字。

```
request.setTitle(“Earthquakes”);  
request.setDescription(“Earthquake XML”);  
```

`setNotificationVisibility`方法可以用来控制什么时候显示Notification，甚至是隐藏该request的Notification。有以下几个参数：

* **Request.VISIBILITY_VISIBLE：**在下载进行的过程中，通知栏中会一直显示该下载的Notification，当下载完成时，该Notification会被移除，这是默认的参数值。
* **Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED：**在下载过程中通知栏会一直显示该下载的Notification，在下载完成后该Notification会继续显示，直到用户点击该
Notification或者消除该Notification。
* **Request.VISIBILITY_VISIBLE_NOTIFY_ONLY_COMPLETION：**只有在下载完成后该Notification才会被显示。
* **Request.VISIBILITY_HIDDEN：**不显示该下载请求的Notification。如果要使用这个参数，需要在应用的清单文件中加上*DOWNLOAD_WITHOUT_NOTIFICATION*权限。

## 指定下载保存地址

默认情况下，所有通过Download Manager下载的文件都保存在一个共享下载缓存中，使用系统生成的文件名每一个Request对象都可以制定一个下载保存的地址，通常情况下，所有的下载文件都应该保存在外部存储中，所以我们需要在应用清单文件中加上**WRITE_EXTERNAL_STORAGE**权限：

```
<uses-permission android:name=”android.permission.WRITE_EXTERNAL_STORAGE”/>
```

下面的代码片段是在外部存储中指定一个任意的保存位置的方法：

```
request.setDestinationUri(Uri.fromFile(f));  
```

f是一个File对象。

如果下载的这个文件是你的应用所专用的，你可能会希望把这个文件放在你的应用在外部存储中的一个专有文件夹中。注意这个文件夹不提供访问控制，所以其他的应用也可以访问这个文件夹。在这种情况下，如果你的应用卸载了，那么在这个文件夹也会被删除。

下面的代码片段是指定存储文件的路径是应用在外部存储中的专用文件夹的方法：

```
request.setDestinationInExternalFilesDir(this,  
  Environment.DIRECTORY_DOWNLOADS, “Bugdroid.png”);  
```

如果下载的文件希望被其他的应用共享，特别是那些你下载下来希望被Media Scanner扫描到的文件（比如音乐文件），那么你可以指定你的下载路径在外部存储的公共文件夹之下，下面的代码片段是将文件存放到外部存储中的公共音乐文件夹的方法：

```
request.setDestinationInExternalPublicDir(Environment.DIRECTORY_MUSIC,  
  "Android_Rock.mp3");  
```

在默认的情况下，通过Download Manager下载的文件是不能被Media Scanner扫描到的，进而这些下载的文件（音乐、视频等）就不会在Gallery和Music Player这样的应用中看到。

为了让下载的音乐文件可以被其他应用扫描到，我们需要调用Request对象的allowScaningByMediaScanner方法。

如果我们希望下载的文件可以被系统的Downloads应用扫描到并管理，我们需要调用Request对象的`setVisibleInDownloadsUi`方法，传递参数true。

## 取消和删除下载
Download Manager的remove方法可以用来取消一个准备进行的下载，中止一个正在进行的下载，或者删除一个已经完成的下载。

remove方法接受若干个download 的ID作为参数，你可以设置一个或者几个你想要取消的下载的ID，如下代码段所示：

```
downloadManager.remove(REFERENCE_1, REFERENCE_2, REFERENCE_3);  
```

该方法返回成功取消的下载的个数，如果一个下载被取消了，所有相关联的文件，部分下载的文件和完全下载的文件都会被删除。

## 查询Download Manager
你可以通过查询Download Manager来获得下载任务的状态，进度，以及各种细节，通过query方法返回一个包含了下载任务细节的Cursor。

query方法传递一个DownloadManager.Query对象作为参数，通过DownloadManager.Query对象的setFilterById方法可以筛选我们希望查询的下载任务的ID。也可以使用setFilterByStatus方法筛选我们希望查询的某一种状态的下载任务，传递的参数是**DownloadManager.STATUS_\***常量，可以指定正在进行、暂停、失败、完成四种状态。

Download Manager包含了一系列COLUMN_*静态String常量，可以用来查询Cursor中的结果列索引。我们可以查询到下载任务的各种细节，包括状态，文件大小，已经下载的字节数，标题，描述，URI，本地文件名和URI，媒体类型以及Media Provider download URI。

下面的代码段是通过注册监听下载完成事件的广播接受者来查询下载完成文件的本地文件名和URI的实现方法：

```
@Override  
public void onReceive(Context context, Intent intent) {  
  long reference = intent.getLongExtra(DownloadManager.EXTRA_DOWNLOAD_ID, -1);  
  if (myDownloadReference == reference) {  
       
    Query myDownloadQuery = new Query();  
    myDownloadQuery.setFilterById(reference);  
        
    Cursor myDownload = downloadManager.query(myDownloadQuery);  
    if (myDownload.moveToFirst()) {  
      int fileNameIdx =   
        myDownload.getColumnIndex(DownloadManager.COLUMN_LOCAL_FILENAME);  
      int fileUriIdx =   
        myDownload.getColumnIndex(DownloadManager.COLUMN_LOCAL_URI);  
  
      String fileName = myDownload.getString(fileNameIdx);  
      String fileUri = myDownload.getString(fileUriIdx);  
      
      // TODO Do something with the file.  
      Log.d(TAG, fileName + " : " + fileUri);  
    }  
    myDownload.close();  
  
  }  
} 
```

对于暂停和失败的下载，我们可以通过查询**COLUMN_REASON**列查询出原因的整数码。

对于**STATUS_PAUSED**状态的下载，可以通过**DownloadManager.PAUSED_\***静态常量来翻译出原因的整数码，进而判断出下载是由于等待网络连接还是等待WiFi连接还是准备重新下载三种原因而暂停。

对于**STATUS_FAILED**状态的下载，我们可以通过**DownloadManager.ERROR_\***来判断失败的原因，可能是错误码（失败原因）包括没有存储设备，存储空间不足，重复的文件名，或者HTTP errors。

下面的代码是如何查询出当前所有的暂停的下载任务，提取出暂停的原因以及文件名称，下载标题以及当前进度的实现方法：

```
// Obtain the Download Manager Service.  
String serviceString = Context.DOWNLOAD_SERVICE;  
DownloadManager downloadManager;  
downloadManager = (DownloadManager)getSystemService(serviceString);  
  
// Create a query for paused downloads.  
Query pausedDownloadQuery = new Query();  
pausedDownloadQuery.setFilterByStatus(DownloadManager.STATUS_PAUSED);  
  
// Query the Download Manager for paused downloads.  
Cursor pausedDownloads = downloadManager.query(pausedDownloadQuery);  
  
// Find the column indexes for the data we require.  
int reasonIdx = pausedDownloads.getColumnIndex(DownloadManager.COLUMN_REASON);  
int titleIdx = pausedDownloads.getColumnIndex(DownloadManager.COLUMN_TITLE);  
int fileSizeIdx =   
  pausedDownloads.getColumnIndex(DownloadManager.COLUMN_TOTAL_SIZE_BYTES);      
int bytesDLIdx =   
  pausedDownloads.getColumnIndex(DownloadManager.COLUMN_BYTES_DOWNLOADED_SO_FAR);  
  
// Iterate over the result Cursor.  
while (pausedDownloads.moveToNext()) {  
  // Extract the data we require from the Cursor.  
  String title = pausedDownloads.getString(titleIdx);  
  int fileSize = pausedDownloads.getInt(fileSizeIdx);  
  int bytesDL = pausedDownloads.getInt(bytesDLIdx);  
  
  // Translate the pause reason to friendly text.  
  int reason = pausedDownloads.getInt(reasonIdx);  
  String reasonString = "Unknown";  
  switch (reason) {  
    case DownloadManager.PAUSED_QUEUED_FOR_WIFI :   
      reasonString = "Waiting for WiFi"; break;  
    case DownloadManager.PAUSED_WAITING_FOR_NETWORK :   
      reasonString = "Waiting for connectivity"; break;  
    case DownloadManager.PAUSED_WAITING_TO_RETRY :  
      reasonString = "Waiting to retry"; break;  
    default : break;  
  }  
  
  // Construct a status summary  
  StringBuilder sb = new StringBuilder();  
  sb.append(title).append("\n");  
  sb.append(reasonString).append("\n");  
  sb.append("Downloaded ").append(bytesDL).append(" / " ).append(fileSize);  
  
  // Display the status   
  Log.d("DOWNLOAD", sb.toString());  
}  
  
// Close the result Cursor.  
pausedDownloads.close();  
```

----------------

## 更新的例子

```
package com.test.update;

import android.annotation.SuppressLint;
import android.app.Activity;
import android.app.DownloadManager;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.net.Uri;
import android.os.Environment;
import android.text.Html;
import android.view.Gravity;
import android.view.View;
import android.widget.TextView;
import android.widget.Toast;

import org.json.JSONObject;

import java.io.File;

import okhttp3.Call;

/**
 * 升级管理器
 *
 * @author wangheng
 */
public class UpdateManager {


    public static final String KEY_UPDATE_DOWNLOAD_ID = "keyUpdateDownloadId";
    public static final int INVALID_UPDATE_DOWNLOAD_ID = -1;

    private static final String KEY_HAS_NEW_VERSION = "keyHasNewVersion";

    private UpdateManager(){

    }

    private static class Generator{
        private static final UpdateManager INSTANCE = new UpdateManager();
    }

    public static UpdateManager getInstance(){
        return Generator.INSTANCE;
    }

    /**
     * 下载
     * @param info 更新信息
     */
    public void download(UpdateInfo info){

        DownloadManager dm = (DownloadManager) App.getInstance().getContext()
                .getSystemService(Context.DOWNLOAD_SERVICE);

        DownloadManager.Request request = new DownloadManager.Request(Uri.parse(info.getDownloadUrl()));
        request.setAllowedNetworkTypes(DownloadManager.Request.NETWORK_MOBILE
                | DownloadManager.Request.NETWORK_WIFI);

        File dir = null;
        if(Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())){
//            dir = App.getInstance().getContext().getExternalCacheDir();
            String path = FileManager.getInstance().getApkSaveDir();
            if(path == null){

                Toast.makeText(App.getInstance().getContext(),
                        R.string.no_sdcard,Toast.LENGTH_LONG).show();
                return;
            }
            dir = new File(path);
            if(!dir.exists()){
                Toast.makeText(App.getInstance().getContext(),
                        R.string.no_sdcard,Toast.LENGTH_LONG).show();
                return;

            }
            Logger.i("wangheng","###########" + path);
        }else{
            Toast.makeText(App.getInstance().getContext(),
                    R.string.no_sdcard,Toast.LENGTH_LONG).show();
            return;
        }

        File file = new File(dir,info.getVersionCode() + ".apk");
//        if(file.exists()){
//            file.delete();
//        }

//        Logger.i("wangheng",file.getAbsolutePath() + "########file exists :" + file.exists() + "\n" + getMD5(file));
        if(file.exists()){
            String md5 = MD5.getMD5(file);
            if(md5 != null && md5.equalsIgnoreCase(info.getCheckCode())) {
                Intent installIntent = new Intent(Intent.ACTION_VIEW);
                installIntent.setDataAndType(Uri.fromFile(file),
                        "application/vnd.android.package-archive");
                installIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                App.getInstance().getContext().startActivity(installIntent);
                return;
            }
        }

        request.setDestinationUri(Uri.fromFile(file));

        request.setMimeType("application/vnd.android.package-archive");
        request.allowScanningByMediaScanner();
        // 设置为可见和可管理
        request.setVisibleInDownloadsUi(true);
        // 自定义标题和描述
        request.setTitle(App.getInstance().getString(R.string.update_statusbar_title));

        String appName = App.getInstance().getString(R.string.app_name);
        String downloading = App.getInstance().getString(R.string.update_statusbar_description);
        request.setDescription(appName + "V" + info.getVersionName() + downloading);

        // 在下载过程中通知栏会一直显示该下载的Notification，在下载完成后该Notification会继续显示，
        // 直到用户点击该Notification或者消除该Notification
        request.setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED);

        // 返回值是系统为当前的下载请求分配的一个唯一的ID，
        // 我们可以通过这个ID重新获得这个下载任务，进行一些自己想要进行的操作或者查询
        long downloadId = dm.enqueue(request);
        SPManager.getInstance().putLong(KEY_UPDATE_DOWNLOAD_ID,downloadId);

        IntentFilter filter = new IntentFilter(DownloadManager.ACTION_DOWNLOAD_COMPLETE);
        App.getInstance().getContext().registerReceiver(new UpdateReceiver(),filter);
    }

    /**
     * 检查新版本
     * @param callback callback
     */
    public void checkUpdate(final ICheckUpdateCallback callback){

        if(null == callback){
            return;
        }

        String channel = App.getInstance().getChannel();
        int versionCode = PackageUtils.getVersionCode(App.getInstance().getContext());
        String versionName = PackageUtils.getVersionName(App.getInstance().getContext());

        PhoneLiveApi.checkUpdate(versionCode, versionName, channel, new StringCallback() {
            @Override
            public void onError(Call call, Exception e) {

                SPManager.getInstance().putBoolean(KEY_HAS_NEW_VERSION,false);

                callback.onFailure(e);
            }

            @Override
            public void onResponse(String response) {

                UpdateInfo info = new UpdateInfo();
                JSONObject jsonObject = null;
                try {
                    jsonObject = new JSONObject(response);
                    JSONObject dataObject = new JSONObject(jsonObject.getString("data"));
                    JSONObject infoObject = new JSONObject(dataObject.getString("info"));

                    info.setNeedUpdate(infoObject.getInt("needUpdate"));
                    info.setVersionName(infoObject.getString("versionName"));
                    info.setContent(infoObject.getString("content"));
                    info.setVersionCode(infoObject.getInt("versionCode"));
                    info.setSize(infoObject.getString("size"));
                    info.setDownloadUrl(infoObject.getString("downloadUrl"));
                    info.setCheckCode(infoObject.getString("checkCode"));

                    SPManager.getInstance().putBoolean(KEY_HAS_NEW_VERSION,info.hasNewVersion());

                    callback.onSuccess(info);
                } catch (Exception e) {

                    e.printStackTrace();
                    info.setNeedUpdate(UpdateInfo.NOT_HAS_NEW_VERSION);
                    SPManager.getInstance().putBoolean(KEY_HAS_NEW_VERSION,false);

                    callback.onSuccess(info);
                }
            }
        });
    }

    public DialogPlus showUpdateDialog(final Activity activity, final UpdateInfo info){
        if(null == info){
            return null;
        }

        @SuppressLint("InflateParams")
        View contentView = App.getInstance().getLayoutInflater()
                .inflate(R.layout.dialog_update,null);


        // 更新版本
        TextView versionNameTextView = (TextView) contentView.findViewById(R.id.tvUpdateVersionName);
        String versionNameMode = App.getInstance().getString(R.string.update_version_name_mode);
        versionNameTextView.setText(String.format(versionNameMode,info.getVersionName()));

        // 更新大小
        TextView sizeTextView = (TextView) contentView.findViewById(R.id.tvUpdateSize);
        String sizeMode = App.getInstance().getString(R.string.update_size_mode);
        sizeTextView.setText(String.format(sizeMode,info.getSize()));

        // 更新内容
        TextView contentTextView = (TextView) contentView.findViewById(R.id.tvUpdateContent);
        contentTextView.setText(Html.fromHtml(String.valueOf(info.getContent())));

        return DialogHelp.showNormalDialog(activity, contentView, Gravity.CENTER,
                new OnClickListener() {
                    @Override
                    public void onClick(DialogPlus dialog, View view) {
                        switch (view.getId()) {
                            case R.id.tvUpdateCancel:
                                dialog.dismiss();
                                break;
                            case R.id.tvUpdateCommit:
                                dialog.dismiss();
                                download(info);
                                break;
                            default:
                                break;

                        }
                    }
                }, true, R.drawable.dialog_phone_background);
    }

    public interface ICheckUpdateCallback{
        void onSuccess(UpdateInfo info);
        void onFailure(Exception e);
    }

    /**
     * 是否有新版本
     * @return
     */
    public boolean hasNewVersion(){
        return SPManager.getInstance().getBoolean(KEY_HAS_NEW_VERSION,false);
    }
}
```

Receiver

```
package com.test.update;

import android.app.DownloadManager;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;

import java.io.File;

/**
 * 升级Receiver
 *
 * @author wangheng
 */
public class UpdateReceiver extends BroadcastReceiver {

    private static final long INVALID_DOWNLOAD_ID = -1;

    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        // 下载完成广播
        if(DownloadManager.ACTION_DOWNLOAD_COMPLETE.equals(action)){

            // 得到DownloadId
            long downloadId = intent.getLongExtra(DownloadManager.EXTRA_DOWNLOAD_ID,INVALID_DOWNLOAD_ID);
            long currentId = SPManager.getInstance().getLong(UpdateManager.KEY_UPDATE_DOWNLOAD_ID
                    ,UpdateManager.INVALID_UPDATE_DOWNLOAD_ID);

            if(downloadId == currentId){
                Logger.i("wangheng","auto update when download completed");
                installApk(context, downloadId);
            }
        }else if(DownloadManager.ACTION_NOTIFICATION_CLICKED.equals(action)){
            long ids[] = intent.getLongArrayExtra(DownloadManager.EXTRA_NOTIFICATION_CLICK_DOWNLOAD_IDS);
            if(null == ids){
                return;
            }
            long currentId = SPManager.getInstance().getLong(UpdateManager.KEY_UPDATE_DOWNLOAD_ID
                    ,UpdateManager.INVALID_UPDATE_DOWNLOAD_ID);
            for(long id : ids){
                if(id == currentId){
                    Logger.i("wangheng","click status bar notification");
                    installApk(context,currentId);
                }
            }
        }
    }

    private void installApk(Context context, long downloadId) {
        DownloadManager dm = (DownloadManager) App.getInstance().getContext()
                .getSystemService(Context.DOWNLOAD_SERVICE);
        Uri uri = dm.getUriForDownloadedFile(downloadId);
        String path = uri.getPath();


        Intent installIntent = new Intent(Intent.ACTION_VIEW);
        installIntent.setDataAndType(Uri.fromFile(new File(path)), "application/vnd.android.package-archive");
        installIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(installIntent);
    }
}
```

```
package com.test.update;

import java.io.Serializable;

/**
 *
 * 更新信息
 *
 * <pre>
 response:

 needUpdate:int 1 (1:有新版本，其他值没有新版本)
 versionCode:int 版本号
 versionName:String 版本名(不带V)
 size:String 大小(如15.4M)
 downloadUrl:String 下载地址
 content:String 更新内容
 // forceUpdate:int 1(1表示必须更新，其他值表示非必须更新)


 request:

 versionCode:int 版本号
 versionName:String 版本名(不带V）
 channel:String 渠道号
 osType:int 1(1表示andrsoid,2表示ios)
 </pre>
 *
 * @author wangheng
 */
public class UpdateInfo implements Serializable{

    private static final long serialVersionUID = -3613164738336297527L;

    // 操作系统类型
    public static final String UPDATE_OS_TYPE = "1";

    // 有新版本
    private static final int HAS_NEW_VERSION = 1;
    // 没有新版本
    public static final int NOT_HAS_NEW_VERSION = 0;

    // 1 代表有新版本 ；其他值代表没有新版本
    private int needUpdate;
    // 版本号 int 类型
    private int versionCode;
    // 版本名(不带V)
    private String versionName;
    // 更新内容
    private String content;

    // 下载地址
    private String downloadUrl;

    // APK大小，比如1.5M
    private String size;

    // md5值
    private String checkCode;

    public String getCheckCode() {
        return checkCode;
    }

    public void setCheckCode(String checkCode) {
        this.checkCode = checkCode;
    }

    public boolean hasNewVersion(){
        return needUpdate == HAS_NEW_VERSION;
    }

    public int getNeedUpdate() {
        return needUpdate;
    }

    public void setNeedUpdate(int needUpdate) {
        this.needUpdate = needUpdate;
    }

    public int getVersionCode() {
        return versionCode;
    }

    public void setVersionCode(int versionCode) {
        this.versionCode = versionCode;
    }

    public String getVersionName() {
        return versionName;
    }

    public void setVersionName(String versionName) {
        this.versionName = versionName;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public String getDownloadUrl() {
        return downloadUrl;
    }

    public void setDownloadUrl(String downloadUrl) {
        this.downloadUrl = downloadUrl;
    }

    public String getSize() {
        return size;
    }

    public void setSize(String size) {
        this.size = size;
    }

    @Override
    public String toString() {
        return "UpdateInfo{" +
                "needUpdate=" + needUpdate +
                ", versionCode=" + versionCode +
                ", versionName='" + versionName + '\'' +
                ", content='" + content + '\'' +
                ", downloadUrl='" + downloadUrl + '\'' +
                ", size='" + size + '\'' +
                ", checkCode='" + checkCode + '\'' +
                '}';
    }
}
```
