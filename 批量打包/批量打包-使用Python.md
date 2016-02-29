
# 批量打包-使用Python
## 1. 使用方式

### 1.1 按照正常流程打包APK

### 1.2 修改渠道文件channels.txt
> 文件内的每一行一个渠道号

例子：

```
360
xiaomi
baidu
91
guanwang
offline
tencent
wandoujia
```

### 1.3 批量打包Python脚本

> 机器上一定要安装python,建议安装2.7.*版本

### 1.4 执行python脚本

```
python package.py APK文件名 输出文件夹名
```

例子：

```
python package.py test-2015-05-19.apk out
```

## 2 原理

* 如果能直接修改apk的渠道号，而不需要再重新签名能节省不少打包的时间。幸运的是我们找到了这种方法。直接解压apk，解压后的根目录会有一个META-INF目录

* 如果在META-INF目录内添加空文件，可以不用重新签名应用。因此，通过为不同渠道的应用添加不同的空文件，可以唯一标识一个渠道。

* 下面的python代码用来给apk添加空的渠道文件，渠道名的前缀为laowang_：


```

import zipfile
import shutil
import sys
import os

apk_path = sys.argv[1]
out_path = sys.argv[2]

if not os.path.exists(out_path):
    os.makedirs(out_path)

name = os.path.basename(apk_path)

channels_file = open('channels.txt')

origin_apk_name = os.path.splitext(name)[0]

for channel in channels_file:
    channel_apk_name = "{}_{}.apk".format(origin_apk_name, channel.strip())
    channel_apk_path = os.path.join(out_path, channel_apk_name)
    shutil.copy2(apk_path, channel_apk_path)
    zipped = zipfile.ZipFile(channel_apk_path, 'a', zipfile.ZIP_DEFLATED)
    empty_channel_file = "META-INF/laowang_{}".format(channel.strip())
    zipped.writestr(empty_channel_file, '')
    zipped.close()
```

* 执行Python命令,将会输出所有指定渠道号的APK文件

> python package.py test-2015-05-19-2.apk out

* 在Android中得到渠道号

```
public static String getMetaInfChannel(Context context) {
        ApplicationInfo appinfo = context.getApplicationInfo();
        String sourceDir = appinfo.sourceDir;
        String ret = "";
        ZipFile zipfile = null;
        try {
            zipfile = new ZipFile(sourceDir);
            Enumeration<?> entries = zipfile.entries();
            while (entries.hasMoreElements()) {
                ZipEntry entry = ((ZipEntry) entries.nextElement());
                String entryName = entry.getName();
                //如果想修改此标示，直接编辑pack.py即可
                if (entryName.startsWith("META-INF/laowang")) {
                    ret = entryName;
                    break;
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (zipfile != null) {
                try {
                    zipfile.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        String[] split = ret.split("_");
        if (split != null && split.length >= 2) {
            return ret.substring(split[0].length() + 1);
        } else {
            return "";
        }
    }
```