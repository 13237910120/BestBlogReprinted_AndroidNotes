# Android 通过adb shell命令查看内存，CPU，启动时间，电量等信息

来源:[http://blog.sina.com.cn/s/blog_13cc013b50102vfwu.html](http://blog.sina.com.cn/s/blog_13cc013b50102vfwu.html)

## 1、  查看内存信息
### 1）查看所有内存信息

命令：

> dumpsys meminfo

或者

> adb shell dumpsys meminfo

 
例：

```
C:\Users\laiyu>adb shell
shell@android:/ $ dumpsys meminfo
dumpsys meminfo
Applications Memory Usage (kB):
Uptime: 80066272 Realtime: 226459939
 
Total PSS by process:
    90058 kB: com.tencent.mobileqq (pid 16731)
    57416 kB: system (pid 651)
    52052 kB: com.miui.home (pid 1121)
    …………(篇幅问题，略)
 
Total PSS by OOM adjustment:
   223177 kB: Persistent
               57416 kB: system (pid 651)
               50036 kB: com.android.deskclock (pid 1096)
…………
   252678 kB: Foreground
               90058 kB: com.tencent.mobileqq (pid 16731)
…………
    50944 kB: Visible
               20318 kB: com.miui.miwallpaper (pid 974)
…………
    90855 kB: Perceptible
               36448 kB: com.google.android.inputmethod.pinyin (pid 987)
…………
    39654 kB: A Services
               23320 kB: com.tencent.android.qqdownloader (pid 14080)
…………
 
    49659 kB: B Services
               20085 kB: com.tencent.mobileqq:qzone (pid 19646)
…………
   148413 kB: Background
               21457 kB: com.miui.weather2 (pid 14296)
…………
                3453 kB: com.miui.providers.datahub (pid 14651)
 
Total PSS by category:
   454627 kB: Dalvik
   137206 kB: Unknown
   100835 kB: .so mmap
    62670 kB: .dex mmap
    54208 kB: Other dev
    30258 kB: Other mmap
     8527 kB: .apk mmap
     4752 kB: .ttf mmap
     2216 kB: Ashmem
       60 kB: Cursor
       21 kB: .jar mmap
        0 kB: Native
 
Total PSS: 855380 kB
      KSM: 0 kB saved from shared 0 kB
           0 kB unshared; 0 kB volatile
```

### 2）查看某个包的内存信息

命令：

> dumpsys meminfo pkg_name

或者

> adb shell dumpsys meminfo pkg_name

 
例：

```
wangheng:adb wangheng$ adb shell dumpsys meminfo com.reshow.rebo
Applications Memory Usage (kB):
Uptime: 1114931193 Realtime: 1493301728

** MEMINFO in pid 3473 [com.reshow.rebo] **
                   Pss  Private  Private  Swapped     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap     8417     1572        0        0    29655    29655    22568
  Dalvik Heap    19668      992        0        0    63981    48568    15413
 Dalvik Other      702      200        0        0                           
        Stack      280       84        0        0                           
    Other dev    18250     1848     3236        0                           
     .so mmap     5478       28     3064        0                           
    .apk mmap      394        0      220        0                           
    .ttf mmap       29        0        0        0                           
    .dex mmap     9000        0     8996        0                           
    code mmap     2683        0      892        0                           
   image mmap     2246       72       64        0                           
   Other mmap      450        4      160        0                           
      Unknown     1812      368        0        0                           
        TOTAL    69409     5168    16632        0    93636    78223    37981
 
 Objects
               Views:      265         ViewRootImpl:        1
         AppContexts:        7           Activities:        1
              Assets:        2        AssetManagers:        2
       Local Binders:       23        Proxy Binders:       27
    Death Recipients:        1
     OpenSSL Sockets:        0
 
 SQL
         MEMORY_USED:      435
  PAGECACHE_OVERFLOW:      128          MALLOC_SIZE:       62
 
 DATABASES
      pgsz     dbsz   Lookaside(b)          cache  Dbname
         4       20             25         2/20/3  /storage/emulated/0/emlibs/libs/monitor.db
         4       20             37        76/19/5  /data/data/com.reshow.rebo/databases/sharesdk.db
         4       16             36        76/17/5  /storage/emulated/0/Mob/comm/dbs/.dh

```

具体输出项含义请搜索网络
 
## 2、  查看CPU信息

### 方法1：linux系统的top命令
 
> top -d 1 | busybox grep "pkg_name"

或者

> adb shell top -d 1 | busybox grep "pkg_name"

或者

> top -d 1 | grep "pkg_name"

或者

> adb shell top -d 1 | grep "pkg_name"

注：直接使用grep可能报错，提示找不到命令，这时如果busybox中有grep命令，可以如上，busybox grep

例子:

```
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  1   1% S    84 1667756K 222156K  fg u0_a362  com.reshow.rebo
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  1   9% S    94 1680536K 226336K  fg u0_a362  com.reshow.rebo
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  0  15% S   102 1738492K 232592K  fg u0_a362  com.reshow.rebo
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  1  48% S   104 1770592K 251836K  fg u0_a362  com.reshow.rebo
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  0  73% S   104 1773752K 269244K  fg u0_a362  com.reshow.rebo
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  2  59% S   104 1774844K 271324K  fg u0_a362  com.reshow.rebo
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  2  62% S   105 1775908K 272168K  fg u0_a362  com.reshow.rebo
 4425  0   0% S     1 1542328K  82204K  fg u0_a362  com.reshow.rebo
 3473  2  65% S   105 1775908K 274144K  fg u0_a362  com.reshow.rebo

```

第三列的百分数就是CPU利用率

### 方法2：通过dummpsys cpuinfo命令

> adb shell dumpsys cpuinfo

或者分成两部走(参考 查看电量信息)

先`adb shell`，然后`dumpsys cpuinfo`
 
例：

```
wangheng:adb wangheng$ adb shell dumpsys cpuinfo
Load: 13.72 / 9.22 / 6.14
CPU usage from 7969ms to 2155ms ago:
  214% 3473/com.reshow.rebo: 194% user + 20% kernel / faults: 34850 minor
  91% 10545/mediaserver: 81% user + 10% kernel / faults: 1292 minor
  23% 4249/com.qihoo.gameassist: 17% user + 5.8% kernel / faults: 1498 minor
  21% 29278/com.tencent.mm: 19% user + 1.5% kernel / faults: 854 minor
  10% 10413/surfaceflinger: 4.9% user + 5.3% kernel
  7.1% 237/logd: 6.3% user + 0.8% kernel
  2.9% 10689/system_server: 2% user + 0.8% kernel / faults: 22 minor
  2.5% 494/adbd: 0.1% user + 2.3% kernel / faults: 228 minor
  1.8% 491/kdfrgx: 0% user + 1.8% kernel
  1.8% 4054/kworker/u8:0: 0% user + 1.8% kernel
  1.5% 4453/kworker/u8:1: 0% user + 1.3% kernel + 0.1% iowait
  1.5% 4459/kworker/u8:3: 0% user + 1.5% kernel
  1.3% 232/irq/104-isp_irq: 0% user + 1.3% kernel
  1.3% 294/dhd_dpc: 0% user + 1.3% kernel
  1.1% 4560/kworker/u8:5: 0% user + 1.1% kernel
  1% 4184/com.qihoo.daemon: 0.5% user + 0.5% kernel / faults: 11 minor
  1% 11325/logcat: 0.3% user + 0.6% kernel
  0.5% 128/mmcqd/0: 0% user + 0.5% kernel
  0.5% 29472/com.tencent.mm:push: 0% user + 0.5% kernel / faults: 1069 minor
  0.3% 153/irq/24-intel_ss: 0% user + 0.3% kernel
  0.3% 5048/kworker/u8:6: 0% user + 0.3% kernel
  0.3% 32435/kworker/2:2: 0% user + 0.3% kernel
  0% 3/ksoftirqd/0: 0% user + 0% kernel
  0.1% 8/rcu_preempt: 0% user + 0.1% kernel
  0.1% 37/irq/47-intel_ps: 0% user + 0.1% kernel
  0.1% 293/dhd_watchdog_th: 0% user + 0.1% kernel
  0.1% 308/sensorhubd: 0% user + 0.1% kernel
  0.1% 17315/com.eg.android.AlipayGphone:push: 0.1% user + 0% kernel / faults: 66 minor
  0.1% 19699/wpa_supplicant: 0% user + 0.1% kernel
99% TOTAL: 81% user + 15% kernel + 1.8% irq + 0.5% softirq

```


## 3、查看应用启动时间

### 方法1
命令：

> adb logcat -c && adb logcat -f /mnt/sdcard/up.txt -s tag
 
选项说明

```
-c   清屏
-f   指定运行结果输出文件，默认输出到标准设备（一般是显示器
-s   设置默认的过滤级别为Silent
tag  仅显示priority/tag
```

更多信息烦请参考 `adb logcat -help`
 
例：

先启动app，然后执行如下命令

> 例：
先启动app，然后执行如下命令
C:\Users\laiyu>adb logcat -c && adb logcat -f /mnt/sdcard/up.txt -s ActivityMana
ger

备注：I/ActivityManager： I 代表优先级，ActivityManager代表tag


### 方法2

> adb shell am start -W package_name/package_name.activivty_name

例子：

> Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.reshow.rebo/.entry.splash.SplashActivity }<br/>
> Status: ok<br/>
> Activity: com.reshow.rebo/.entry.splash.SplashActivity<br/>
> ThisTime: 192<br/>
> TotalTime: 216<br/>
> WaitTime: 229<br/>
> Complete<br/>


## 4、  查看电量信息
命令：

> dumpsys battery
 
例：

```
shell@android:/ $ dumpsys battery
dumpsys battery
Current Battery Service state:
  AC powered: false
  USB powered: true
  status: 5
  health: 2
  present: true
  level: 100
  scale: 100
  voltage:4211
  temperature: 297
  technology: Li-poly
shell@android:/ $
```

## 5、查看当前显示的Activity

linux:

> adb shell dumpsys activity | grep "mFocusedActivity"

windows:

> adb shell dumpsys activity | findstr "mFocusedActivity"