---
title: SharedPreferences解读
date: 2021-04-30 14:22:41
tags: [Android]
---

# SharedPreferences解读

>SharedPreferences是我们平时常用的简单储存工具。优点就是用起来方便，而且线程安全，甚至有的公司干脆连sqLite都不用了，全用SP！但是突然有一天腾讯弄出了个MMKV用来替代他，最近google也弄了个DataStorage，而且最近面试也是必问原理，于是我决定死磕一下SP。

## 初始化
### getSharedPreferences

我们通过这个方法拿到SP，该方法在Activity中内置，最终调用的是ContextWrapper的方法：

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {
	return mBase.getSharedPreferences(name, mode);
}
```

ContextWrapper里面调用的mBase的对应方法，通过百度，查到mBase其实就是传说中的`ContextImpl`：

```java
@Override
public SharedPreferences getSharedPreferences(String name, int mode) {

	if (mPackageInfo.getApplicationInfo().targetSdkVersion <
			Build.VERSION_CODES.KITKAT) {
		if (name == null) {
			name = "null";
		}
	}

	File file;
	synchronized (ContextImpl.class) {
		if (mSharedPrefsPaths == null) {
			mSharedPrefsPaths = new ArrayMap<>();
		}
		file = mSharedPrefsPaths.get(name);
		if (file == null) {
			file = getSharedPreferencesPath(name);
			
			//拿到文件后加入缓存，下次再拿文件就快了
			mSharedPrefsPaths.put(name, file);
		}
	}
	//调用重载方法执行真正的逻辑
	return getSharedPreferences(file, mode);
}
```

这个方法主要是用来获取文件，并且把文件放入ArrayMap缓存起来以备以后再次获取file可以提高效率。key就是文件的path，value就是文件映射的File对象。

然后使用文件作为参数，调用了同名的重载方法。

### 生成文件

上述代码中出现了一个`file`局部变量，它是个什么角色呢，下一步我们就要来揭开它神秘的面纱。

首先看到`Synchronized`关键字，并且充当了`ContextIml`的类锁，那么带来线程安全的同时，会使所有并发访问该类的操作阻塞。那么会不会造成UI线程卡顿呢？会，但是不明显。

`mSharedPrefsPaths = new ArrayMap<>();`，那么不用想，mSharedPrefsPaths是个ArrayMap，一定是个内存缓存。

此处代码不重要，请直接看下面文字结论。

```java
/**
*	拿到SP文件地址
**/
@Override
public File getSharedPreferencesPath(String name) {
	return makeFilename(getPreferencesDir(), name + ".xml");
}

/**
*	拿到SP所在目录
**/
@UnsupportedAppUsage
private File getPreferencesDir() {
	synchronized (mSync) {
		if (mPreferencesDir == null) {
		
			//创建File对象
			mPreferencesDir = new File(getDataDir(), "shared_prefs");
		}
		//确保文件一定存在
		return ensurePrivateDirExists(mPreferencesDir);
	}
}

/**
*	拿到数据目录，里面包括SP文件
**/
@Override
public File getDataDir() {
	if (mPackageInfo != null) {
		File res = null;
		if (isCredentialProtectedStorage()) {
			res = mPackageInfo.getCredentialProtectedDataDirFile();
		} else if (isDeviceProtectedStorage()) {
			res = mPackageInfo.getDeviceProtectedDataDirFile();
		} else {
			res = mPackageInfo.getDataDirFile();
		}

		if (res != null) {
			if (!res.exists() && android.os.Process.myUid() == android.os.Process.SYSTEM_UID) {
				Log.wtf(TAG, "Data directory doesn't exist for package " + getPackageName(),
						new Throwable());
			}
			return res;
		} else {
			throw new RuntimeException(
					"No data directory found for package " + getPackageName());
		}
	} else {
		throw new RuntimeException(
				"No package details found for package " + getPackageName());
	}
}

private File makeFilename(File base, String name) {
	if (name.indexOf(File.separatorChar) < 0) {
		final File res = new File(base, name);
		// We report as filesystem access here to give us the best shot at
		// detecting apps that will pass the path down to native code.
		BlockGuard.getVmPolicy().onPathAccess(res.getPath());
		return res;
	}
	throw new IllegalArgumentException(
			"File " + name + " contains a path separator");
}
```

上述代码没必要研究，就是经过一系列拼拼凑凑，最后拿到/data/data/包名/shared_prefs/[name].xml 文件，然后调用`ensurePrivateDirExists`方法确保文件一定存在(不存在就创建)。

### getSharedPreferences 重载方法

接下来调用了一个重载方法，主要是创建`SharedPreferenceImpl`对象，通过缓存进arrayMap，并且以file作为key一一绑定。

```java
@Override
public SharedPreferences getSharedPreferences(File file, int mode) {
	SharedPreferencesImpl sp;
	synchronized (ContextImpl.class) {
		//获取缓存，该缓存把file和SharedPreferencesImpl一一绑定
		final ArrayMap<file, sharedpreferencesimpl> cache = getSharedPreferencesCacheLocked();
		sp = cache.get(file);
		if (sp == null) {
			checkMode(mode);
			
			//创建SharedPreferencesImpl
			sp = new SharedPreferencesImpl(file, mode);
			//缓存SP
			cache.put(file, sp);
			return sp;
		}
	}

	//如果开启多进程模式，或者小于3.0版本，则每次都重新加载文件
	if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
		getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
		
		sp.startReloadIfChangedUnexpectedly();
	}
	return sp;
}

