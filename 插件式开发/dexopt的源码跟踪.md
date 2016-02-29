# dexopt的源码跟踪

来源:[zongwu233.github.io](http://zongwu233.github.io/dexopt-process/)

关于dexopt的预备知识在[这里](http://www.netmite.com/android/mydroid/dalvik/docs/dexopt.html)。

一般都是从`AndroidStudio run`出来的apk安装到手机，这里从`adb shell pm install -r xxx.apk`开始看起：
* 找到`android/frameworks/base/cmds/pm/src/com/android/commands/pm/pm.java` `pm`命令的源码
* `main`函数调用了`new Pm().run(args);`
* 找到 `runInstall（）`方法
> 哦刚才的常用参数 `-r` 表示覆盖安装模式`INSTALL_REPLACE_EXISTING`
* `runInstall（）`方法最终会调用

```
mPm.installPackageWithVerification(apkURI, obs, installFlags, 
    installerPackageName,verificationURI, null, encryptionParams);

```

而`mPm = IPackageManager.Stub.asInterface(ServiceManager.getService("package"));`就是`PackageMangerService`

`PackageMangerService`的`main`方法：

```
public static final IPackageManager main(Context context, Installer installer,
 boolean factoryTest, boolean onlyCore) {
	PackageManagerService m = new PackageManagerService(context, installer,
 			factoryTest, onlyCore);
 	ServiceManager.addService("package", m);
 	return m;
}
```

去看`installPackageWithVerification()`

```
@Override
 public void installPackageWithVerification(Uri packageURI, 
 								IPackageInstallObserver observer,
 								int flags, 
 								String installerPackageName, 
 								Uri verificationURI,
 								ManifestDigest manifestDigest, 
 								ContainerEncryptionParams encryptionParams) {
 								
 	VerificationParams verificationParams = new VerificationParams(
 									verificationURI, 
 									null, 
 									null,
 									VerificationParams.NO_UID, 
 									manifestDigest);
 									
 	installPackageWithVerificationAndEncryption(
 									packageURI, 
 									observer, 
 									flags,
 									installerPackageName, 
 									verificationParams, 
 									encryptionParams);
 }

```

接着看`installPackageWithVerificationAndEncryption()`方法

```
public void installPackageWithVerificationAndEncryption(
							Uri packageURI,
 							IPackageInstallObserver observer, 
 							int flags, 
 							String installerPackageName,
 							VerificationParams verificationParams, 
 							ContainerEncryptionParams encryptionParams)

```

在最后的时候：

```
final Message msg = mHandler.obtainMessage(INIT_COPY);
msg.obj = new InstallParams(packageURI, observer, 
			filteredFlags, installerPackageName,
			verificationParams, encryptionParams, user);
mHandler.sendMessage(msg);

```

发messge 给handler，找到这个handler的定义

```
mHandlerThread.start();
mHandler = new PackageHandler(mHandlerThread.getLooper());
final HandlerThread mHandlerThread = new HandlerThread("PackageManager",
		Process.THREAD_PRIORITY_BACKGROUND);
final PackageHandler mHandler;

```

看来是个工作线程。

跟进`handler`的`handlemsg()` 消息类型会从`INIT_COPY`绕成`MCS_BOUND`。总算调用了`params.startCopy()`先看`handleStartCopy()`. `InstallParams`对该方法的实现。 略过各种检查和校验 找到了`ret = args.copyApk(mContainerService, true);`copy apk文件到`/data/app`目录下 而且是以一种临时文件的格式。

```
final boolean startCopy() {
	boolean res;
	try {
		if (DEBUG_INSTALL) Slog.i(TAG, "startCopy");
		if (++mRetries > MAX_RETRIES) {
			Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
			mHandler.sendEmptyMessage(MCS_GIVE_UP);
			handleServiceError();
			return false;
		} else {
			handleStartCopy();
			res = true;
		}
	} catch (RemoteException e) {
		if (DEBUG_INSTALL) Slog.i(TAG, "Posting install MCS_RECONNECT");
		mHandler.sendEmptyMessage(MCS_RECONNECT);
		res = false;
	}
	handleReturnCode();
	return res;
}

```

然后是`handleReturnCode()`主要是:

```
private void processPendingInstall(final InstallArgs args, final int currentStatus) 
```

该方法调用

```
private void installNewPackageLI(PackageParser.Package pkg,
			int parseFlags, int scanMode, UserHandle user,
 			String installerPackageName, PackageInstalledInfo res)
```

终于到了`installnewPackage`了

```
private void installNewPackageLI(PackageParser.Package pkg,
			int parseFlags, int scanMode, UserHandle user,
			String installerPackageName, PackageInstalledInfo res)
			
private PackageParser.Package scanPackageLI(File scanFile,
			int parseFlags, int scanMode, long currentTime, UserHandle user)
``` 

然后调用了

```
// Note that we invoke the following method only if we are about to unpack an application
PackageParser.Package scannedPkg = scanPackageLI(pkg, parseFlags, scanMode
 	| SCAN_UPDATE_SIGNATURE, currentTime, user);

```

其中有

```
//invoke installer to do the actual installation
int ret = createDataDirsLI(pkgName, pkg.applicationInfo.uid);

if((scanMode & SCAN_NO_DEX) == 0) {
    if(performDexOptLI(pkg, forceDex, (scanMode & SCAN_DEFER_DEX) != 0) == DEX_OPT_FAILED) {
        mLastScanError = PackageManager.INSTALL_FAILED_DEXOPT;
        return null;
    }
}
```

先看`scanPackageLI`

```
private int createDataDirsLI(String packageName, int uid) {
    int[] users = sUserManager.getUserIds();
    int res = mInstaller.install(packageName, uid, uid);
    if(res < 0) {
        return res;
    }
    for(int user: users) {
        if(user != 0) {
            res = mInstaller.createUserData(packageName, 
            			UserHandle.getUid(user, uid), user);
            if(res < 0) {
                return res;
            }
        }
    }
    return res;
}
```

`mInstaller.install()`出现了,再看`performDexOptLI()`

```
private int performDexOptLI(PackageParser.Package pkg, boolean forceDex, boolean defer) {
    boolean performed = false;
    if((pkg.applicationInfo.flags & ApplicationInfo.FLAG_HAS_CODE) != 0) {
        String path = pkg.mScanPath;
        int ret = 0;
        try {
            if(forceDex || dalvik.system.DexFile.isDexOptNeeded(path)) {
                if( ! forceDex && defer) {
                    mDeferredDexOpt.add(pkg);
                    return DEX_OPT_DEFERRED;
                } else {
                    Log.i(TAG, "Running dexopt on: " + pkg.applicationInfo.packageName);
                    final int sharedGid = UserHandle.getSharedAppGid(pkg.applicationInfo.uid);
                    ret = mInstaller.dexopt(path, sharedGid, ! isForwardLocked(pkg));
                    pkg.mDidDexOpt = true;
                    performed = true;
                }
            }
        } catch(FileNotFoundException e) {
            Slog.w(TAG, "Apk not found for dexopt: " + path);
            ret = - 1;
        } catch(IOException e) {
            Slog.w(TAG, "IOException reading apk: " + path, e);
            ret = - 1;
        } catch(dalvik.system.StaleDexCacheError e) {
            Slog.w(TAG, "StaleDexCacheError when reading apk: " + path, e);
            ret = - 1;
        } catch(Exception e) {
            Slog.w(TAG, "Exception when doing dexopt : ", e);
            ret = - 1;
        }
        if(ret < 0) {
            // error from installer
            return DEX_OPT_FAILED;
        }
    }
    return performed ? DEX_OPT_PERFORMED : DEX_OPT_SKIPPED;
}
```

`mInstaller.dexopt()`终于也找出来了

`PackageManagerService`通过`socket`访问`installd`的服务进程，而`installd`的服务是在`init`进程里被启动的。继续看`frameworks/base/cmds/installd/installd.c`

```
int main(const int argc, const char *argv[]) {
	char buf[BUFFER_MAX];
	struct sockaddr addr;
	socklen_t alen;
	int lsocket, s, count;
	if (initialize_globals() < 0) {
		ALOGE("Could not initialize globals; exiting.\n");
		exit(1);
	}
	if (initialize_directories() < 0) {
		ALOGE("Could not create directories; exiting.\n");
		exit(1);
	}
	lsocket = android_get_control_socket(SOCKET_PATH);
	if (lsocket < 0) {
		ALOGE("Failed to get socket from environment: %s\n", strerror(errno));
		exit(1);
	}
	if (listen(lsocket, 5)) {
		ALOGE("Listen on socket failed: %s\n", strerror(errno));
		exit(1);
	}
	fcntl(lsocket, F_SETFD, FD_CLOEXEC);
	for (;;) {
		alen = sizeof(addr);
		s = accept(lsocket, &addr, &alen);
		if (s < 0) {
			ALOGE("Accept failed: %s\n", strerror(errno));
			continue;
		}
		fcntl(s, F_SETFD, FD_CLOEXEC);
		ALOGI("new connection\n");
		for (;;) {
			unsigned short count;
			if (readx(s, &count, sizeof(count))) {
				ALOGE("failed to read size\n");
				break;
			}
			if ((count < 1) || (count >= BUFFER_MAX)) {
				ALOGE("invalid size %d\n", count);
				break;
			}
			if (readx(s, buf, count)) {
				ALOGE("failed to read command\n");
				break;
			}
			buf[count] = 0;
			if (execute(s, buf))
				break;
		}
		ALOGI("closing connection\n");
		close(s);
	}
	return 0;
}
```

接收的命令:

```
struct cmdinfo {
	const char *name;
	unsigned numargs;
	int (*func)(char **arg, char reply[REPLY_MAX]);
};
struct cmdinfo cmds[] = { { "ping", 0, do_ping }, 
					{ "install", 3, do_install },
					{ "dexopt", 3, do_dexopt }, 
					{ "movedex", 2, do_move_dex }, 
					{ "rmdex", 1,do_rm_dex }, 
					{ "remove", 2, do_remove }, 
					{ "rename", 2,do_rename }, 
					{ "freecache", 1, do_free_cache }, 
					{ "rmcache", 1,do_rm_cache }, 
					{ "protect", 2, do_protect }, 
					{ "getsize", 4,do_get_size }, 
					{ "rmuserdata", 2, do_rm_user_data }, 
					{"movefiles", 0, do_movefiles }, 
					{ "linklib", 2, do_linklib }, 
					{"unlinklib", 1, do_unlinklib }, 
					{ "mkuserdata", 3,do_mk_user_data }, 
					{ "rmuser", 1, do_rm_user }, 
					{"cloneuserdata", 3, do_clone_user_data }, };
```

比如

```
static int do_install(char **arg, char reply[REPLY_MAX]) {
	/* pkgname, uid, gid */
	return install(arg[0], atoi(arg[1]), atoi(arg[2])); 
}
static int do_dexopt(char **arg, char reply[REPLY_MAX]) {
	/* apk_path, uid, is_public */
	return dexopt(arg[0], atoi(arg[1]), atoi(arg[2]));
}
```

`frameworks/base/cmds/installd/commands.c`

```
int install(const char *pkgname, uid_t uid, gid_t gid) {
	char pkgdir[PKG_PATH_MAX];
	char libdir[PKG_PATH_MAX];
	if ((uid < AID_SYSTEM) || (gid < AID_SYSTEM)) {
		ALOGE("invalid uid/gid: %d %d\n", uid, gid);
		return -1;
	}
	if (create_pkg_path(pkgdir, pkgname, PKG_DIR_POSTFIX, 0)) {
		ALOGE("cannot create package path\n");
		return -1;
	}
	if (create_pkg_path(libdir, pkgname, PKG_LIB_POSTFIX, 0)) {
		ALOGE("cannot create package lib path\n");
		return -1;
	}
	if (mkdir(pkgdir, 0751) < 0) {
		ALOGE("cannot create dir '%s': %s\n", pkgdir, strerror(errno));
		return -errno;
	}
	if (chmod(pkgdir, 0751) < 0) {
		ALOGE("cannot chmod dir '%s': %s\n", pkgdir, strerror(errno));
		unlink(pkgdir);
		return -errno;
	}
	if (chown(pkgdir, uid, gid) < 0) {
		ALOGE("cannot chown dir '%s': %s\n", pkgdir, strerror(errno));
		unlink(pkgdir);
		return -errno;
	}
	if (mkdir(libdir, 0755) < 0) {
		ALOGE("cannot create dir '%s': %s\n", libdir, strerror(errno));
		unlink(pkgdir);
		return -errno;
	}
	if (chmod(libdir, 0755) < 0) {
		ALOGE("cannot chmod dir '%s': %s\n", libdir, strerror(errno));
		unlink(libdir);
		unlink(pkgdir);
		return -errno;
	}
	if (chown(libdir, AID_SYSTEM, AID_SYSTEM) < 0) {
		ALOGE("cannot chown dir '%s': %s\n", libdir, strerror(errno));
		unlink(libdir);
		unlink(pkgdir);
		return -errno;
	}
	return 0;
}
```

·adb pm -r install xxx.apk`的流程跟踪结束。

**再来看看`MultiDex.install（）`的流程：**

`android.support.multidex.MultiDex.V14#install()`
`android.support.multidex.MultiDex.V14#makeDexElements();`

通过反射调用了`DexPathList`的`makeDexElements()`,该方法就是调用`dex = loadDexFile(file, optimizedDirectory);`

看看`loadDexFile`

```
/**
 * Constructs a {@code DexFile} instance, as appropriate depending 
 * on whether {@code optimizedDirectory} is {@code null}.
 */
private static DexFile loadDexFile(File file, 
    File optimizedDirectory) throws IOException {
    if(optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
}
```

第一次`load dex`文件的时候是`optimizedDirectory`空的 。 进入`return new DexFile(file);`的分支。

```
/*
* Open a DEX file. The value returned is a magic VM cookie. On
* failure, an IOException is thrown.
*/
native private static int openDexFile(String sourceName, String outputName,
			int flags) throws IOException;

```

进入	native	层代码	openDexFile()	

```
static void Dalvik_dalvik_system_DexFile_openDexFile(const u4* args,
			JValue* pResult)
```
该方法比较长 主要是调用了`dvmJarFileOpen()`方法，在`android/dalvik/vm/JarFile.cpp`

```
/*
 * Open a Jar file. It's okay if it's just a Zip archive without all of
 * the Jar trimmings, but we do insist on finding "classes.dex" inside
 * or an appropriately-named ".odex" file alongside.
 *
 * If "isBootstrap" is not set, the optimizer/verifier regards this DEX as
 * being part of a different class loader.
 */
int dvmJarFileOpen(const char* fileName, const char* odexOutputName,
		JarFile** ppJarFile, bool isBootstrap) {
	/*
	 * TODO: This function has been duplicated and modified to become
	 * dvmRawDexFileOpen() in RawDexFile.c. This should be refactored.
	 */
	ZipArchive archive;
	DvmDex* pDvmDex = NULL;
	char* cachedName = NULL;
	bool archiveOpen = false;
	bool locked = false;
	int fd = -1;
	int result = -1;
	/* Even if we're not going to look at the archive, we need to
	 * open it so we can stuff it into ppJarFile.
	 */
	if (dexZipOpenArchive(fileName, &archive) != 0)
		goto bail;
	archiveOpen = true;
	/* If we fork/exec into dexopt, don't let it inherit the archive's fd.
	 */
	dvmSetCloseOnExec(dexZipGetArchiveFd(&archive));
	/* First, look for a ".odex" alongside the jar file. It will
	 * have the same name/path except for the extension.
	 */
	fd = openAlternateSuffix(fileName, "odex", O_RDONLY, &cachedName);
	if (fd >= 0) {
		ALOGV("Using alternate file (odex) for %s ...", fileName);
		if (!dvmCheckOptHeaderAndDependencies(fd, false, 0, 0, true, true)) {
			ALOGE("%s odex has stale dependencies", fileName);
			free(cachedName);
			cachedName = NULL;
			close(fd);
			fd = -1;
			goto tryArchive;
		} else {
			ALOGV("%s odex has good dependencies", fileName);
			//TODO: make sure that the .odex actually corresponds
			// to the classes.dex inside the archive (if present).
			// For typical use there will be no classes.dex.
		}
	} else {
		ZipEntry entry;
		tryArchive:
		/*
		 * Pre-created .odex absent or stale. Look inside the jar for a
		 * "classes.dex".
		 */
		entry = dexZipFindEntry(&archive, kDexInJarName);
		if (entry != NULL) {
			bool newFile = false;
			/*
			 * We've found the one we want. See if there's an up-to-date copy
			 * in the cache.
			 *
			 * On return, "fd" will be seeked just past the "opt" header.
			 *
			 * If a stale .odex file is present and classes.dex exists in
			 * the archive, this will *not* return an fd pointing to the
			 * .odex file; the fd will point into dalvik-cache like any
			 * other jar.
			 */
			if (odexOutputName == NULL) {
				cachedName = dexOptGenerateCacheFileName(fileName,
						kDexInJarName);
				if (cachedName == NULL)
					goto bail;
			} else {
				cachedName = strdup(odexOutputName);
			}
			ALOGV("dvmJarFileOpen: Checking cache for %s (%s)", fileName,
					cachedName);
			fd = dvmOpenCachedDexFile(fileName, cachedName,
					dexGetZipEntryModTime(&archive, entry),
					dexGetZipEntryCrc32(&archive, entry), isBootstrap, &newFile, /*createIfMissing=*/
					true);
			if (fd < 0) {
				ALOGI("Unable to open or create cache for %s (%s)", fileName,
						cachedName);
				goto bail;
			}
			locked = true;
			/*
			 * If fd points to a new file (because there was no cached version,
			 * or the cached version was stale), generate the optimized DEX.
			 * The file descriptor returned is still locked, and is positioned
			 * just past the optimization header.
			 */
			if (newFile) {
				u8 startWhen, extractWhen, endWhen;
				bool result;
				off_t dexOffset;
				dexOffset = lseek(fd, 0, SEEK_CUR);
				result = (dexOffset > 0);
				if (result) {
					startWhen = dvmGetRelativeTimeUsec();
					result = dexZipExtractEntryToFile(&archive, entry, fd) == 0;
					extractWhen = dvmGetRelativeTimeUsec();
				}
				if (result) {
					result = dvmOptimizeDexFile(fd, dexOffset,
							dexGetZipEntryUncompLen(&archive, entry), fileName,
							dexGetZipEntryModTime(&archive, entry),
							dexGetZipEntryCrc32(&archive, entry), isBootstrap);
				}
				if (!result) {
					ALOGE("Unable to extract+optimize DEX from '%s'", fileName);
					goto bail;
				}
				endWhen = dvmGetRelativeTimeUsec();
				ALOGD("DEX prep '%s': unzip in %dms, rewrite %dms", fileName,
						(int) (extractWhen - startWhen) / 1000,
						(int) (endWhen - extractWhen) / 1000);
			}
		} else {
			ALOGI("Zip is good, but no %s inside, and no valid .odex "
					"file in the same directory", kDexInJarName);
			goto bail;
		}
	}
	/*
	 * Map the cached version. This immediately rewinds the fd, so it
	 * doesn't have to be seeked anywhere in particular.
	 */
	if (dvmDexFileOpenFromFd(fd, &pDvmDex) != 0) {
		ALOGI("Unable to map %s in %s", kDexInJarName, fileName);
		goto bail;
	}
	if (locked) {
		/* unlock the fd */
		if (!dvmUnlockCachedDexFile(fd)) {
			/* uh oh -- this process needs to exit or we'll wedge the system */
			ALOGE("Unable to unlock DEX file");
			goto bail;
		}
		locked = false;
	}
	ALOGV("Successfully opened '%s' in '%s'", kDexInJarName, fileName);
	*ppJarFile = (JarFile*) calloc(1, sizeof(JarFile));
	(*ppJarFile)->archive = archive;
	(*ppJarFile)->cacheFileName = cachedName;
	(*ppJarFile)->pDvmDex = pDvmDex;
	cachedName = NULL; // don't free it below
	result = 0;
	bail:
	/* clean up, closing the open file */
	if (archiveOpen && result != 0)
		dexZipCloseArchive(&archive);
	free(cachedName);
	if (fd >= 0) {
		if (locked)
			(void) dvmUnlockCachedDexFile(fd);
		close(fd);
	}
	return result;
}
```

其中 , 执行了`dvmOpenCachedDexFile（）`后`newFile`为`true`。 会执行

```
result = dvmOptimizeDexFile(fd, dexOffset,
			dexGetZipEntryUncompLen(&archive, entry),
			fileName,
			dexGetZipEntryModTime(&archive, entry),
			dexGetZipEntryCrc32(&archive, entry),
			isBootstrap);
 ```
 
这里调用了`/dalvik/vm/RawDexFile.cpp`中的`dvmOptimizeDexFile()`方法

```
/*
 * Given a descriptor for a file with DEX data in it, produce an
 * optimized version.
 *
 * The file pointed to by "fd" is expected to be a locked shared resource
 * (or private); we make no efforts to enforce multi-process correctness
 * here.
 *
 * "fileName" is only used for debug output. "modWhen" and "crc" are stored
 * in the dependency set.
 *
 * The "isBootstrap" flag determines how the optimizer and verifier handle
 * package-scope access checks. When optimizing, we only load the bootstrap
 * class DEX files and the target DEX, so the flag determines whether the
 * target DEX classes are given a (synthetic) non-NULL classLoader pointer.
 * This only really matters if the target DEX contains classes that claim to
 * be in the same package as bootstrap classes.
 *
 * The optimizer will need to load every class in the target DEX file.
 * This is generally undesirable, so we start a subprocess to do the
 * work and wait for it to complete.
 *
 * Returns "true" on success. All data will have been written to "fd".
 */
bool dvmOptimizeDexFile(int fd, off_t dexOffset, long dexLength,
		const char* fileName, u4 modWhen, u4 crc, bool isBootstrap) {
	const char* lastPart = strrchr(fileName, '/');
	if (lastPart != NULL)
		lastPart++;
	else
		lastPart = fileName;
	ALOGD("DexOpt: --- BEGIN '%s' (bootstrap=%d) ---", lastPart, isBootstrap);
	pid_t pid;
	/*
	 * This could happen if something in our bootclasspath, which we thought
	 * was all optimized, got rejected.
	 */
	if (gDvm.optimizing) {
		ALOGW("Rejecting recursive optimization attempt on '%s'", fileName);
		return false;
	}
	pid = fork();
	if (pid == 0) {
		static const int kUseValgrind = 0;
		static const char* kDexOptBin = "/bin/dexopt";
		static const char* kValgrinder = "/usr/bin/valgrind";
		static const int kFixedArgCount = 10;
		static const int kValgrindArgCount = 5;
		static const int kMaxIntLen = 12; // '-'+10dig+'\0' -OR- 0x+8dig
		int bcpSize = dvmGetBootPathSize();
		int argc = kFixedArgCount + bcpSize
				+ (kValgrindArgCount * kUseValgrind);
		const char* argv[argc + 1]; // last entry is NULL
		char values[argc][kMaxIntLen];
		char* execFile;
		const char* androidRoot;
		int flags;
		/* change process groups, so we don't clash with ProcessManager */
		setpgid(0, 0);
		/* full path to optimizer */
		androidRoot = getenv("ANDROID_ROOT");
		if (androidRoot == NULL) {
			ALOGW("ANDROID_ROOT not set, defaulting to /system");
			androidRoot = "/system";
		}
		execFile = (char*) alloca(strlen(androidRoot) + strlen(kDexOptBin) + 1);
		strcpy(execFile, androidRoot);
		strcat(execFile, kDexOptBin);
		/*
		 * Create arg vector.
		 */
		int curArg = 0;
		if (kUseValgrind) {
			/* probably shouldn't ship the hard-coded path */
			argv[curArg++] = (char*) kValgrinder;
			argv[curArg++] = "--tool=memcheck";
			argv[curArg++] = "--leak-check=yes"; // check for leaks too
			argv[curArg++] = "--leak-resolution=med"; // increase from 2 to 4
			argv[curArg++] = "--num-callers=16"; // default is 12
			assert(curArg == kValgrindArgCount);
		}
		argv[curArg++] = execFile;
		argv[curArg++] = "--dex";
		sprintf(values[2], "%d", DALVIK_VM_BUILD);
		argv[curArg++] = values[2];
		sprintf(values[3], "%d", fd);
		argv[curArg++] = values[3];
		sprintf(values[4], "%d", (int) dexOffset);
		argv[curArg++] = values[4];
		sprintf(values[5], "%d", (int) dexLength);
		argv[curArg++] = values[5];
		argv[curArg++] = (char*) fileName;
		sprintf(values[7], "%d", (int) modWhen);
		argv[curArg++] = values[7];
		sprintf(values[8], "%d", (int) crc);
		argv[curArg++] = values[8];
		flags = 0;
		if (gDvm.dexOptMode != OPTIMIZE_MODE_NONE) {
			flags |= DEXOPT_OPT_ENABLED;
			if (gDvm.dexOptMode == OPTIMIZE_MODE_ALL)
				flags |= DEXOPT_OPT_ALL;
		}
		if (gDvm.classVerifyMode != VERIFY_MODE_NONE) {
			flags |= DEXOPT_VERIFY_ENABLED;
			if (gDvm.classVerifyMode == VERIFY_MODE_ALL)
				flags |= DEXOPT_VERIFY_ALL;
		}
		if (isBootstrap)
			flags |= DEXOPT_IS_BOOTSTRAP;
		if (gDvm.generateRegisterMaps)
			flags |= DEXOPT_GEN_REGISTER_MAPS;
		sprintf(values[9], "%d", flags);
		argv[curArg++] = values[9];
		assert(
				((!kUseValgrind && curArg == kFixedArgCount)
						|| ((kUseValgrind
								&& curArg == kFixedArgCount + kValgrindArgCount))));
		ClassPathEntry* cpe;
		for (cpe = gDvm.bootClassPath; cpe->ptr != NULL; cpe++) {
			argv[curArg++] = cpe->fileName;
		}
		assert(curArg == argc);
		argv[curArg] = NULL;
		if (kUseValgrind)
		execv(kValgrinder, const_cast<char**>(argv));
		else
		execv(execFile, const_cast<char**>(argv));
		ALOGE("execv '%s'%s failed: %s", execFile,
				kUseValgrind ? " [valgrind]" : "", strerror(errno));
		exit(1);
	} else {
		ALOGV("DexOpt: waiting for verify+opt, pid=%d", (int) pid);
		int status;
		pid_t gotPid;
		/*
		 * Wait for the optimization process to finish. We go into VMWAIT
		 * mode here so GC suspension won't have to wait for us.
		 */
		ThreadStatus oldStatus = dvmChangeStatus(NULL, THREAD_VMWAIT);
		while (true) {
			gotPid = waitpid(pid, &status, 0);
			if (gotPid == -1 && errno == EINTR) {
				ALOGD("waitpid interrupted, retrying");
			} else {
				break;
			}
		}
		dvmChangeStatus(NULL, oldStatus);
		if (gotPid != pid) {
			ALOGE("waitpid failed: wanted %d, got %d: %s", (int) pid,
					(int) gotPid, strerror(errno));
			return false;
		}
		if (WIFEXITED(status) && WEXITSTATUS(status) == 0) {
			ALOGD("DexOpt: --- END '%s' (success) ---", lastPart);
			return true;
		} else {
			ALOGW("DexOpt: --- END '%s' --- status=0x%04x, process failed",
					lastPart, status);
			return false;
		}
	}
}
```

终于找到`ALOGD("DexOpt: --- BEGIN '%s' (bootstrap=%d) ---", lastPart, isBootstrap);`这句日志的出处了！

该方法的注释写的也比较详细了，其中`pid=fork();`表明`dexopt`确实开启了新的进程来完成，而且当前进程是使用`while`循环等待这个子进程返回结果的。

那么之前关于`dexopt`的所有疑惑都解开了。
 
