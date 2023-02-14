---
title: "DiskLruCache"
subtitle: "OkHttp3中的实现"
date: 2021-09-16T10:05:42+08:00
lastmod: 2021-09-16T10:05:42+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: [第三方框架]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

之前已经了解过，OkHttp默认添加CacheInterceptor来处理缓存，而后者是依靠`DiskLruCache`来实现硬盘缓存读写，所以我先去看了一下Android内置的`LruCache`原理，是依靠java的`LinkedHashMap`的LRU排序算法实现的，最后我准备详细看看`DiskLruCache`是怎么基于硬盘缓存实现LRU的。
<!--more-->

### 写在前面

**据说**最早`DiskLruCache`并不是Android实现的，所以系统中并没有内置这个类，所以一般使用前先从github上下载，而OkHttp中更干脆，直接自己实现(mo gai)了一个，所以本文的`DiskLruCache`是位于`okhttp3.internal.cache`包下的。不过实现原理也都差不多。

### 静态创建方法

```java
public static DiskLruCache create(FileSystem fileSystem, File directory, int appVersion, int valueCount, long maxSize) {

  // Use a single background thread to evict entries.
  // 创建了一个单线程的线程池
  Executor executor = new ThreadPoolExecutor(0, 1, 60L, TimeUnit.SECONDS,
      new LinkedBlockingQueue<Runnable>(), Util.threadFactory("OkHttp DiskLruCache", true));

  return new DiskLruCache(fileSystem, directory, appVersion, valueCount, maxSize, executor);
}
```

首先创建了一个单线程的线程池，核心线程数为0，最大线程数为1，活跃时间为1分钟，创建的线程是**守护线程**。

然后创建一个`DiskLruCache`，并传递了线程池。

### 构造方法

```java
DiskLruCache(FileSystem fileSystem, File directory, int appVersion, int valueCount, long maxSize, Executor executor) {
  
  this.fileSystem = fileSystem;
  this.directory = directory;
  this.appVersion = appVersion;
  this.journalFile = new File(directory, JOURNAL_FILE);
  this.journalFileTmp = new File(directory, JOURNAL_FILE_TEMP);
  this.journalFileBackup = new File(directory, JOURNAL_FILE_BACKUP);
  this.valueCount = valueCount;
  this.maxSize = maxSize;
  this.executor = executor;
}
```

这里创建了3个文件：
  - journalFile：标准的journal文件
  - journalFileTmp
  - journalFileBackup

### edit方法

对应的是map的`put`方法。

```java
// 看看，还是基于这玩意的「插入排序」实现的
final LinkedHashMap<String, Entry> lruEntries = new LinkedHashMap<>(0, 0.75f, true);

// edit方法
synchronized Editor edit(String key, long expectedSequenceNumber) throws IOException {
  
  // 首先初始化
  initialize();

  // 当前DiskLruCache的关闭检查，必须关闭之后才能edit。
  checkNotClosed();

  // key的安全检查
  validateKey(key);

  // 查询到对应的entry
  Entry entry = lruEntries.get(key);
  if (expectedSequenceNumber != ANY_SEQUENCE_NUMBER && (entry == null
      || entry.sequenceNumber != expectedSequenceNumber)) {
    return null; // Snapshot is stale.
  }
  if (entry != null && entry.currentEditor != null) {
    return null; // Another edit is in progress.
  }

  if (mostRecentTrimFailed || mostRecentRebuildFailed) {
    // The OS has become our enemy! If the trim job failed, it means we are storing more data than
    // requested by the user. Do not allow edits so we do not go over that limit any further. If
    // the journal rebuild failed, the journal writer will not be active, meaning we will not be
    // able to record the edit, causing file leaks. In both cases, we want to retry the clean up
    // so we can get out of this state!
    //异步执行
    executor.execute(cleanupRunnable);
    return null;
  }

  // Flush the journal before creating files to prevent file leaks.
  journalWriter.writeUtf8(DIRTY).writeByte(' ').writeUtf8(key).writeByte('\n');
  journalWriter.flush();

  if (hasJournalErrors) {
    return null; // Don't edit; the journal can't be written.
  }

  if (entry == null) {
    entry = new Entry(key);
    lruEntries.put(key, entry);
  }
  Editor editor = new Editor(entry);
  entry.currentEditor = editor;
  return editor;
}
```

先调用了初始化方法`initialize`，主要干了两件事情：
  - 从备份中恢复；
  - 通过`okio`，从文件中读取键值对到LinkedHashMap。

```java
public synchronized void initialize() throws IOException {
    assert Thread.holdsLock(this);

  if (initialized) {
    return; // Already initialized.
  }

  // 从备份中恢复
  if (fileSystem.exists(journalFileBackup)) {
    // If journal file also exists just delete backup file.
    if (fileSystem.exists(journalFile)) {
      fileSystem.delete(journalFileBackup);
    } else {
      fileSystem.rename(journalFileBackup, journalFile);
    }
  }

  // Prefer to pick up where we left off.
  if (fileSystem.exists(journalFile)) {
    try {
      // 读取杂志文件
      readJournal();
      processJournal();
      initialized = true;
      return;
    } catch (IOException journalIsCorrupt) {
      Platform.get().log(WARN, "DiskLruCache " + directory + " is corrupt: "
          + journalIsCorrupt.getMessage() + ", removing", journalIsCorrupt);
    }

    try {
      delete();
    } finally {
      closed = false;
    }
  }

  rebuildJournal();

  initialized = true;
}
```

### 懒得看了

找到一篇文章讲的不错：

[感谢原作者](https://blog.csdn.net/yangxi_pekin/article/details/73459961)

接下来想先去看看okio。