private ArrayMap<file, sharedpreferencesimpl> getSharedPreferencesCacheLocked() {
	if (sSharedPrefsCache == null) {
		sSharedPrefsCache = new ArrayMap<>();
	}

	final String packageName = getPackageName();
	ArrayMap<file, sharedpreferencesimpl> packagePrefs = sSharedPrefsCache.get(packageName);
	if (packagePrefs == null) {
		packagePrefs = new ArrayMap<>();
		sSharedPrefsCache.put(packageName, packagePrefs);
	}

	return packagePrefs;
}
```

这个重载方法，主要是创建`SharedPreferencesImpl`对象并且缓存，File作为key，SharedPreferencesImpl作为value，保证了File和SharedPreferencesImpl的一一对应。

### SharedPreferencesImpl构造方法

```java
@UnsupportedAppUsage
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    //创建备份文件对象
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    
    //开始加载
    startLoadFromDisk();
}

/**
* 创建一个同名文件，以.bak作为后缀
**/
static File makeBackupFile(File prefsFile) {
    return new File(prefsFile.getPath() + ".bak");
}
```

在构造方法中，在sp所在目录中先创建了一个以“.bak”为后缀的备份文件的对象，然后在子线程中从硬盘加载文件：

```java
/**
*在子线程加载文件
**/
@UnsupportedAppUsage
private void startLoadFromDisk() {
    synchronized (mLock) {
		// 变更标志位
        mLoaded = false;
    }
	//开启子线程从硬盘加载文件
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}


private void loadFromDisk() {
    synchronized (mLock) {
		//dlc机制
        if (mLoaded) {
            return;
        }
        
        //如果备份文件实体存在，则从备份恢复sp文件
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }


    Map<string, object="object"> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
            
                //读取文件内容
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16 * 1024);
                
                //采用poll方式解析xml
                map = (Map<string, object="object">) XmlUtils.readMapXml(str);
            } catch (Exception e) {
                Log.w(TAG, "Cannot read " + mFile.getAbsolutePath(), e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        // An errno exception means the stat failed. Treat as empty/non-existing by
        // ignoring.
    } catch (Throwable t) {
        thrown = t;
    }

	//注意这个锁
    synchronized (mLock) {
		//变更标志位
        mLoaded = true;
        mThrowable = thrown;

        try {
            if (thrown == null) {
                if (map != null) {
                    mMap = map;
                    mStatTimestamp = stat.st_mtim;
                    mStatSize = stat.st_size;
                } else {
                    mMap = new HashMap<>();
                }
            }
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
        
            //唯一唤醒锁的时机
            mLock.notifyAll();
        }
    }
}
```

#### getXXX()

```java
@Override
@Nullable
public String getString(String key, @Nullable String defValue) {
	//注意这个锁
	synchronized (mLock) {
		//持续等待
		awaitLoadedLocked();
		String v = (String)mMap.get(key);
		return v != null ? v : defValue;
	}
}

@GuardedBy("mLock")
private void awaitLoadedLocked() {
	if (!mLoaded) {
		BlockGuard.getThreadPolicy().onReadFromDisk();
	}
	while (!mLoaded) {
		try {
			mLock.wait();
		} catch (InterruptedException unused) {
		}
	}
	if (mThrowable != null) {
		throw new IllegalStateException(mThrowable);
	}
}
```

这里要注意的就是，`getXXX()`方法与构造方法中读取文件的地方是共享同一把锁`mLock`，从而使这两个方法存在互斥关系。如果文件很大，那么程序运行`sharedPreferenceImpl`构造方法的耗时就会比较长，此时调用`getXXX()`方法因为互斥锁的关系，就会持续等待，一直到构造方法执行完毕之后才能返回结果，导致主线程被阻塞，出现UI卡顿，甚至出现ANR。

### 阶段总结

`getSharedPreferences(String name,int mode)`方法用来初始化SP，主要就是创建xml文件，进而使用文件和mode创建`getSharedPreferencesImpl`对象并且返回。

而在创建`getSharedPreferencesImpl`时就会开启一个子线程从硬盘解析xml文件到内存中，同时创建一个以.bat为后缀的备份文件。

如果在构造方法读取文件期间调用了`getXXX()`方法，后者会持续等待前者完成之后才执行，会造成UI卡顿甚至ANR。

当开启**多进程模式**时，每次调用`getSharedPreferences()`时都会重新读取文件。因为SP不知道文件是否已被更新，所以只能每次重新读取。


## Editor

### 获取
按照SP的使用流程，下一步就是要获取到`Editor`对象了，我们通过`Editor`提供的`putXXX()`方法来向SP中写入数据。

`Editor`是一个接口，提供了一些列的put虚方法。接口是不能直接调用的，所以我们调用的其实是他的实现类——`EditorImpl`，它是`getSharedPreferencesImpl`的**内部匿名类**，由于内部匿名类持有外部类的引用，那么它就可以访问外层类的成员！

```java
@Override
public Editor edit() {

	//又看到了这把锁
	synchronized (mLock) {
		//持续等待
		awaitLoadedLocked();
	}
	//创建EditorImpl
	return new EditorImpl();
}
```

每次调用`sp.editor()`方法，也会共享`mLock`这把锁，那么同样也存在上面说的问题，会出现卡顿。

### EditorImpl定义和Put操作

```java
 public final class EditorImpl implements Editor {
 	//一把锁
	private final Object mEditorLock = new Object();

	//缓存
	@GuardedBy("mEditorLock")
	private final Map<string, object> mModified = new HashMap<>();

	//标志位
	@GuardedBy("mEditorLock")
	private boolean mClear = false;

	//喜闻乐见的putString
	@Override
	public Editor putString(String key, @Nullable String value) {
		synchronized (mEditorLock) {
			mModified.put(key, value);
			return this;
		}
	}
	
......
```

代码很简单，put操作其实就是对HashMap缓存进行写入，同时为了保证线程安全，加了锁。每个PUT操作都是互斥关系。改动并没有即时提交到硬盘文件，也没有提交到真正的缓存mMap，而是提交到了editor内部的缓存**mModified**里面了。

### remove

```java
@Override
public Editor remove(String key) {
	synchronized (mEditorLock) {
		//为何不设置为null?;
		mModified.put(key, this);
		return this;
	}
}
```

当我看到`remove()`方法定义时，我惊呆了！mModified里面放了this，而this又持有了mModified，这不就造成循环引用了吗，不会出问题吗？希望日后能找到答案吧。

我觉得一般人都会设置null吧。

### clear

```java
@Override
public Editor clear() {
	synchronized (mEditorLock) {
		mClear = true;
		return this;
	}
}
```

`clear`方法也是非常奇怪，只是设置了一下标志位，并没有做其他的事情。

#### 小结

怎样解决卡顿问题？

1. 尽量把大文件拆分成多个小文件。
2. 尽量提前初始化sp。
3. 然后尽量不要在初始化之后马上就用sp。

### commit()

```java
@Override
public boolean commit() {

	//提交到内存
	MemoryCommitResult mcr = commitToMemory();
	//子线程写入文件
	SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
	try {
		//计数锁等待
		mcr.writtenToDiskLatch.await();
	} catch (InterruptedException e) {
		return false;
	} finally {

	}

	//通知监视器
	notifyListeners(mcr);

	//返回读写结果true or false
	return mcr.writeToDiskResult;
}
```

commit方法通过计数锁阻塞了当前线程，等待子线程中的写操作完成，才会被唤醒，充分体现了线程协作的思想。

### apply()

```java
@Override
public void apply() {

	//提交到内存
	final MemoryCommitResult mcr = commitToMemory();

	//其实并不起作用，迷之操作
	final Runnable awaitCommit = new Runnable() {
			@Override
			public void run() {
				try {
					//计数锁开始等待
					mcr.writtenToDiskLatch.await();
				} catch (InterruptedException ignored) {
				}
			}
		};

	//添加生命周期回调
	QueuedWork.addFinisher(awaitCommit);

	Runnable postWriteRunnable = new Runnable() {
			@Override
			public void run() {

				//利用计数锁阻塞当前线程
				awaitCommit.run();
				//移除生命周期回调
				QueuedWork.removeFinisher(awaitCommit);
			}
		};

	//子线程写入文件
	SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

	//通知监视器
	notifyListeners(mcr);
}
```

### 小结

这两个方法主要干了三件事：

1. 调用`commitToMemory()`方法从`mModified`提交改动到`mMap`；
2. 调用`SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);`把改动写入文件；
3. 调用`notifyListeners(mcr);`通知监视器。

不同点在于，`commit`方法在写入文件时，通过计数锁阻塞了当前线程，那么如果在主线程调用`commit`的话，就会造成卡顿了。而`apply`方法是把计数锁放到了另一个子线程中去执行了，所以阻塞的是另一个子线程，在一般情况下对主线程没有影响。

但是，当生命周期处于`handleStopService()` 、 `handlePauseActivity()` 、 `handleStopActivity()`时，会调用 `QueuedWork`的`waitToFinish()`方法，在这个方法中会遍历执行所有的`finisher`，所以如果内容很多，**`Apply`方法还是会引起ANR**。

最后，`apply`方法没有返回写入的成功失败。

怎么解决ANR问题？

我觉得在子线程中调用commit方法应该是个不错的选择吧。

### enqueueDiskWrite

`commit`和`apply`两个方法都是在主线程调用了这个方法完成写入：

```java
/**
*使已经提交给内存的结果排队，以将其写入磁盘。它们将按入队的顺序一次写入磁盘
**/
private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
	//commit-true apply-false
	final boolean isFromSyncCommit = (postWriteRunnable == null);

	final Runnable writeToDiskRunnable = new Runnable() {
			@Override
			public void run() {
				synchronized (mWritingToDiskLock) {
					writeToFile(mcr, isFromSyncCommit);
				}
				synchronized (mLock) {

					//文件写完之后才会减1
					mDiskWritesInFlight--;
				}
				//apply会执行
				if (postWriteRunnable != null) {
					//主要是为了移除Finisher
					postWriteRunnable.run();
				}
			}
		};

	// commit执行
	if (isFromSyncCommit) {
		boolean wasEmpty = false;
		synchronized (mLock) {
			//如果在commit之后紧接着又调用了commit，那么这里就是false了
			wasEmpty = mDiskWritesInFlight == 1;
		}
		if (wasEmpty) {
			writeToDiskRunnable.run();
			return;
		}
	}

	// apply执行；频繁commit时后面的commit会转为异步执行
	QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```

在这里先定义了一个`Runnable`叫做writeToDiskRunnable ，一看名字就知道是用来IO写磁盘的。所以其中必然有`writeToFile`方法的调用啦。由于在同一线程顺序执行，`writeToFile`方法中必然会把`mcr.writtenToDiskLatch.countDown()`，所以后面的`postWriteRunnable.run();`只是为了移除finisher，而之前的`awaitCommit`这个Runnable中的`mcr.writtenToDiskLatch.await();`并没有起什么作用。

我们看到，系统通过变换`mDiskWritesInFlight`尽可能把`commit`转换为`apply`去执行。

`commit`在当前线程就直接`run`了，而`apply`则是提交给了`QueuedWork.queue(writeToDiskRunnable,false)`。

这里要注意`mDiskWritesInFlight`这个计数器只有在`commitToMemory()`方法中才会++，只有在`writeToFile()`完成后才会--，所以当commit之后，紧接着再次调用commit，那么之后的commit都会转成异步方式执行写入，而异步是通过HandlerThread实现的（8.0以前则是通过SingleThreadPool实现），`QueuedWork.queue()`方法是把任务提交到了一个静态队列里面，由HandlerThread顺序执行。

### commitToMemory

```java
// Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
	long memoryStateGeneration;
	boolean keysCleared = false;
	List<String> keysModified = null;
	Set<OnSharedPreferenceChangeListener> listeners = null;
	Map<String, Object> mapToWriteToDisk;

	//注意这把锁
	synchronized (SharedPreferencesImpl.this.mLock) {
		
		if (mDiskWritesInFlight > 0) {
			mMap = new HashMap<String, Object>(mMap);
		}

		// 转换对象
		mapToWriteToDisk = mMap;
		mDiskWritesInFlight++;

		//同样是转换对象
		boolean hasListeners = mListeners.size() > 0;
		if (hasListeners) {
			keysModified = new ArrayList<String>();
			listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
		}

		//又一把锁
		synchronized (mEditorLock) {
			boolean changesMade = false;

			//注意这个标志位，为true说明调用过editor.clear()方法
			if (mClear) {
				if (!mapToWriteToDisk.isEmpty()) {
					changesMade = true;
					mapToWriteToDisk.clear();
				}
				keysCleared = true;
				mClear = false;
			}

			//遍历拷贝
			for (Map.Entry<String, Object> e : mModified.entrySet()) {
				String k = e.getKey();
				Object v = e.getValue();
				if (v == this || v == null) {//说明被editor.remove()过了
					if (!mapToWriteToDisk.containsKey(k)) {
						continue;
					}
					mapToWriteToDisk.remove(k);
				} else {
					if (mapToWriteToDisk.containsKey(k)) {
						Object existingValue = mapToWriteToDisk.get(k);
						if (existingValue != null && existingValue.equals(v)) {//如果值没变就跳过
							continue;
						}
					}
					mapToWriteToDisk.put(k, v);
				}

				changesMade = true;
				if (hasListeners) {
					keysModified.add(k);
				}
			}

			//清除editor缓存，释放内存
			mModified.clear();

			if (changesMade) {
				mCurrentMemoryStateGeneration++;
			}

			memoryStateGeneration = mCurrentMemoryStateGeneration;
		}
	}

	//返回结果
	return new MemoryCommitResult(memoryStateGeneration, keysCleared, keysModified,
			listeners, mapToWriteToDisk);
}
```

`commitToMemory`方法顾名思义就是把`editor`的`mModified`中的值提交到`sharedferenceImpl`的`mMap`。

这里加了`SharedPreferencesImpl.this.mLock`，说明多次调用`commit`或`apply`方法都会阻塞执行，而且在进行`commit`或者`apply`还没结束时就调用`getXXX()`，依然会造成卡顿。

而这里出现的第二把锁`mEditorLock`是为了保证`mModified`的线程安全。



#### MemoryCommitResult

MemoryCommitResult是一个打包类，它把当前的内存缓存、内存状态、监听回调、写入结果全部装箱到一起。

```java
private static class MemoryCommitResult {

	//内存状态版本号
	final long memoryStateGeneration;
	final boolean keysCleared;
	@Nullable final List<string> keysModified;
	@Nullable final Set<onsharedpreferencechangelistener> listeners;
	final Map<string, object> mapToWriteToDisk;
	final CountDownLatch writtenToDiskLatch = new CountDownLatch(1);

	@GuardedBy("mWritingToDiskLock")
	volatile boolean writeToDiskResult = false;
	boolean wasWritten = false;

	//私有构造方法
	private MemoryCommitResult(long memoryStateGeneration, boolean keysCleared,
			@Nullable List<string> keysModified,
			@Nullable Set<onsharedpreferencechangelistener> listeners,
			Map<string, object> mapToWriteToDisk) {
		this.memoryStateGeneration = memoryStateGeneration;
		this.keysCleared = keysCleared;
		this.keysModified = keysModified;
		this.listeners = listeners;
		this.mapToWriteToDisk = mapToWriteToDisk;
	}
	
	//唯一提供的方法，用来变更标记为，储存写入结果
	//只有这里能唤醒计数锁！！！
	void setDiskWriteResult(boolean wasWritten, boolean result) {
		this.wasWritten = wasWritten;
		writeToDiskResult = result;
		writtenToDiskLatch.countDown();
	}
}
```

### apply方法的QueuedWork机制

```java
/**
 * @hide 很奇怪为啥好东西都不让我们用呢？
 */
public class QueuedWork {

    private static final long DELAY = 100;

    /** 我是一把锁 */
    private static final Object sLock = new Object();

    
    private static Object sProcessingWork = new Object();

    @GuardedBy("sLock")
    @UnsupportedAppUsage
    private static final LinkedList<Runnable> sFinishers = new LinkedList<>();

    @GuardedBy("sLock")
    private static Handler sHandler = null;

    /** Work queued via {@link #queue} */
    @GuardedBy("sLock")
    private static final LinkedList<Runnable> sWork = new LinkedList<>();

    @GuardedBy("sLock")
    private static boolean sCanDelay = true;


    /**
     * Add task
     */
    @UnsupportedAppUsage
    public static void addFinisher(Runnable finisher) {
        synchronized (sLock) {
            sFinishers.add(finisher);
        }
    }

    /**
     * Remove task
     */
    @UnsupportedAppUsage
    public static void removeFinisher(Runnable finisher) {
        synchronized (sLock) {
            sFinishers.remove(finisher);
        }
    }

    /**
     * 在主线程立即处理剩余任务
     */
    public static void waitToFinish() {

        boolean hadMessages = false;

        Handler handler = getHandler();

        synchronized (sLock) {

			//解除loop
            if (handler.hasMessages(QueuedWorkHandler.MSG_RUN)) {

				//移除消息，使handlerThread剩余的任务不再执行了
                handler.removeMessages(QueuedWorkHandler.MSG_RUN);
            }

            // We should not delay any work as this might delay the finishers
            sCanDelay = false;
        }

		//线程使用自检
        StrictMode.ThreadPolicy oldPolicy = StrictMode.allowThreadDiskWrites();
        try {
			//当生命周期变化时，系统会主动触发写入，这可是在主线程哦
            processPendingWork();
        } finally {
			//允许在当前线程进行文件写入操作
            StrictMode.setThreadPolicy(oldPolicy);
        }

		//事后检查机制，保证所有finisher都被执行
        try {
            while (true) {
                Runnable finisher;

                synchronized (sLock) {
                    finisher = sFinishers.poll();
                }

				//由于在同一线程中先调用了processPendingWork()，完成写操作后就会remove掉finisher，所以大多数情况会走到这里
                if (finisher == null) {
                    break;
                }

                finisher.run();
            }
        } finally {
            sCanDelay = true;
        }
    }
	//同一个方法在7.0上的实现
	// 在一个单线程的线程池中执行，就单纯的靠countDownLatch.await()来等待全部任务执行完毕。
	public static void waitToFinish() {
        Runnable toFinish;
        while ((toFinish = sPendingWorkFinishers.poll()) != null) {
          toFinish.run();
        }
   }

    /**
     * 一看就懂
     */
    public static boolean hasPendingWork() {
        synchronized (sLock) {
            return !sWork.isEmpty();
        }
    }

	/**
     * 异步任务调度,其实就是切换到工作线程
	 * @param shouldDelay	apply=false
     */
    @UnsupportedAppUsage
    public static void queue(Runnable work, boolean shouldDelay) {
        Handler handler = getHandler();

        synchronized (sLock) {
            sWork.add(work);

            if (shouldDelay && sCanDelay) {
                handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
            } else {
                handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
            }
        }
    }

	/**
	 *	创建一个HandlerThread，并且返回一个单例的Handler
	 * queue方法和waitToFinish中调用
     */
    @UnsupportedAppUsage
    private static Handler getHandler() {
        synchronized (sLock) {
            if (sHandler == null) {
                HandlerThread handlerThread = new HandlerThread("queued-work-looper",
                        Process.THREAD_PRIORITY_FOREGROUND);
                handlerThread.start();

                sHandler = new QueuedWorkHandler(handlerThread.getLooper());
            }
            return sHandler;
        }
    }

	/**
	 *	自定义的Handler
	**/
    private static class QueuedWorkHandler extends Handler {
        static final int MSG_RUN = 1;

        QueuedWorkHandler(Looper looper) {
            super(looper);
        }

        public void handleMessage(Message msg) {
            if (msg.what == MSG_RUN) {
				//切换到了handlerThread所在线程
                processPendingWork();
            }
        }
    }

	/**
	*	用来处理挂起的任务，核心方法，遍历执行task，此时已切换到工作线程
	**/
    private static void processPendingWork() {

		//这里又是一把锁
        synchronized (sProcessingWork) {
            LinkedList<Runnable> work;
			//这里的锁注意一下
            synchronized (sLock) {
				//浅拷贝，这里的work其实是writeToDiskRunnable，包含文件的写入操作和finisher调用
                work = (LinkedList<Runnable>) sWork.clone();
				//释放内存
                sWork.clear();
				//释放msg
                getHandler().removeMessages(QueuedWorkHandler.MSG_RUN);
            }

			//遍历执行task
            if (work.size() > 0) {
                for (Runnable w : work) {
                    w.run();
                }
            }
        }
    }
}
```

Activity在OnPause时会调用ActivityThread的`handleStopActivity`方法:

```java
@Override
public void handleStopActivity(IBinder token, int configChanges,
		PendingTransactionActions pendingActions, boolean finalStateRequest, String reason) {
	final ActivityClientRecord r = mActivities.get(token);
	r.activity.mConfigChangeFlags |= configChanges;

	final StopInfo stopInfo = new StopInfo();
	performStopActivityInner(r, stopInfo, true /* saveState */, finalStateRequest,
			reason);

	updateVisibility(r, false);

	// 3.0及以后版本，确保所有挂起的写入全都提交到文件
	if (!r.isPreHoneycomb()) {
		QueuedWork.waitToFinish();
	}

	stopInfo.setActivity(r);
	stopInfo.setState(r.state);
	stopInfo.setPersistentState(r.persistentState);
	pendingActions.setStopInfo(stopInfo);
	mSomeActivitiesChanged = true;
}
```

3.0及以后系统，当发生cresh或者Activity、broaderCaster、service生命周期发生改变时，主线程会自动调用`QueuedWork.waitToFinish()`把当前挂起的修改写入到文件系统中！！！如果没有任何修改，那么就会遍历执行所有的finisher！！！这就是`apply`方法提交也会导致ANR的秘密！！！

如果过多的使用apply，可能在activity生命周期发生改变时，没能执行完全部的apply，那么就会在生命周期时自动执行余下的apply，保证异步任务不会丢失。

#### 最重要的方法writeToFile

##### 备份文件

```java
// 判断是否满足写入条件，并且备份文件
if (fileExists) {
	boolean needsWrite = false;

	// 如果硬盘状态版本号低于内存状态版本号，那么只需要写入
	if (mDiskStateGeneration < mcr.memoryStateGeneration) {
		if (isFromSyncCommit) {//commit
			needsWrite = true;//变更标志位
		} else {
			synchronized (mLock) {//apply
				//无需保持中间状态。只需等待最新状态保持不变即可。
				if (mCurrentMemoryStateGeneration == mcr.memoryStateGeneration) {
					needsWrite = true;//变更标志位
				}
			}
		}
	}

	//如果不需要写入，那么直接设置结果
	if (!needsWrite) {
		mcr.setDiskWriteResult(false, true);
		return;
	}

	//判定备份文件是否存在
	boolean backupFileExists = mBackupFile.exists();

	//如果备份不存在，则把文件内容备份
	if (!backupFileExists) {
		if (!mFile.renameTo(mBackupFile)) {
			mcr.setDiskWriteResult(false, false);
			return;
		}
	} else {
		//如果已存在备份，则删掉文件
		mFile.delete();
	}
}
```

##### 真正的写入

```java
// 尝试写入文件，删除备份并尽可能自动地返回true。如果发生任何异常，请删除新文件。下次我们将从备份中还原。
try {
	FileOutputStream str = createFileOutputStream(mFile);

	//如果拿不到输出流就直接返回结果：失败
	if (str == null) {
		mcr.setDiskWriteResult(false, false);
		return;
	}
	
	//执行写入操作
	XmlUtils.writeMapXml(mcr.mapToWriteToDisk, str);

	//保存写入的时间点
	writeTime = System.currentTimeMillis();

	FileUtils.sync(str);

	fsyncTime = System.currentTimeMillis();

	//写完了，关闭输出流
	str.close();
	
	//设置文件权限
	ContextImpl.setFilePermissionsFromMode(mFile.getPath(), mMode, 0);

	try {
		//获取文件系统中的文件状态
		final StructStat stat = Os.stat(mFile.getPath());
		synchronized (mLock) {
			mStatTimestamp = stat.st_mtim;
			mStatSize = stat.st_size;
		}
	} catch (ErrnoException e) {
		// Do nothing
	}

	// 写入成功后删除备份
	mBackupFile.delete();

	mDiskStateGeneration = mcr.memoryStateGeneration;

	//返回结果
	mcr.setDiskWriteResult(true, true);

	long fsyncDuration = fsyncTime - writeTime;
	mSyncTimes.add((int) fsyncDuration);
	mNumSync++;

	return;
} catch (XmlPullParserException e) {
	Log.w(TAG, "writeToFile: Got exception:", e);
} catch (IOException e) {
	Log.w(TAG, "writeToFile: Got exception:", e);
}

// 清理干净没有写入成功的文件
if (mFile.exists()) {
	if (!mFile.delete()) {
		Log.e(TAG, "Couldn't clean up partially-written file " + mFile);
	}
}
//返回结果：失败
mcr.setDiskWriteResult(false, false);
```

这段逻辑真的好长，最后回顾起来却很简单，它只干了3件事情：

- 用xml文件生成一个备份，用来做失败还原
- 用XmlUtil执行写入，写入成功后删除备份返回结果
- 写入失败则清理文件，保留备份

这里还要注意的是，无论如何都会创建一个新的源文件来写入，所以不论是`commit`还是`apply`都是全量更新，效率非常的低！

### notifyListeners

```java
private void notifyListeners(final MemoryCommitResult mcr) {
	if (mcr.listeners == null || (mcr.keysModified == null && !mcr.keysCleared)) {
		return;
	}
	if (Looper.myLooper() == Looper.getMainLooper()) {
	
		//如果当前是主线程，就遍历MemoryCommitResult中listener
		if (mcr.keysCleared && Compatibility.isChangeEnabled(CALLBACK_ON_CLEAR_CHANGE)) {
			for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
				if (listener != null) {
					listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, null);
				}
			}
		}
		for (int i = mcr.keysModified.size() - 1; i >= 0; i--) {
			final String key = mcr.keysModified.get(i);
			for (OnSharedPreferenceChangeListener listener : mcr.listeners) {
				if (listener != null) {
					listener.onSharedPreferenceChanged(SharedPreferencesImpl.this, key);
				}
			}
		}
	} else {
		//如果当前是子线程，就切换到主线程重新调用方法
		ActivityThread.sMainThreadHandler.post(() -> notifyListeners(mcr));
	}
}
```

看了这个方法，我才知道SP还可以注册监听。并且只有在`commit`或者`apply`时才会触发监听。而且**必定在主线程回调**。那么如果监听太多，或者监听里面有耗时操作，那么必定还是会ANR。

下面是监听的用法：

```java
val sp = getSharedPreferences("我擦", Context.MODE_PRIVATE)
        
val listener = object: SharedPreferences.OnSharedPreferenceChangeListener{

	override fun onSharedPreferenceChanged(sharedPreferences: SharedPreferences?, key: String?) {
		// do something
	}
}

sp.registerOnSharedPreferenceChangeListener(listener)

sp.unregisterOnSharedPreferenceChangeListener(listener)
```
## 专题：SP的缓存机制

### 内存缓存

##### `ContextImpl`中

```java
private ArrayMap<String, File> mSharedPrefsPaths;
```

用来缓存sp的file对象，它把name与file一一对应，用来根据name快速找到对应的file。

```java
private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;
```

用来根据file缓存spIml对象。

```java
private final File mFile;
```

sp的硬盘缓存。

##### spImpl中

```java
private Map<String, Object> mMap;
```

内容的内存缓存，主要用于`getXXX()`方法。


##### EditorImpl中

```java
private final Map<string, object> mModified = new HashMap<>();
```

写缓存，主要用于`putXXX()`方法写入的缓存，会先存到这里，等调用`commit`或者`apply`时，通过`commitToMemory`方法一次性刷入`mapToWriteToDisk`中。

##### MemoryCommitResult中

```java
final Map<string, object> mapToWriteToDisk;
```

用来做写入前的缓存，但其实并不是缓存，而是`mMap`的引用。修改它就是修改`mMap`。

#### 硬盘缓存

- mFile ：位于/data/data/包名/shared_prefs/[name].xml，真正的sp文件，xml格式。
- mBackupFile: 位于/data/data/包名/shared_prefs/[name].xml.bak,备份文件。

## 专题：SP的备份恢复机制

#### 备份

```java
private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {

        //如果文件存在而备份不存在，则从源文件创建备份文件。
        if (!backupFileExists) {
            if (!mFile.renameTo(mBackupFile)) {
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {//如果源文件和备份都存在，则删除源文件
            mFile.delete();
        }
    


        ///写入源文件，此时一定会生成一个新的源文件
        FileOutputStream str = createFileOutputStream(mFile);
        ...

        // 如果写入成功，就删除备份
        mBackupFile.delete();

        ///写入失败就删除脏文件，保留备份
        mFile.delete()
}
```

这里就一句话：**写前备份，写后成功删备份，失败删源文件**。

#### 恢复

```java
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    startLoadFromDisk();
}
```

在`spImpl`的构造方法中会创建一个mBackupFile。之后，就会开始从硬盘加载，也就是`startLoadFromDisk`。在这个方法中会创建一个子线程，执行`loadFromDisk`:

```java
private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return;
        }
        //如果有备份就删掉原文件，然后从备份中恢复
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    //从mFile中解析xml到mMap缓存中
}
```

而第一次通过构造方法过来到这里时，`mBackupFile`还没有创建真实的物理文件，所以不会走恢复流程，而是直接从mFile中加载内容。

但是如果`mBackupFile`已经存在，则先删掉源文件，再从备份中恢复，最后再读取到内存缓存。

#### 总结

SP会在写入硬盘之前，先把以前的数据备份，写入成功就删除备份，失败就删除脏的源文件而保留备份。待下次初始化SP时，会重新从硬盘读取内容到缓存，此时如果存在备份，就先删掉源文件再从备份还原。

## 专题：SP中的锁

- mLock 主要用来保护spImpl对象的安全，它是所有访问spImpl对象的操作阻塞，已达到安全性目的。
- mWritingToDiskLock：文件写入锁，当多次调用apply时可以阻塞写入。
- mEditorLock：多线程调用`editor.putXXX()`方法时的同步锁，用来保护editor内部的`mModified`缓存的数据安全性。
- sLock：QueuedWork中用来同步启动HandlerThread和同步访问work和finisher这两条队列的同步锁。
- sProcessingWork：确保同一时间只有一个线程调用`processPendingWork()`方法(前面说过apply会一部调用这方法，而activityThread会在生命周期变化时同步调用这方法)。
- writtenToDiskLatch: 文件写入结果MemoryCommitResult中的计数锁，阻塞当前线程，只能countDown一次，代表文件读写完成(并不代表成功)，然后才能继续执行线程。

## 专题：SP造成的卡顿与ANR分析

### xml文件造成的卡顿

SP在初始化过程中，要创建SPImpl对象时会创建一个**子线程**访问硬盘，**加载并解析xml文件**到内存缓存。如果这个文件比较大就会造成卡顿。那为什么在子线程中玩这一套会影响到主线程呢？就因为玩的是xml格式文件。

SP使用了高效的poll方式来解析xml成map，在此过程中会创建大量的临时对象，造成频繁的GC，而java在GC时会暂停所有的线程，当然也包括主线程，这就造成了主线程的卡顿了。

### 初始化后直接读取造成的卡顿

在初始化过程中会调用`startLoadFromDisk()`加载文件，如果文件还没有加载完就在主线程调用`getXXX()`方法也会造成卡顿，甚至ANR。因为这两个方法都竞争**`mLock`**锁，并且在`getXXX()`中还会死循环让`mLock.wait()`，而只有文件全部读取完成后这把锁才会notifyAll。

### commit方法造成的卡顿

commit是在当前线程去写入，写入完成之后还会返回结果，如果在主线程调用这个方法，就会容易引起ANR。

### apply方法引起的ANR

apply本身是依靠handlerThread任务队列来异步写入的，但是`ActivityThread`在生命周期发生改变时，会在主线程主动调用`QueuedWork`的`waitToFinish()`方法，遍历执行所有的finisher，而finisher中只有一行代码，就是让mcr的计数锁去await当前线程，对于activityThread来说就是主线程。

Android8.0之后，还加入可主动调用`processPendingWork()`方法执行写入，所以导致Apply方法也很容易造成ANR。

而如果我们过多的使用了apply，那么就放到了QueuedWork中的一个队列，依靠仅仅一个HandlerThread，一个一个的慢慢处理。那么当生命周期改变时，这些work没有被处理完，`waitToFinish()`方法就会移除剩余的msg，剩下的work会切换到主线程来执行，并且靠计数锁阻塞主线程，就造成了ANR。

>从这里可以看出，google希望在页面生命周期改变时，sp应该切换到主线程同步写入，宁可阻塞UI也要保证sp的写入完整性，这样才能保证到下一个页面我们直接get时能够拿到最新的值。同时这里也体现了sp应该只存入少量数据的设计思想。

## 专题：SP的问题梳理

1. xml格式，导致文件庞大，占用硬盘；
2. xml解析带来的频繁GC，造成卡顿；
3. 初始化时把所有文件内容缓存，造成内存压力；
4. commit和apply都会造成ANR；
5. 每次全量写入；
6. 进程不安全；
7. 只能在主线程监听；

## 专题：注意事项：

1. 不要存过多内容，尤其不要存json字符串，因为json字符串中存在大量的转义字符&；
2. 尽量拆分多个sp文件；
3. 不要在初始化sp后就直接put;
4. 不要过多调用`sp.edit()`因为每次都会生成新的`editor`对象；
5. 尽量使用apply方法，不要多次使用apply方法。
6. 多进程使用依赖**contentProvider**。

## 专题：改造思路

1. 采用protobuf减少文件体积，提高序列化、反序列化效率，提高io速度；
2. 引入懒加载机制；
3. 增量写入；
4. 解除监听回调的主线程限制。
5. 终极方案当然是改用mmkv或者dataStorge